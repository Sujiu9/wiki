# Windows 基础概念

## 概览

Windows 基于 NT（New Technology）内核，采用混合内核架构：
- **NT 内核 (ntoskrnl.exe)** — 线程调度、中断处理、底层同步
- **执行体 (Executive)** — 内存管理、I/O 管理、对象管理、进程管理
- **HAL (硬件抽象层)** — 屏蔽硬件差异
- **Win32 子系统 (csrss.exe / win32k.sys)** — 提供 Windows API

## 目录

- [NT 内核架构](./nt-architecture.md) 🟢
- [进程与线程](./process-and-thread.md) 🟢
- [内存管理](./memory-management.md) 🟡
- [用户态与内核态](./user-kernel-space.md) 🟡
- [Windows 对象模型](./object-model.md) 🟡

## 关键术语

| 术语 | 英文 | 说明 |
|------|------|------|
| NT 内核 | NT Kernel | Windows 操作系统核心 |
| 执行体 | Executive | 内核之上的高层系统服务 |
| HAL | Hardware Abstraction Layer | 硬件抽象层 |
| 会话 | Session | 用户登录会话，隔离用户资源 |
| 子系统 | Subsystem | 为应用提供 API 的环境（如 Win32） |
| 驱动 | Driver (WDM/WDF/KMDF/UMDF) | 内核/用户态驱动程序框架 |
| 服务 | Windows Service | 后台运行的长期进程 |
