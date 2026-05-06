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

## Windows vs macOS：为什么 Windows 能通过"打包"逃逸而 macOS 不能

### 快速对照表

| 维度 | Windows (MSIX/AppContainer) | macOS (App Sandbox) |
|------|---------------------------|---------------------|
| 沙箱实现层 | 用户态 Token + DACL 检查 | 内核态 MACF 钩子 |
| 进程 exec 限制 | 几乎无限制，可自由 CreateProcess | 严格白名单，只能 exec bundle 内 |
| Profile 覆盖 | 可通过 manifest 声明 capability 获得额外权限 | 只有 Entitlement，权限单调递减 |
| 管理员提权 | UAC 弹窗后可获得 admin token | 无法提权突破 sandbox |
| 子进程继承 | AppContainer token 可选不继承 | Seatbelt profile **强制继承** |
| Bundle 逃逸 | 打包为 bundle 后 SmartScreen/MOTW 跳过 | 打包为 .app 不影响 sandbox 行为 |

---

### 深层原理：两套完全不同的安全哲学

要理解"为什么 Windows 能通过打包解决问题而 macOS 不能"，必须从两个操作系统安全模型的**根本哲学差异**说起。

#### Windows：DAC（自主访问控制）为核心

```
┌─────────────────────────────────────────────────────────────┐
│  Windows 安全决策路径                                         │
│                                                             │
│  进程创建 CreateProcess()                                    │
│    │                                                        │
│    ▼                                                        │
│  ┌──────────────────────────────────────────────────┐       │
│  │ 用户态检查层 (可绕过)                               │       │
│  │                                                  │       │
│  │ ① SmartScreen：检查 EXE 的 Mark-of-the-Web       │       │
│  │    → 来自网络下载？→ 弹警告                        │       │
│  │    → 本地创建/bundle 内？→ 直接通过                 │       │
│  │                                                  │       │
│  │ ② Authenticode：检查数字签名                       │       │
│  │    → 有受信签名？→ SmartScreen 更宽容              │       │
│  │    → MSIX 包签名？→ 操作系统信任，全部跳过          │       │
│  │                                                  │       │
│  │ ③ AppLocker/WDAC：企业策略（默认未启用）           │       │
│  │    → 普通消费者 PC 上通常关闭                      │       │
│  └─────────────────────┬────────────────────────────┘       │
│                        │                                     │
│                        ▼ (通过用户态检查后)                    │
│  ┌──────────────────────────────────────────────────┐       │
│  │ 内核态：纯 DAC 检查                               │       │
│  │                                                  │       │
│  │ 唯一看的是：进程 Token 中的 SID/权限               │       │
│  │                                                  │       │
│  │ → Token 里有 Administrators SID？→ 允许           │       │
│  │ → DACL 允许该 SID 访问？→ 允许                    │       │
│  │ → 没有 restricted SID？→ 正常权限                 │       │
│  │                                                  │       │
│  │ 内核从不关心：                                    │       │
│  │   - EXE 从哪里来的                               │       │
│  │   - 是否有签名                                   │       │
│  │   - 是否打包在 MSIX 中                           │       │
│  │   - SmartScreen 怎么判断的                       │       │
│  └──────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

**关键洞察**：Windows 内核（ntoskrnl）在做文件/网络访问决策时，**只看 Token（令牌）**。Token 是一个用户态可以操纵的对象。SmartScreen、MOTW、Authenticode 全部是**用户态/Shell 层**的检查——它们发生在 `CreateProcess` 之前或期间，一旦进程创建成功，内核就只认 Token。

#### macOS：MAC（强制访问控制）为核心

```
┌─────────────────────────────────────────────────────────────┐
│  macOS 安全决策路径                                           │
│                                                             │
│  进程创建 posix_spawn() / execve()                           │
│    │                                                        │
│    ▼                                                        │
│  ┌──────────────────────────────────────────────────┐       │
│  │ 内核态检查层 (不可绕过)                             │       │
│  │                                                  │       │
│  │ ① MACF (Mandatory Access Control Framework)      │       │
│  │    → Seatbelt (App Sandbox):                     │       │
│  │      读取嵌入的 Entitlements → 生成 Profile       │       │
│  │      每个 syscall 经过 MACF 钩子检查              │       │
│  │      profile 不可修改、不可移除、叠加只取交集       │       │
│  │                                                  │       │
│  │ ② 代码签名验证 (内核级)                           │       │
│  │    → 不是可选的！内核在 execve 时强制验证          │       │
│  │    → 签名不匹配 → SIGKILL（不是弹窗，是杀进程）    │       │
│  │                                                  │       │
│  │ ③ AMFI (Apple Mobile File Integrity)             │       │
│  │    → 验证 entitlements 是否合法                   │       │
│  │    → 只有 Apple 签发的 provisioning profile      │       │
│  │      才能获得特权 entitlements                    │       │
│  │                                                  │       │
│  │ ④ TCC (Transparency, Consent, and Control)       │       │
│  │    → 额外的资源保护层                             │       │
│  │    → 即使 sandbox profile allow 也可能被 TCC deny │       │
│  └─────────────────────┬────────────────────────────┘       │
│                        │                                     │
│                        ▼                                     │
│  ┌──────────────────────────────────────────────────┐       │
│  │ 结果：进程能做什么 = 多层策略的交集                  │       │
│  │                                                  │       │
│  │ 有效权限 = Sandbox ∩ SIP ∩ TCC ∩ AMFI ∩ POSIX   │       │
│  │                                                  │       │
│  │ 任何一层 deny → 最终 deny（无法从用户态推翻）       │       │
│  └──────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

