# 🟡 macOS FileProvider 最小 Provider

> 📋 前置知识：[覆盖层文件系统从零实现](./覆盖层文件系统从零实现.md)

## 1. 为什么 macOS 用 FileProvider

macOS 没有 Windows ProjFS 的同款接口。要把一个虚拟文件树展示给 Finder 和系统文件访问路径，常见方式是 `FileProvider（NSFileProvider）`。

它的核心不是“挂载一个目录”，而是让系统向你的 Extension 查询：

- 某个目录有哪些 item；
- 某个 item 的元数据是什么；
- 用户需要内容时如何 materialize；
- 数据变化时如何通知系统刷新。

## 2. 解决方案概览

| 方案 | 简述 | 适用场景 |
|------|------|----------|
| 真实目录复制 | 直接生成目录 | 小目录、简单 Demo |
| FUSE 类方案 | 用户态文件系统 | 兼容性和分发成本较高 |
| FileProvider | Apple 官方文件提供者框架 | Finder/系统集成 |

WorkBuddySandbox macOS 侧用 FileProvider Extension 展示投影目录，并通过 App Group 与主 App / Rust 侧共享状态。

## 3. 方案对比

| 维度 | 真实复制 | FileProvider |
|------|----------|--------------|
| 系统集成 | 普通目录 | Finder/Files 语义 |
| 按需加载 | 弱 | 强 |
| 实现复杂度 | 低 | 高 |
| 缓存/刷新 | 自己处理 | 系统参与，需理解 domain |

FileProvider 的难点不是枚举一个数组，而是 identifier、domain、materialized 状态和系统缓存。

## 4. 如何写一个最小 Provider

最小模型有四个概念：

```text
Domain: 一个文件提供者空间，例如 workbuddy-sandbox-session
ItemIdentifier: 稳定 ID，例如 root、slot0、file:/src/a.txt
Enumerator: 系统问某个目录有哪些 item
Materialized: item 已经在本地被系统物化出来
```

伪代码：

```swift
func enumerateItems(parent: ItemIdentifier) -> [Item] {
    if parent == .root {
        return sessions.map { Item(id: $0.slotId, name: $0.displayName) }
    }

    let session = resolveSession(parent)
    return overlayMergedChildren(session, parent.path)
}
```

关键点：FileProvider 依赖稳定 identifier。文件名可以变化，但 identifier 变化会让系统认为是另一个对象。

## 5. 常见坑

- **identifier 和文件名混用**：identifier 应稳定，filename 是展示名。
- **materialized 缓存**：系统可能记住旧目录名，光 signal 不一定刷新。
- **Extension 生命周期不可控**：系统可能杀掉、节流、延迟启动 Extension。
- **App Group 路径错误**：主 App 和 Extension 必须共享同一个容器。
- **刷新时机**：先写 registry，再注册 domain，否则初始枚举拿不到 session 信息。

## 6. 回到 WorkBuddySandbox

| 机制 | 项目位置 |
|------|----------|
| FileProvider 主实现 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/FileExtension/Sources/FileProviderExtension.swift` |
| 枚举器 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/FileExtension/Sources/FileProviderEnumerator.swift` |
| 宿主管理 domain | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/SandboxExt/Sources/Manager/FileProviderManager.swift` |
| Rust 发起投影请求 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/file/provider_macos.rs` |
| Swift 动态调用 Rust overlay | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/FileExtension/Sources/ProjFSNative.swift` |
| 代理取数通道 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/FileExtension/Sources/ProxyTransport.swift` |

特别要记住 WorkBuddySandbox 的规则：App 启动时清理旧 domain，`startProjection` 必须先写 registry，再注册 domain。

## 7. 练习题：你要亲手实现什么

1. 设计一个内存文件树：`root -> session -> files`。
2. 给每个节点分配稳定 `itemIdentifier`，然后改文件名，观察 identifier 是否保持不变。
3. 写一个 `enumerate(parentId)`，返回 children。
4. 模拟 materialized 缓存：同一个 identifier 改名后，思考为什么系统可能不刷新。
5. 回到项目，追踪 `slot0` 是如何映射到某个 session 的。

## 8. 延伸阅读

- [macOS 代码签名与 entitlements 实战](../07-调试与系统权限/macOS代码签名与entitlements实战.md)
- [Unix Domain Socket 服务端](../03-IPC协议与服务进程/Unix-Domain-Socket服务端.md)
