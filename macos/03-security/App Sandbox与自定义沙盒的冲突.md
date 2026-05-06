# 🔴 App Sandbox 与自定义沙盒的冲突

> 📋 前置知识：[进程沙盒机制](./进程沙盒机制.md)、[FileProvider 文件投影](../04-filesystem/FileProvider文件投影.md)、[NetworkExtension 网络拦截](../05-networking/NetworkExtension网络拦截.md)

## 问题背景

Windows 上可以通过将应用打包为 MSIX/AppX bundle 逃避某些沙箱限制（如 SmartScreen、Mark-of-the-Web），但 macOS 的 **App Sandbox** 机制设计得远比 Windows 严格。

一个核心问题：如果宿主应用开启了 Apple 原生的 App Sandbox（如需要上架 Mac App Store），AgentOS 项目中的 sandbox-exec 进程沙盒、NetworkExtension 透明代理、FileProvider 全盘投影等核心功能是否还能正常工作？

**答案是：大部分走不通。** 以下从内核层面解释为什么。

---

## macOS 的双层沙盒体系

macOS 实际上存在**两套完全不同的沙盒机制**，它们工作在不同的内核层：

```
┌─────────────────────────────────────────────────────────────────┐
│                     用户态（User Space）                          │
│                                                                 │
│  ┌─────────────────────┐    ┌─────────────────────────────┐    │
│  │   App Sandbox        │    │   sandbox-exec (SBPL)       │    │
│  │   (Entitlements)     │    │   (自定义 Profile)           │    │
│  │                     │    │                             │    │
│  │  编译时声明权限       │    │  运行时动态注入规则          │    │
│  │  签名后不可修改       │    │  启动时确定，不可热修改      │    │
│  └────────┬────────────┘    └──────────────┬──────────────┘    │
│           │                                │                    │
├───────────┼────────────────────────────────┼────────────────────┤
│           ▼                                ▼                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │            Seatbelt (内核 MACF 策略模块)                   │   │
│  │                                                         │   │
│  │  - 拦截点：vnode_check_open / vnode_check_exec /        │   │
│  │           socket_check_connect / proc_check_fork ...    │   │
│  │  - 决策引擎：读取进程的 sandbox profile 做 allow/deny     │   │
│  │  - 规则叠加：多个 profile 独立评估，全部 allow 才放行       │   │
│  └────────────────────────────┬────────────────────────────┘   │
│                               │                                 │
│           内核态（Kernel Space）                                  │
├───────────────────────────────┼─────────────────────────────────┤
│                               ▼                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              MACF (Mandatory Access Control Framework)    │   │
│  │              TrustedBSD 安全框架                           │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 关键认知：两套沙盒共用同一个内核执行引擎

App Sandbox 和 sandbox-exec **不是**独立的两套系统——它们最终都由内核中的 **Seatbelt** 模块执行。区别只在于 profile 的**来源**：

| 维度 | App Sandbox | sandbox-exec |
|------|-------------|--------------|
| Profile 来源 | 由 macOS 根据 Entitlements 自动生成 | 由开发者手写 SBPL 文本传入 |
| 注入时机 | 进程启动时由 dyld 自动 apply | 调用 `sandbox_init()` 或 `sandbox-exec` 时 apply |
| 是否可叠加 | ✅ | ✅ |
| 是否可移除 | ❌ 一旦 apply 永不可逆 | ❌ 同样不可逆 |
| 子进程继承 | ✅ 子进程自动继承父进程的 profile | ✅ 同样继承 |

---

## 底层原理：为什么 Profile 叠加只会更严格

### Seatbelt 的决策模型

内核中，每个进程可以关联**多个** sandbox profile。当系统调用被 MACF 钩子拦截时：

```c
// 内核伪代码 - sandbox 检查逻辑
int sandbox_check(proc_t proc, const char *operation, ...) {
    // 遍历进程关联的所有 profile
    for each profile in proc->sandbox_profiles {
        int result = evaluate_profile(profile, operation, args);
        if (result == DENY) {
            return DENY;  // 任何一个 profile deny → 最终 deny
        }
    }
    return ALLOW;  // 全部 allow → 才放行
}
```

**这意味着**：
- Profile 之间是 **AND** 关系（交集），不是 OR（并集）
- 新叠加的 profile **只能缩小**权限范围，永远不能扩大
- 即使 sandbox-exec 的 profile 写了 `(allow file-read*)`，如果 App Sandbox 的 profile deny 了某路径，最终还是 deny

### 类比理解

```
App Sandbox profile:  只允许读 ~/Documents, ~/Library/Container/xxx
sandbox-exec profile: 只允许读 /Users/dev/project, /tmp