**关键洞察**：macOS 的安全决策**全部在内核态完成**，用户态进程无法观察到也无法干预这些检查。打包为 .app bundle 只是一种**文件组织方式**，不影响内核的任何安全决策。

---

### 具体场景对比：打包如何影响安全行为

#### 场景 1：下载并运行一个 EXE / App

```
Windows:
  下载 installer.exe (带 Zone.Identifier ADS = MOTW 标记)
    → 双击 → SmartScreen 拦截 → "Windows 已保护你的电脑"
    → 用户点"仍然运行" → 创建进程
    → 进程 Token = 当前用户 → 拥有完整权限
  
  打包为 MSIX bundle (微软签名):
    → 安装时操作系统信任签名 → 不标记 MOTW
    → 从 Start Menu 启动 → 不经过 SmartScreen
    → 进程 Token = 当前用户 → 拥有完整权限（和上面一样！）
    → 🎯 区别仅在"启动前"的信任检查，启动后一模一样

macOS:
  下载 App.zip (带 com.apple.quarantine xattr)
    → 双击 → Gatekeeper 检查签名 → "xxx 是从互联网下载的..."
    → 用户确认 → 创建进程
    → 如果有 App Sandbox entitlement → 内核施加 profile
    → 每次 syscall 都被 Seatbelt 拦截检查
  
  打包为 .app bundle (Developer ID 签名):
    → Gatekeeper 验证签名 → 通过 → 不弹 quarantine 警告
    → 创建进程
    → ⚠️ 如果有 App Sandbox entitlement → 内核照样施加 profile
    → ⚠️ 每次 syscall 照样被 Seatbelt 拦截检查
    → 🎯 打包签名只跳过了 Gatekeeper 弹窗，不影响运行时行为
```

#### 场景 2：进程需要执行额外的二进制

```
Windows:
  主进程 (无论是否打包):
    CreateProcessW("python3.exe", "--version", ...);
    → 内核看 Token：有权限执行任意 EXE
    → ✅ 成功

  即使在 AppContainer 中:
    → AppContainer 的 process-exec 限制非常弱
    → 只要 EXE 路径的 DACL 允许 AppContainer SID 读取就能执行
    → 大部分系统 EXE（C:\Windows\System32\*）都允许

macOS (App Sandbox):
  主进程 (com.apple.security.app-sandbox = true):
    posix_spawn("python3", ...);
    → 内核 vnode_check_exec 钩子触发
    → Seatbelt 检查: python3 在不在 process-exec 白名单中？
    → 白名单 = {bundle内路径, /System/Library/Frameworks/...}
    → /usr/bin/python3 不在白名单中
    → ❌ EPERM — 操作被拒绝
    
    即使改用绝对路径或 PATH 搜索都没用 — 这是内核级决策
```

#### 场景 3：子进程需要继承/逃脱安全上下文

