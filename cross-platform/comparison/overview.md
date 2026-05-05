# 操作系统整体架构对比：macOS vs Windows

## 内核对比

| 方面 | macOS (XNU) | Windows (NT) |
|------|-------------|--------------|
| 内核类型 | 混合内核 (Mach + BSD) | 混合内核 (微内核设计 + 宏内核实现) |
| 开源情况 | Darwin 部分开源 | 闭源 |
| 驱动框架 | IOKit (C++) / DriverKit (现代) | WDM / WDF (KMDF + UMDF) |
| 系统调用 | Mach traps + BSD syscalls | NTDLL → ntoskrnl (syscall/int 2e) |
| 调度器 | Mach 调度器 (优先级队列) | NT 调度器 (多级反馈队列) |

## 用户态架构

```
macOS:                              Windows:
┌─────────────────────┐            ┌─────────────────────┐
│   Application       │            │   Application       │
├─────────────────────┤            ├─────────────────────┤
│  Cocoa / SwiftUI    │            │  Win32 / WinRT / UWP│
├─────────────────────┤            ├─────────────────────┤
│  Core Frameworks    │            │  Windows API (Subsys)│
│  (Foundation, CF)   │            │  (kernel32, user32)  │
├─────────────────────┤            ├─────────────────────┤
│  libSystem (libc)   │            │  NTDLL.dll          │
├─────────────────────┤            ├─────────────────────┤
│  Mach + BSD syscall │            │  syscall / int 2Eh  │
╞═════════════════════╡            ╞═════════════════════╡
│  XNU Kernel         │            │  NT Kernel          │
└─────────────────────┘            └─────────────────────┘
```

## 安全模型对比

| 方面 | macOS | Windows |
|------|-------|---------|
| 权限提升 | Authorization Services + sudo | UAC (User Account Control) |
| 代码签名 | codesign + Notarization | Authenticode |
| 系统保护 | SIP (System Integrity Protection) | WRP (Windows Resource Protection) |
| 应用沙盒 | App Sandbox (entitlements) | AppContainer / WDAC |
| 密钥存储 | Keychain | DPAPI / Credential Manager |
| 全盘加密 | FileVault (Core Storage / APFS) | BitLocker |

## 进程间通信 (IPC) 对比

| 方面 | macOS | Windows |
|------|-------|---------|
| 原生 IPC | Mach Ports | ALPC (Advanced Local Procedure Call) |
| 高层 IPC | XPC Services | COM / DCOM |
| 管道 | Unix Pipes / Named Pipes | Named Pipes / Anonymous Pipes |
| 共享内存 | mmap / POSIX shm | File Mapping (CreateFileMapping) |
| 套接字 | Unix Domain Sockets | Unix Sockets (Win10+) / TCP loopback |
| 消息 | Distributed Notifications | Window Messages (WM_) / Mailslots |

## 文件系统对比

| 方面 | macOS (APFS) | Windows (NTFS) |
|------|--------------|----------------|
| 快照 | ✅ 原生支持 | VSS (Volume Shadow Copy) |
| 加密 | ✅ 卷级/文件级 | EFS / BitLocker |
| 压缩 | ✅ 透明压缩 | ✅ NTFS 压缩 |
| 硬链接 | ✅ | ✅ |
| 符号链接 | ✅ | ✅ (需权限) |
| 扩展属性 | ✅ xattr | ✅ Alternate Data Streams |
| 大小写 | 默认不敏感（可选敏感） | 不敏感 |
