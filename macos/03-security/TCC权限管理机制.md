# 🟡 macOS TCC 权限管理机制

> 📋 前置知识：[进程沙盒机制](./进程沙盒机制.md)、[App Sandbox 与自定义沙盒的冲突](./App%20Sandbox与自定义沙盒的冲突.md)

## 为什么需要 TCC？

macOS 10.14 Mojave 之前，任何应用都能自由访问用户的文件、摄像头、麦克风。TCC（Transparency, Consent, and Control）解决的核心问题是：

- 用户应该**知道**哪些应用在访问隐私数据（Transparency）
- 用户应该**主动同意**才允许访问（Consent）
- 用户应该能**随时撤销**授权（Control）

在 AI Agent 场景中，TCC 是一道关键屏障：Agent 需要读取 `~/Documents`、`~/Desktop` 中的项目文件，但 TCC 会弹窗拦截。

---

## TCC 的内核级架构

```
┌────────────────────────────────────────────────────────────┐
│  用户态                                                      │
│                                                            │
│  App 进程                           TCC 守护进程            │
│  ┌──────────┐                      ┌──────────────┐       │
│  │ open()   │──── XPC ───────────► │ tccd          │       │
│  │ ~/Docs/  │                      │              │       │
│  └──────────┘                      │ • 查询 TCC.db│       │
│       │                            │ • 弹授权窗口 │       │
│       │                            │ • 更新记录   │       │
│       │                            └──────┬───────┘       │
│       │                                   │                │
├───────┼───────────────────────────────────┼────────────────┤
│       ▼                                   │                │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  内核 Seatbelt + MACF 钩子                           │  │
│  │                                                     │  │
│  │  vnode_check_open(vp):                              │  │
│  │    if is_tcc_protected(vp):                         │  │
│  │      result = query_tccd(proc, service, vp)         │  │
│  │      if result == DENY: return EACCES               │  │
│  └─────────────────────────────────────────────────────┘  │
│                    内核态                                    │
└────────────────────────────────────────────────────────────┘
```

### TCC 保护的路径（macOS 14+）

| 路径 | TCC 服务名 | 触发条件 |
|------|-----------|---------|
| `~/Desktop` | `kTCCServiceSystemPolicyDesktopFolder` | 任何文件访问 |
| `~/Documents` | `kTCCServiceSystemPolicyDocumentsFolder` | 任何文件访问 |
| `~/Downloads` | `kTCCServiceSystemPolicyDownloadsFolder` | 任何文件访问 |
| `~/Library/Mail` | `kTCCServiceMail` | 读取邮件数据 |
| `~/Library/Messages` | `kTCCServiceMessages` | 读取消息数据 |
| `~/Library/Safari` | `kTCCServiceWebBrowserPublicKeyCredential` | 浏览器数据 |
| 整个磁盘 | `kTCCServiceSystemPolicyAllFiles` | Full Disk Access |
| 摄像头 | `kTCCServiceCamera` | AVCaptureDevice |
| 麦克风 | `kTCCServiceMicrophone` | AVAudioSession |
| 屏幕录制 | `kTCCServiceScreenCapture` | CGWindowListCreateImage |

---

## TCC 数据库

TCC 决策存储在 SQLite 数据库中：

```
用户级：~/Library/Application Support/com.apple.TCC/TCC.db
系统级：/Library/Application Support/com.apple.TCC/TCC.db
```

### 数据库 Schema（简化）

```sql
CREATE TABLE access (
    service        TEXT NOT NULL,     -- kTCCServiceSystemPolicyAllFiles
    client         TEXT NOT NULL,     -- Bundle ID 或可执行文件路径
    client_type    INTEGER NOT NULL,  -- 0=bundle_id, 1=absolute_path
    auth_value     INTEGER NOT NULL,  -- 0=denied, 2=allowed, 3=limited
    auth_reason    INTEGER NOT NULL,  -- 1=user_consent, 2=user_set, 4=system_policy
    auth_version   INTEGER NOT NULL,
    csreq          BLOB,             -- Code Signing Requirement (DER 编码)
    policy_id      INTEGER,
    indirect_object_identifier TEXT,
    flags          INTEGER,
    last_modified  INTEGER NOT NULL   -- Unix timestamp
);
```

### 关键字段解释

| 字段 | 含义 |
|------|------|
| `client_type=0` | 通过 Bundle ID 标识客户端 |
| `client_type=1` | 通过绝对路径标识（命令行工具常用） |
| `auth_value=2` | 用户已授权 |
| `auth_value=0` | 用户已拒绝 |
| `csreq` | 代码签名要求——确保只有正确签名的 App 才能使用此授权 |

---

## TCC 检查的触发流程

### sandboxed App vs unsandboxed App 的区别

```c
// 内核态 TCC 检查逻辑（概念性伪代码）
int tcc_check_file_access(proc_t proc, vnode_t vp, int operation) {
    
    // 1. 判断路径是否受 TCC 保护
    tcc_service_t service = get_tcc_service_for_path(vp);
    if (service == NONE) return ALLOW;  // 非保护路径，直接放行
    
    // 2. 区分 sandboxed 和 unsandboxed
    if (proc_is_sandboxed(proc)) {
        // Sandboxed App: 列目录也需要 TCC 授权
        if (operation == READ_DIR || operation == READ_FILE) {
            return query_tccd(proc, service);
        }
    } else {
        // Unsandboxed App: 列目录 FREE PASS！
        if (operation == READ_DIR) {
            return ALLOW;  // ← 这就是 ProxyTransport 的设计基础
        }
        // 只有读取文件内容时才检查 TCC
        if (operation == READ_FILE) {
            return query_tccd(proc, service);
        }
    }
    
    return ALLOW;
}
```