实际权限 = 两者交集 = 几乎什么都读不了
```

---

## 逐项分析：App Sandbox 对 AgentOS 各模块的致命影响

### 1. sandbox-exec 进程沙盒 → 完全不可用

**根因**：App Sandbox 禁止执行任意系统二进制

```
App Sandbox Profile (系统自动生成，简化):
  (deny process-exec)
  (allow process-exec
    (subpath "/Applications/MyApp.app/Contents/")  ; 只允许执行自己 bundle 内的
    (literal "/usr/bin/open")                       ; 极少数系统工具
    ... )
```

而 sandbox-exec 命令位于 `/usr/bin/sandbox-exec`，它**不在** App Sandbox 的白名单中：

```
尝试: Command::new("/usr/bin/sandbox-exec").arg("-p").arg(&profile).arg("python3")...
结果: operation denied by App Sandbox (process-exec of /usr/bin/sandbox-exec)
```

**即使通过 `sandbox_init()` C API 绕过 sandbox-exec 命令**，也存在不可解决的问题：

```c
// 尝试在代码中直接调用 sandbox_init()
#include <sandbox.h>

char *error = NULL;
// ❌ 如果进程已经在 App Sandbox 中，sandbox_init 会失败
// 因为已存在的 profile 中 deny 了 sandbox_init 所需的 mach-lookup
int ret = sandbox_init(profile_str, SANDBOX_NAMED_BUILTIN, &error);
// ret == -1, error: "not permitted"
```

**核心矛盾**：
- sandbox-exec 需要 fork + exec 出新进程并为其 apply 新 profile
- App Sandbox 对 `process-exec` 做了严格白名单
- 子进程无论如何都会继承父进程的 App Sandbox profile
- 继承的 profile 会与自定义 profile 取**交集**，导致权限比预期严格得多

### 2. FileProvider 全盘投影 → ProxyTransport 设计基础坍塌

**正常架构（unsandboxed 主 App）**：

```
FileProvider Extension (sandboxed, 系统强制)
    │
    │ "我无法读取 ~/Documents，请帮我读"
    ▼
ProxyTransport (App Group 文件邮箱)
    │
    ▼
主 App (unsandboxed, "业主身份")
    │
    │ 以 "业主" 身份直接列目录/读文件
    │ 列目录不触发 TCC 弹窗（因为不在沙盒中）
    ▼
返回文件内容给 Extension
```

**开启 App Sandbox 后**：

```
FileProvider Extension (sandboxed)
    │
    │ "我无法读取 ~/Documents，请帮我读"
    ▼
ProxyTransport (App Group 文件邮箱)
    │
    ▼
主 App (也是 sandboxed!)
    │
    │ ❌ 同样无法读取 ~/Documents
    │ ❌ 列目录也要走 TCC 弹窗
    │ ❌ 被限制在 ~/Library/Containers/com.xxx/ 中
    ▼
