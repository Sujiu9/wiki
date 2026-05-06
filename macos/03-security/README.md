# macOS 安全机制

## 概览

macOS 拥有多层安全机制，从内核级别的 SIP（系统完整性保护）到应用级别的 App Sandbox，构成了纵深防御体系。

## 核心安全机制

| 机制 | 用途 | 层级 |
|------|------|------|
| SIP (System Integrity Protection) | 保护系统文件不被修改 | 内核 |
| App Sandbox | 限制应用访问系统资源 | 应用 |
| sandbox-exec + SBPL | 细粒度进程沙盒 | 进程 |
| Gatekeeper | 阻止未签名/公证的应用运行 | 应用启动 |
| TCC (Transparency, Consent, Control) | 隐私权限管理 | 系统服务 |
| Keychain | 安全凭证存储 | 系统服务 |

## 目录

- [进程沙盒机制](./进程沙盒机制.md) 🟡
- [App Sandbox 与自定义沙盒的冲突](./App%20Sandbox与自定义沙盒的冲突.md) 🔴
- [TCC 权限管理机制](./TCC权限管理机制.md) 🟡 — TCC 数据库、授权流程、Full Disk Access
- [代码签名与 Entitlements](./代码签名与Entitlements.md) 🟡 — 签名层次、Team ID、App Group
- [SIP 系统完整性保护](./SIP系统完整性保护.md) 🟡
