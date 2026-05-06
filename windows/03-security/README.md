# Windows 安全机制

## 概览

Windows 的安全模型基于 DAC（自主访问控制）+ 完整性级别，与 macOS 的 MAC（强制访问控制）设计理念不同。核心组件包括安全令牌、ACL、UAC 和 AppContainer。

## 核心安全机制

| 机制 | 用途 | 层级 |
|------|------|------|
| Security Token | 进程身份标识（SID、特权、完整性级别） | 进程 |
| ACL (DACL/SACL) | 对象访问控制列表 | 对象 |
| UAC | 用户账户控制/管理员提权 | 用户交互 |
| AppContainer | UWP 应用沙盒（类似 App Sandbox） | 应用 |
| DLL 注入 + Hook | 用户态系统调用拦截 | 进程（非标准） |
| Minifilter | 内核级文件系统过滤 | 内核驱动 |

## 目录

- [用户态 Hook 与 DLL 注入](./用户态Hook与DLL注入.md) 🔴 — ntdll inline-hook、进程注入、子进程传播
- [Windows 安全令牌与 ACL](./安全令牌与ACL.md) 🟡
- [UAC 与完整性级别](./UAC与完整性级别.md) 🟡
