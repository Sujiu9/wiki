# 🟡 macOS FileProvider 文件投影机制

> 📋 前置知识：[macOS 基础概念](../01-fundamentals/README.md)

## 为什么需要文件投影？

AI Agent 执行代码修改时，面临一个核心矛盾：

- Agent 需要**看到**用户的完整文件系统才能正确工作
- Agent 的修改**不能直接写入**原始文件——万一改错了呢？

没有文件投影会怎样：
- Agent 直接写入磁盘 → 改错了无法撤销，用户数据受损
- 简单的文件复制 → 大型项目（node_modules 几万个文件）耗时过长
- 符号链接 → 修改会直接影响源文件，没有隔离效果

**核心需求**：让 Agent 看到完整的文件树，所有修改记录在隔离层（overlay），用户审核后才真正写回。

---

## 解决方案概览

| 方案 | 简述 | 适用场景 |
|------|------|----------|
| FileProvider Extension (macOS) | 系统级虚拟文件系统，按需提供文件内容 | macOS 应用 |
| ProjFS (Windows) | Windows 投影文件系统 API | Windows 应用 |
| FUSE | 用户态文件系统框架 | Linux/macOS（需内核扩展） |
| OverlayFS | 联合文件系统 | Linux 原生支持 |
| 文件复制 + Git diff | 完整复制后追踪差异 | 通用但性能差 |

---

## 方案对比

| 维度 | FileProvider | FUSE | 文件复制 |
|------|-------------|------|----------|
| 性能 | 高（按需加载，零复制） | 中（用户态切换开销） | 低（全量复制） |
| 稳定性 | 高（系统进程 fileproviderd 托管） | 中（依赖第三方 kext/macfuse） | 高 |
| 隔离性 | 完整（独立目录挂载） | 完整 | 完整 |
| macOS 兼容性 | ✅ 原生支持（macOS 11+） | ⚠️ 需禁用 SIP 或用 macFUSE | ✅ |
| 上架 App Store | ✅ | ❌ | ✅ |
| 实时性 | 实时（内容由 Extension 动态提供） | 实时 | 非实时（需要同步） |

---

## FileProvider Extension 架构

### 系统组件

```
┌───────────────────────────────────────────────────────────────┐
│  用户应用 (宿主 App)                                           │
│  └── FileProviderManager: 管理 domain 注册/注销/信号           │
└───────────────────────┬───────────────────────────────────────┘
                        │ 通过 NSFileProviderManager API
┌───────────────────────▼───────────────────────────────────────┐
│  fileproviderd (系统守护进程)                                    │
│  负责调度、缓存管理、生命周期控制                                  │
└───────────────────────┬───────────────────────────────────────┘
                        │ 按需回调
┌───────────────────────▼───────────────────────────────────────┐
│  FileProvider Extension (独立进程)                              │
│  实现 NSFileProviderReplicatedExtension 协议                   │
│  ├── item(for:) — 返回文件元数据                               │
│  ├── enumerator(for:) — 枚举目录内容                           │
│  └── fetchContents(for:) — 提供文件真实内容                     │
└───────────────────────────────────────────────────────────────┘
```

### Domain 的概念

FileProvider 使用 **Domain** 来区分不同的投影实例。每个 Domain 在 Finder 中显示为一个独立的「位置」：

```swift
// 注册一个投影 domain
let domain = NSFileProviderDomain(
    identifier: NSFileProviderDomainIdentifier("workbuddy-sandbox-session1"),
    displayName: "AI Workspace - Project"
)
NSFileProviderManager.add(domain) { error in
    // domain 注册成功后，Extension 会被系统自动加载
}
```

---

## Overlay 层设计（COW 写时复制）

### 核心思想

```
用户请求读文件：
  1. 检查 overlay 层是否有修改版本 → 有则返回 overlay 版
  2. 检查 deletions 集合是否标记为已删除 → 是则报告文件不存在
  3. 以上都没有 → 从源文件系统读取原始内容

用户请求写文件：
  → 将修改后的内容写入 overlay 层（源文件不受影响）

用户请求删除文件：
  → 在 deletions 集合中标记该路径（源文件不受影响）
```

### 数据结构

```
overlay-root/
├── <session-id>/
│   ├── <mount-unit-hash>/
│   │   ├── modified/          # 被修改的文件（完整内容）
│   │   │   ├── src/main.rs
│   │   │   └── Cargo.toml
│   │   ├── created_dirs/      # 新创建的空目录标记
│   │   └── deletions.json     # 被删除的文件路径集合
│   └── ...
```

