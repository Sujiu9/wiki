# 🟡 macOS 代码签名与 Entitlements

> 📋 前置知识：[进程沙盒机制](./进程沙盒机制.md)

## 为什么需要代码签名？

代码签名解决的核心问题是**信任链**：

1. **来源可信**：这个二进制确实来自某个已知开发者
2. **完整性**：二进制文件没有被篡改过
3. **权限声明**：Entitlements 声明了此程序需要的系统权限

在 AI Agent 沙盒项目中，代码签名直接影响：
- NE Extension 与主 App 能否通过 App Group 共享数据（需要同一 Team ID）
- sandbox-exec 子进程是否能被正确识别为受信代码
- System Extension 能否被系统加载

---

## 签名层次体系

```
Apple Root CA
  │
  ├── Apple Worldwide Developer Relations CA
  │     │
  │     ├── Developer ID Application: Company (TEAMID)
  │     │     └── 用于 App Store 外分发
  │     │
  │     ├── Apple Distribution: Company (TEAMID)
  │     │     └── 用于 App Store 分发
  │     │
  │     └── Apple Development: developer@email.com (TEAMID)
  │           └── 用于开发/调试
  │
  └── Apple Code Signing CA
        └── 系统二进制（/usr/bin/* 等）
```

### 签名类型

| 签名类型 | 用途 | App Sandbox 要求 | 分发渠道 |
|---------|------|-----------------|---------|
| Developer ID | macOS 直接分发 | **不强制** | 官网/DMG |
| Apple Distribution | Mac App Store | **强制** | App Store |
| Apple Development | 开发调试 | 可选 | Xcode 直接安装 |
| Ad Hoc | 无证书临时签名 | 不可用 | 手动安装 |

---

## Mach-O 签名结构

代码签名嵌入在 Mach-O 二进制文件中：

```
Mach-O Binary
├── __TEXT segment (代码段)
├── __DATA segment (数据段)
├── __LINKEDIT segment
│     └── Code Signature
│           ├── Code Directory (CD)
│           │     ├── 版本、标志位
│           │     ├── 每页 Hash（4KB 页粒度）
│           │     ├── Info.plist Hash
│           │     ├── Entitlements Hash
│           │     └── Requirements Hash
│           │
│           ├── CMS Signature (PKCS#7)
│           │     └── 证书链 + 开发者私钥签名
│           │
│           ├── Entitlements (嵌入式)
│           │     └── XML plist 格式的权限声明
│           │
│           └── Requirements (代码要求)
│                 └── 编译后的 requirement 表达式
```

### 内核验证流程

```c
// XNU 内核中 exec 时的签名验证（简化）
int verify_code_signature(vnode_t vp, proc_t proc) {
    // 1. 读取 __LINKEDIT 中的 Code Signature
    cs_blob_t *blob = ubc_cs_blob_get(vp);
    
    // 2. 验证 CMS 签名（证书链 → Apple Root CA）
    if (!cms_verify(blob->cms_signature))
        return EBADEXEC;  // 签名无效
    
    // 3. 逐页验证 Hash（按需验证，不是启动时全部检查）
    // 当某页被 page fault 加载时才验证
    
    // 4. 提取 Entitlements
    proc->p_entitlements = parse_entitlements(blob->entitlements);
    
    // 5. 根据 Entitlements 生成/叠加 sandbox profile
    if (proc->p_entitlements->app_sandbox) {
        sandbox_profile_t *profile = generate_sandbox_profile(proc->p_entitlements);
        sandbox_apply(proc, profile);
    }
    
    return 0;
}
```

**关键发现**：Entitlements → Sandbox Profile 的转换发生在**内核中的 exec 路径**，这就是为什么 Entitlements 一旦编译进去就无法运行时修改。

---

## Entitlements 深入解析

### Entitlements 是什么？

Entitlements 是嵌入在代码签名中的 **XML plist**，声明进程需要的系统权限：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" ...>
<plist version="1.0">
<dict>
    <!-- 核心：是否启用 App Sandbox -->
    <key>com.apple.security.app-sandbox</key>
    <true/>
    
    <!-- 网络权限 -->
    <key>com.apple.security.network.client</key>
    <true/>
    <key>com.apple.security.network.server</key>
    <true/>
    
    <!-- 文件访问 -->
    <key>com.apple.security.files.user-selected.read-write</key>
    <true/>
    
    <!-- App Group（跨进程共享） -->
    <key>com.apple.security.application-groups</key>
    <array>
        <string>group.com.example.myapp</string>
    </array>
    
    <!-- System Extension -->
    <key>com.apple.developer.system-extension.install</key>
    <true/>
    
    <!-- Network Extension -->
    <key>com.apple.developer.networking.networkextension</key>
    <array>
        <string>app-proxy-provider</string>
        <string>content-filter-provider</string>
        <string>packet-tunnel-provider</string>
    </array>