**这个差异是 AgentOS 全盘投影设计的核心前提**：
- FileProvider Extension（sandboxed）：列 `~/Documents` 就会触发 TCC
- 主 App（unsandboxed）：列目录**不触发 TCC**，只有读文件内容才弹窗
- 所以设计了 ProxyTransport：让 unsandboxed 的主 App 代为列目录

---

## Full Disk Access (FDA)

### 什么是 FDA？

Full Disk Access 是 TCC 的"超级权限"——一旦授予，应用可以访问**所有** TCC 保护的路径，无需逐个授权。

```
授权位置：系统偏好设置 → 隐私与安全 → Full Disk Access
TCC 服务名：kTCCServiceSystemPolicyAllFiles
```

### FDA 授权的内核检查顺序

```c
int check_full_disk_access(proc_t proc, tcc_service_t requested_service) {
    // 1. 先查是否有 Full Disk Access
    if (tcc_db_lookup(proc, "kTCCServiceSystemPolicyAllFiles") == ALLOWED) {
        return ALLOW;  // FDA 覆盖一切
    }
    
    // 2. 再查具体服务的授权
    return tcc_db_lookup(proc, requested_service);
}
```

### 预授权探针

AgentOS 使用"预授权探针"在会话创建时提前触发 TCC 弹窗：

```swift
// WholeDiskTCCPreauth — 预授权探针设计
class WholeDiskTCCPreauth {
    /// 触发 Full Disk Access TCC 弹窗
    /// 原理：尝试列举一个 TCC 保护的目录来激活 tccd
    func triggerFDAPrompt() -> Bool {
        let protectedPaths = [
            NSHomeDirectory() + "/Library/Mail",      // 稳定存在
            NSHomeDirectory() + "/Library/Messages",  // 稳定存在
        ]
        
        for path in protectedPaths {
            let result = FileManager.default.contentsOfDirectory(atPath: path)
            // 如果成功 → FDA 已授权
            // 如果失败 → TCC 弹窗已触发（用户看到）
        }
    }
}
```

---

## tccutil 工具

`/usr/bin/tccutil` 可以程序化重置 TCC 授权：

```bash
# 重置某个 App 的所有 TCC 权限
tccutil reset All com.example.myapp

# 重置所有 App 的特定服务
tccutil reset SystemPolicyAllFiles

# 重置某个 App 的特定服务
tccutil reset SystemPolicyDesktopFolder com.example.myapp
```

### 为什么 AgentOS 需要 tccutil？

场景：用户误点了"拒绝"Full Disk Access → 全盘投影功能彻底失效。

解决方案：

```swift
// TCCResetHelper — 重置 TCC 后重启自身
class TCCResetHelper {
    func resetAndRelaunch(bundleID: String) {
        // 1. 调用 tccutil 重置权限（需要 unsandboxed！）
        let task = Process()
        task.executableURL = URL(fileURLWithPath: "/usr/bin/tccutil")
        task.arguments = ["reset", "SystemPolicyAllFiles", bundleID]
        task.launch()
        task.waitUntilExit()
        
        // 2. 重启自身（重新触发 TCC 弹窗）
        let openTask = Process()
        openTask.executableURL = URL(fileURLWithPath: "/usr/bin/open")
        openTask.arguments = ["-n", Bundle.main.bundlePath]
        openTask.launch()
        
        // 3. 退出当前实例
        NSApp.terminate(nil)
    }
}
```

**注意**：`tccutil` 和 `/usr/bin/open` 都需要 `process-exec` 权限——App Sandbox 下两者都不可用。

---

## TCC 与代码签名的绑定

TCC 授权不是仅凭 Bundle ID 识别的——它会验证**代码签名**：

```
授权记录 = (Bundle ID) + (Code Signing Requirement)
```

如果 App 重新签名（如更换开发者证书），之前的 TCC 授权会失效，用户需要重新授权。

### csreq 字段

```bash
# 查看某个 App 的 Code Signing Requirement
codesign -dr - /Applications/MyApp.app

# 输出类似：
# designated => identifier "com.example.myapp" and anchor apple generic and
#   certificate leaf[subject.CN] = "Developer ID Application: Company (TEAMID)"
```

TCC.db 中的 `csreq` 字段存储的就是上述 requirement 的 DER 编码。

---

## 局限性与绕过

| 限制 | 说明 |
|------|------|
| 无法静默授权 | TCC 授权必须通过用户交互（弹窗或系统偏好设置） |
| MDM 例外 | 通过 MDM（Mobile Device Management）可以静默预授权，企业部署可用 |
| Automation 权限 | AppleScript 控制其他 App 需要单独的 TCC 授权 |
| 继承规则 | 子进程可以继承父进程的 TCC 授权（同 Bundle ID/签名） |
| SIP 保护 | TCC.db 本身受 SIP 保护，无法直接修改数据库文件 |

---

## 延伸阅读

- [App Sandbox 与自定义沙盒的冲突](./App%20Sandbox与自定义沙盒的冲突.md)
- [代码签名与 Entitlements](./代码签名与Entitlements.md)
- [FileProvider 文件投影](../04-filesystem/FileProvider文件投影.md) — 依赖 TCC 的全盘投影设计