```
Windows:
  父进程 Token = {User SID, Administrators SID, ...}
    │
    │ CreateProcessW(child_exe, ...)
    ▼
  子进程默认继承父 Token → 权限和父进程完全相同
    │
    │ 但也可以：
    │ CreateProcessAsUser(restricted_token, ...) → 降权
    │ CreateProcessW(..., AppContainer_attrs, ...) → 放入容器
    │ 都不做？→ 子进程 = 完全自由
    │
    ▼
  🎯 Windows 的 Token 继承是"默认给全权，可选择降权"

macOS (App Sandbox):
  父进程 sandbox profile = {deny file-write*, allow file-read* (subpath X), ...}
    │
    │ posix_spawn(child, ...)
    ▼
  子进程自动继承父进程的 sandbox profile（内核在 fork1 中 sandbox_copy）
    │
    │ 不可选择！不可逆！
    │ 即使 execve 也不能清除！
    │ 新的 exec 如果自带 entitlement → 额外叠加新 profile
    │ 叠加 = AND = 更严格
    │
    ▼
  🎯 macOS 的 profile 继承是"默认给枷锁，只能越戴越多"
```

---

### 根因分析：为什么会有这种设计差异

#### Windows 的历史包袱决定了 DAC 模型

```
Win32 API 诞生(1993) → Token/DACL 安全模型确立
       │
       ├── 数十万个 Win32 应用假设"我能执行任意 EXE"
       ├── 数十万个 Win32 应用假设"我能读写 C:\Users\xxx 任意文件"
       ├── 生态系统依赖"进程天生自由"的假设
       │
       ▼
微软不敢在内核层强制 MAC（向后兼容性优先）
       │
       ├── SmartScreen → 用户态可跳过的检查
       ├── UAC → 用户态弹窗（点允许就过）
       ├── AppContainer → 可选的沙箱（只有 UWP 强制）
       ├── MSIX → 打包格式（可选，跳过 SmartScreen）
       │
       ▼
🎯 结果：安全机制都是"可绕过的附加层"，不是"不可逃脱的内核约束"
```

#### macOS 的 iOS 血统决定了 MAC 模型

```
iOS 诞生(2007) → 从第一天就是全 sandbox + 全签名
       │
       ├── App Store 唯一分发渠道
       ├── 所有 App 必须 sandboxed
       ├── 所有代码必须签名
       ├── 内核 AMFI 验证一切
       │
       ▼
macOS 逐步"iOS 化"(2012 Gatekeeper → 2019 Hardened Runtime → 2020 签名强制)
       │
       ├── Mac App Store → 强制 App Sandbox
       ├── Developer ID → 公证(Notarization)
       ├── Seatbelt 内核模块 ← 直接复用 iOS 沙盒引擎
       ├── AMFI ← 直接复用 iOS 签名验证
       │
       ▼
🎯 结果：安全机制是"内核级强制约束"，与打包方式、分发方式无关
```

---

### Windows "打包逃逸"的具体机制

所谓 Windows "通过打包逃逸"，具体绕过的是哪些检查：

#### 1. Mark-of-the-Web (MOTW) 逃逸

```
MOTW 实现原理：NTFS Alternate Data Stream (ADS)
  文件从网络下载时，浏览器写入:
    file.exe:Zone.Identifier → "ZoneId=3" (Internet Zone)

SmartScreen 检查流程:
  ShellExecuteEx("file.exe")
    → shell32.dll 读取 Zone.Identifier ADS
    → ZoneId >= 3 → 调用 SmartScreen 服务
    → SmartScreen 查云端信誉数据库
    → 信誉未知/恶意 → 弹窗警告

打包逃逸:
  MSIX 安装器安装应用时：
    → 将 EXE 从包中解压到 C:\Program Files\WindowsApps\xxx\
    → 解压出的文件没有 Zone.Identifier ADS（因为不是"下载"的）
    → 从 Start Menu 启动时 ShellExecute 读不到 MOTW
    → 跳过 SmartScreen 检查
    
  为什么管用？因为 SmartScreen 是 shell32 用户态代码，不是内核检查
  内核看到的只是正常的 CreateProcess → Token → 放行
```

#### 2. Token 与权限的"先天全权"