ProxyTransport 完全失去意义 — 代理人和委托人一样被关在笼子里
```

**TCC（Transparency, Consent, and Control）层面的影响**：

| 操作 | unsandboxed App | sandboxed App |
|------|-----------------|---------------|
| 列举 `~/Documents` 目录 | ✅ 直接成功，不触发 TCC | ❌ 需要 TCC 授权弹窗 |
| 读取 `~/Desktop` 文件内容 | 弹一次 TCC 后永久授权 | 弹 TCC，且受 Container 限制 |
| 调用 `/usr/bin/tccutil` 重置 | ✅ 可执行 | ❌ process-exec 被 deny |
| 调用 `/usr/bin/open` 重启 | ✅ 可执行 | ❌ process-exec 被 deny |

**内核层解释**：

```c
// TCC 检查的内核入口 (简化)
int tcc_check_file_access(proc_t proc, vnode_t vp) {
    // 1. 检查进程是否在 App Sandbox 中
    if (proc_has_sandbox(proc)) {
        // sandboxed 进程访问 TCC-protected 路径需要额外授权
        if (is_tcc_protected_path(vp)) {
            return check_tcc_authorization(proc->bundle_id, "kTCCServiceSystemPolicyDocumentsFolder");
        }
    } else {
        // unsandboxed 进程：列目录 free-pass，读内容才检查
        if (is_read_operation && is_tcc_protected_path(vp)) {
            return check_tcc_authorization(proc->bundle_id, ...);
        }
        return ALLOW;  // 列目录不检查
    }
}
```

### 3. NetworkExtension 透明代理 → System Extension 安装受阻

**NE 的安装流程依赖 System Extension Framework**：

```swift
// 安装 Network Extension
let request = OSSystemExtensionRequest.activationRequest(
    forExtensionWithIdentifier: "com.example.ne-extension",
    queue: .main
)
OSSystemExtensionManager.shared.submitRequest(request)
```

**问题链**：

```
App Sandbox
  │
  ├── ⚠️ OSSystemExtensionRequest 需要特殊 Entitlement
  │   └── com.apple.developer.system-extension.install
  │       （App Sandbox 环境下此 entitlement 行为可能受限）
  │
  ├── ❌ NE 的 App Group 共享路径变化
  │   ├── unsandboxed: ~/Library/Group Containers/group.xxx/
  │   └── sandboxed:   ~/Library/Containers/com.xxx/Data/Library/Group Containers/group.xxx/
  │   └── Extension 和主 App 的实际路径可能不一致
  │
  └── ❌ NE Bridge 的 FFI 调用可能因 mach-lookup 限制失败
      └── App Sandbox 对 mach-lookup 做了白名单过滤
```

### 4. 辅助工具链 → 全部断裂

| 工具 | 功能 | App Sandbox 下的命运 |
|------|------|---------------------|
| `/usr/bin/sandbox-exec` | 启动沙盒进程 | ❌ process-exec deny |
| `/usr/bin/tccutil` | 重置 TCC 授权 | ❌ process-exec deny |
| `/usr/bin/open` | 重启自身 | ❌ process-exec deny |
| `log stream` | 读取沙盒 deny 日志 | ❌ process-exec deny |
| `diskutil` | 磁盘信息查询 | ❌ process-exec deny |

---

## 内核级根因总结

从 XNU 内核源码角度，App Sandbox 导致冲突的**根本原因**是三个内核级约束：

### 约束 1：Profile 不可逆且只取交集

```c
// xnu/bsd/kern/kern_proc.c (概念性)
struct proc {
    struct sandbox_set *p_sandbox;  // sandbox profile 集合
    // ...
};

// 一旦 apply，永远不能 remove
// 子进程 fork 时自动复制父进程的 sandbox_set
// 多个 profile 评估时取 AND（交集）
```

**结论**：在 App Sandbox 已经施加了一层严格限制后，任何通过 sandbox-exec 新增的规则只能**进一步收紧**，不可能放宽。自定义沙盒预期的 "allow network-outbound" 如果被 App Sandbox deny 了，那就是 deny。

### 约束 2：process-exec 严格白名单

```c
// App Sandbox 自动生成的 profile 中（反编译 sandbox-helper 可见）
(deny process-exec)
(allow process-exec
    (subpath MAIN_BUNDLE_PATH)
    (subpath "/System/Library/Frameworks/")
    ;; 极少数例外...
)
```

App Sandbox 下，进程**只能执行自己 bundle 内的二进制和极少数系统框架**。这彻底堵死了：
- 调用 `/usr/bin/sandbox-exec`
- 调用 `/usr/bin/python3` 或 `/bin/bash` 执行用户脚本
- 调用任何系统管理工具

### 约束 3：文件系统容器化隔离

```
unsandboxed App 的文件视图:
/
├── Users/
│   └── dev/
│       ├── Documents/    ← 可直接访问
│       ├── Desktop/      ← 可直接访问
│       └── .ssh/         ← 可直接访问