### 可见性判定算法

```swift
/// 判断文件在投影视图中是否可见
func itemVisible(_ relativePath: String) -> Bool {
    // 1. overlay 中有修改版本 → 可见
    if overlayExists(relativePath) { return true }
    
    // 2. 被标记为删除 → 不可见
    if deletions.contains(relativePath) { return false }
    
    // 3. 源文件系统中存在 → 可见
    return FileManager.default.fileExists(atPath: realPath(relativePath))
}
```

---

## WholeDisk 全盘投影模式

### 为什么需要全盘投影？

Agent 的操作可能涉及工作目录之外的文件：

- 安装全局工具（`/usr/local/bin`）
- 修改环境配置（`~/.config/`）
- 读取依赖包（`~/.cargo/registry/`）

单目录投影无法满足这些场景，需要将**整个文件系统**投影给 Agent。

### Mount Unit 切片

全盘投影按文件系统挂载点（mount unit）切分，每个挂载点独立管理 overlay：

```swift
enum WholeDiskMountUnit {
    case root           // /（根文件系统）
    case home           // /Users/<user>（用户目录，APFS 独立卷）
    case volumes(name: String)  // /Volumes/<Name>（外部卷）
}
```

### Domain 命名约定

```
workbuddy-sandbox-wholedisk-<sessionId>
```

Extension 启动时解析 domain identifier 判断工作模式：

```swift
static func parseDomainMode(rawIdentifier raw: String) -> DomainMode {
    let wdMarker = "workbuddy-sandbox-wholedisk-"
    if let r = raw.range(of: wdMarker) {
        let sid = String(raw[r.upperBound...])
        return .wholeDisk(sessionId: sid)
    }
    return .workspace(domainIndex: 0)
}
```

---

## Apply（落盘）流程

当用户确认 Agent 的修改后，overlay 层的变更需要写回源文件系统：

```
┌──────────────────────────────────────────────────────────────┐
│  PreMerge 阶段                                                │
│  • 对即将被覆盖的源文件做快照（用于回滚）                        │
│  • 基于 libgit2 的语义快照                                     │
├──────────────────────────────────────────────────────────────┤
│  Trash 阶段                                                   │
│  • 非 WorkArea 的删除操作 → 移入系统回收站                      │
│  • WorkArea 的删除操作 → 直接删除                               │
├──────────────────────────────────────────────────────────────┤
│  Write 阶段                                                   │
│  • 将 overlay/modified/ 中的文件原子写回源目录                  │
│  • 失败时自动回滚到 PreMerge 快照                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 与 Rust 核心的 FFI 桥接

FileProvider Extension（Swift）需要调用 Rust 实现的 projfs-core 库：

```swift
/// 通过 dlopen 动态加载 libsandbox_ffi.dylib
final class ProjFSNative {
    static let shared = ProjFSNative()
    private var handle: UnsafeMutableRawPointer?
    
    func load() {
        handle = dlopen("libsandbox_ffi.dylib", RTLD_NOW)
        // 通过 dlsym 获取函数指针
        // projfs_enumerate, projfs_read_content, projfs_write_content ...
    }
}
```

为什么用 dlopen 而不是静态链接？
- Extension 是独立进程，与主 App 分开加载
- 动态库可以独立更新，无需重新签名 Extension
- 避免 Extension 二进制体积过大

---

## 常见问题与陷阱

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| Extension 被系统杀死 | 内存压力或长时间无请求 | 设计为无状态，重启后重建上下文 |
| 缓存刷新不及时 | fileproviderd 有激进的缓存策略 | 调用 `signalEnumerator` 通知变更 |
| 大文件传输慢 | Extension 内存受限（~80MB） | 分块读取，使用 `NSFileProviderItemVersion` 避免重复传输 |
| 并发 deletions.json 写冲突 | 多线程同时修改删除集合 | 使用 per-mount-unit 的 NSLock 隔离 |

---

## 延伸阅读

- [进程沙盒机制](../03-security/进程沙盒机制.md)
- [NetworkExtension 网络拦截](../05-networking/NetworkExtension网络拦截.md)
- [跨平台对比：文件投影技术](../../cross-platform/comparison/文件投影技术对比.md)