```c
// Windows 进程创建时的 Token 状态（非 AppContainer）
typedef struct _TOKEN {
    LUID AuthenticationId;
    SID_AND_ATTRIBUTES User;         // 用户 SID
    SID_AND_ATTRIBUTES Groups[];     // 所在组（含 Administrators？）
    PRIVILEGE_SET Privileges[];       // SeBackupPrivilege? SeDebugPrivilege?
    // ...
} TOKEN;

// 打包应用 vs 非打包应用的 Token 对比：
// 
// 非打包应用 (直接运行 EXE):
//   Token = {User=S-1-5-21-xxx, Groups=[Users, Administrators(deny-only)], ...}
//
// MSIX 打包应用 (非 AppContainer 模式):
//   Token = {User=S-1-5-21-xxx, Groups=[Users, Administrators(deny-only)], ...}
//
// → 一模一样！Token 不因打包方式而不同（除非选择 AppContainer 模式）
//
// 而 MSIX + AppContainer 模式（仅 UWP/打包桌面应用可选）:
//   Token 增加: AppContainerSid=S-1-15-2-xxx, capabilities=[...]
//   → 但这是可选的！传统桌面应用打成 MSIX 可以不走 AppContainer
```

#### 3. 为什么 macOS 做不到同样的事

```
macOS 中没有等价的"MOTW"可以绕过：

  文件下载时:  xattr -w com.apple.quarantine "0081;..." file.app
  
  首次打开时:
    → Gatekeeper 检查: 有签名？签名有效？已公证？
    → 如果通过 → 移除 quarantine xattr → 以后不再检查
    → ⚠️ 这只是"首次打开"的一次性检查，类似 Windows SmartScreen
    → ✅ 实际上签名合法的 .app 也能跳过这步（和 Windows MSIX 类似）

  但关键区别在"之后"：
    → Windows: 进程启动后，内核只看 Token，不关心来源
    → macOS:  进程启动后，内核读取 binary 中嵌入的 entitlements
              如果声明了 app-sandbox → 内核施加 profile → 永久生效
              如果没声明 app-sandbox → 没有额外限制
              
  所以:
    macOS 的"打包为 .app"≈ Windows 的"打包为 MSIX" → 都能跳过下载警告
    但 macOS 的 App Sandbox ≠ Windows 的 AppContainer:
      - macOS App Sandbox: 内核强制，一旦声明不可关闭
      - Windows AppContainer: 可选，不选就没有
```

---

### 关键差异总结：安全检查发生在哪一层

```
                用户态                    │         内核态
                (可绕过)                  │         (不可绕过)
───────────────────────────────────────────┼──────────────────────────────────
                                          │
Windows:  ┌─────────────────────────┐     │  ┌─────────────────────┐
          │ SmartScreen             │     │  │ Token/DACL 检查      │
          │ MOTW                    │     │  │ (只看 SID 和 ACL)    │
          │ Authenticode 弹窗       │     │  │                     │
          │ UAC 弹窗               │     │  │ 不关心签名           │
          │ AppLocker (可选)        │     │  │ 不关心来源           │
          │ ════════════════════════ │     │  │ 不关心打包方式       │
          │ 打包 = 跳过以上检查     │     │  │                     │
          └─────────────────────────┘     │  └─────────────────────┘
                                          │
macOS:    ┌─────────────────────────┐     │  ┌─────────────────────┐
          │ Gatekeeper 首次弹窗     │     │  │ MACF/Seatbelt       │
          │ quarantine xattr        │     │  │ AMFI 签名验证        │
          │ ════════════════════════ │     │  │ TCC 权限检查         │
          │ 签名有效 = 跳过弹窗     │     │  │ SIP 完整性保护       │
          │                         │     │  │                     │
          │ 但这只是入口检查！       │     │  │ 这才是真正的安全层！ │
          │ 运行后不再检查          │     │  │ 每次 syscall 都检查  │
          └─────────────────────────┘     │  └─────────────────────┘
                                          │
```

**一句话总结**：
- Windows 的大部分安全检查在**用户态入口**（进程启动前），打包可以绕过入口检查，之后进程自由
- macOS 的核心安全检查在**内核运行时**（每次系统调用），打包只能跳过入口弹窗，运行时限制丝毫不减

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
