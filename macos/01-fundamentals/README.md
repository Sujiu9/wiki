# macOS 基础概念

## 概览

macOS 基于 Darwin 操作系统，其内核 XNU（X is Not Unix）是一个混合内核，结合了：
- **Mach 微内核** — 负责进程/线程管理、IPC、虚拟内存
- **BSD 层** — 提供 POSIX API、文件系统、网络栈
- **IOKit** — 设备驱动框架（C++ 实现）

## 目录

- [进程与线程模型](./进程与线程模型.md) 🟡 — Mach Task/Thread、fork/exec、进程组/会话、PTY
- [内核架构](./内核架构.md) 🟢
- [内存管理](./内存管理.md) 🟡
- [用户态与内核态](./用户态与内核态.md) 🟡

## 关键术语

| 术语 | 英文 | 说明 |
|------|------|------|
| 混合内核 | Hybrid Kernel | XNU 结合了微内核和宏内核的特点 |
| Mach 端口 | Mach Port | Mach 内核的基本 IPC 原语 |
| 任务 | Task | Mach 中资源容器的概念（对应 BSD 进程） |
| 系统扩展 | System Extension | 取代内核扩展(kext)的现代方案 |
| 沙盒 | Sandbox (App Sandbox) | 限制应用访问系统资源的安全机制 |