</dict>
</plist>
```

### Entitlements 与 Sandbox Profile 的映射

内核将 Entitlements 翻译为 SBPL 规则：

| Entitlement | 生成的 SBPL 规则 |
|------------|-----------------|
| `network.client = true` | `(allow network-outbound)` |
| `network.server = true` | `(allow network-inbound)` |
| `files.user-selected.read-write = true` | `(allow file-read* file-write* (require-any (vnode-type REGULAR-FILE) ...))` + Powerbox 授权 |
| `files.downloads.read-write = true` | `(allow file-read* file-write* (subpath "~/Downloads"))` |
| `app-sandbox = true` | `(deny default)` + 基础权限集 |

### 不可逆性的内核实现

```c
// 内核中 apply sandbox 是单向操作
void sandbox_apply(proc_t proc, sandbox_profile_t *profile) {
    // 添加到进程的 sandbox set 中
    proc->p_sandbox = sandbox_set_add(proc->p_sandbox, profile);
    
    // 注意：没有 sandbox_remove() 函数！
    // 没有任何内核接口可以移除已 apply 的 profile
}
```

---

## Team ID 与 App Group

### Team ID 的作用

Team ID 是开发者/公司的唯一标识，所有同一团队签名的 App 共享相同的 Team ID：

```
证书 Subject: Developer ID Application: Tencent (TEAMID123)
                                                  ^^^^^^^^^^
                                                  Team ID
```

### App Group 的工作机制

App Group 允许同一 Team ID 下的多个进程共享数据：

```
主 App (com.tencent.workbuddy)          Team ID: ABCDEF
NE Extension (com.tencent.workbuddy.ne) Team ID: ABCDEF  ← 必须相同
FileProvider (com.tencent.workbuddy.fp) Team ID: ABCDEF  ← 必须相同

共享目录: ~/Library/Group Containers/group.com.tencent.workbuddy/
                                     ↑ Entitlements 中声明的 group ID
```

### 内核如何验证 App Group 访问权限

```c
// 简化的 App Group 访问检查
int check_app_group_access(proc_t proc, const char *group_id) {
    // 1. 从进程的 Entitlements 中读取 application-groups 数组
    array_t groups = proc->p_entitlements->application_groups;
    
    // 2. 检查请求的 group_id 是否在列表中
    if (!array_contains(groups, group_id))
        return EACCES;
    
    // 3. 验证进程的 Team ID 与 group 所有者的 Team ID 一致
    team_id_t proc_team = get_team_id(proc->p_code_signature);
    team_id_t group_team = get_group_owner_team(group_id);
    if (proc_team != group_team)
        return EACCES;
    
    return 0;
}
```

---

## 实战：AgentOS 的签名要求

### 各组件的签名配置

| 组件 | 签名类型 | App Sandbox | 关键 Entitlements |
|------|---------|-------------|-------------------|
| 主 App | Developer ID | ❌ false | system-extension.install, application-groups |
| NE Extension | Developer ID | ✅ true（强制） | networking.networkextension, application-groups |
| FileProvider | Developer ID | ✅ true（强制） | fileprovider.domain, application-groups |
| sandbox-cli | Developer ID | ❌ false | application-groups |

### 为什么 NE Bridge 要检查 Team ID？

```rust
// ne_bridge.rs 中的检查
pub fn new(app_group: &str) -> Self {
    // FFI 调用 Objective-C 检查
    let available = unsafe { ne_bridge_check_app_group(group_cstr.as_ptr()) };
    // 如果当前进程的签名 Team ID 与 App Group 所有者不匹配
    // available = false → NE Bridge 不可用
}
```

如果 sandbox-cli 使用了不同 Team ID 的证书签名，它就无法读写 App Group 中的 NE 规则文件 → 网络拦截功能断裂。

---

## 签名验证时机

| 时机 | 验证内容 | 失败后果 |
|------|---------|---------|
| exec 加载 | 证书链完整性 | EBADEXEC，无法启动 |
| 页面调入 (page-in) | 逐页 Hash | SIGKILL（Code Signature Invalid） |
| 运行时 | Library Validation | 无法加载未签名/签名不匹配的 dylib |
| TCC 查询 | csreq 匹配 | TCC 授权被拒绝 |
| App Group 访问 | Team ID 匹配 | 无法读写共享容器 |

---

## 延伸阅读

- [TCC 权限管理机制](./TCC权限管理机制.md)
- [进程沙盒机制](./进程沙盒机制.md)
- [App Sandbox 与自定义沙盒的冲突](./App%20Sandbox与自定义沙盒的冲突.md)