sandboxed App 的文件视图:
/
├── Users/
│   └── dev/
│       └── Library/
│           └── Containers/
│               └── com.example.myapp/
│                   └── Data/       ← 这就是你的"全世界"
│                       ├── Documents/   (映射到 container 内)
│                       ├── Library/
│                       └── tmp/
```

sandboxed App 的 `NSHomeDirectory()` 返回的是 Container 路径，而非真实的 `~`。这意味着所有基于真实路径的逻辑（SBPL 中的 `(subpath "/Users/dev/project")`）都会失效。

---

## Windows vs macOS：为什么 Windows 能"逃逸"而 macOS 不能

| 维度 | Windows (MSIX/AppContainer) | macOS (App Sandbox) |
|------|---------------------------|---------------------|
| 沙箱实现层 | 用户态 Token + DACL 检查 | 内核态 MACF 钩子 |
| 进程 exec 限制 | 几乎无限制，可自由 CreateProcess | 严格白名单，只能 exec bundle 内 |
| Profile 覆盖 | 可通过 manifest 声明 capability 获得额外权限 | 只有 Entitlement，权限单调递减 |
| 管理员提权 | UAC 弹窗后可获得 admin token | 无法提权突破 sandbox |
| 子进程继承 | AppContainer token 可选不继承 | Seatbelt profile **强制继承** |
| Bundle 逃逸 | 打包为 bundle 后 SmartScreen/MOTW 跳过 | 打包为 .app 不影响 sandbox 行为 |

**核心差异**：Windows 的安全检查多在**用户态 token 层**，是"拥有权限才能做"的 DAC 模型；macOS App Sandbox 基于**内核 MAC（强制访问控制）**，是"即使你有权限也不让做"的 MAC 模型。

Windows 的 AppContainer 沙箱是可选的（不打包就没有），而 macOS App Sandbox 一旦在 Entitlements 中声明 `com.apple.security.app-sandbox = true`，就是**不可逆的内核级枷锁**。

---

## 实践结论与架构选择

```
                      ┌─────────────────────────────────┐
                      │   要上 Mac App Store 吗？         │
                      └───────────────┬─────────────────┘
                                      │
                        ┌─────────────┴─────────────┐
                        │                           │
                        ▼                           ▼
                   是(强制 sandbox)            否(Developer ID 签名)
                        │                           │
                        ▼                           ▼
              ┌────────────────────┐    ┌──────────────────────────┐
              │ 需要完全重新设计：   │    │ 当前方案可行：             │
              │                    │    │                          │
              │ • 放弃 sandbox-exec│    │ • sandbox-exec ✅          │
              │ • XPC Service 替代 │    │ • FileProvider proxy ✅    │
              │ • 放弃全盘投影     │    │ • NE transparent proxy ✅  │
              │ • 用 Bookmark API  │    │ • TCC reset ✅             │
              │ • 放弃系统命令调用  │    │ • 系统命令调用 ✅           │
              └────────────────────┘    └──────────────────────────┘
```

### 如果必须走 App Store（理论方案）

| 原能力 | 替代方案 | 代价 |
|--------|----------|------|
| sandbox-exec 进程沙盒 | XPC Service + 极有限的 Entitlements | 失去自定义 SBPL 的灵活性 |
| 全盘投影 (FileProvider) | Security-Scoped Bookmark + 用户手动授权 | 用户体验极差，无法自动化 |
| NE 透明代理 | 仍可用（NE Extension 本身就是 sandboxed） | 但主 App 与 NE 的协调受限 |
| TCC 重置 | 引导用户去系统偏好设置手动操作 | 完全无法程序化 |
| 日志监控 | OSLog API（有限） | 无法获取完整 deny 日志 |

---

## 延伸阅读

- [进程沙盒机制](./进程沙盒机制.md)
- [FileProvider 文件投影](../04-filesystem/FileProvider文件投影.md)
- [NetworkExtension 网络拦截](../05-networking/NetworkExtension网络拦截.md)
- [跨平台对比：文件投影技术](../../cross-platform/comparison/文件投影技术对比.md)
