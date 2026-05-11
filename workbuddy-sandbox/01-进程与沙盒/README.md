# 🟢 进程与沙盒

> 📋 前置知识：会使用命令行启动程序，理解环境变量、工作目录、标准输入输出。

本模块不从 WorkBuddySandbox 的架构图开始，而是从“如何自己写一个受控进程启动器”开始。

## 你要解决的问题

AI Agent 会执行命令、读写文件、访问网络。如果直接让它运行在宿主环境里，就会遇到三个问题：

1. **边界不清**：它能访问哪些文件？哪些网络？
2. **结果不可控**：进程超时、卡住、输出爆炸怎么办？
3. **审计困难**：它到底被拦截了什么、为什么失败？

沙盒执行的目标不是“神秘地保护系统”，而是把一次命令执行拆成可控制的输入、环境、权限、输出、退出状态。

## 学习顺序

1. [从 fork/exec 到进程隔离](./从fork-exec到进程隔离.md)：先学会手写最小进程启动器。
2. [macOS sandbox-exec 最小实现](./macOS-sandbox-exec最小实现.md)：理解 SBPL profile 如何限制文件访问。
3. [Windows DLL 注入与 API Hook 最小实现](./Windows-DLL注入与API-Hook最小实现.md)：理解 Windows 下为什么常用 Hook 做拦截。

## 方案对比

| 方案 | 解决什么 | 优点 | 局限 |
|------|----------|------|------|
| 普通子进程 | 启动、等待、收集输出 | 最简单，所有平台都有 | 不能限制文件/网络 |
| macOS `sandbox-exec` | 文件/系统调用级限制 | 系统原生，profile 可读 | 能力受系统策略限制，调试依赖日志 |
| Windows Hook | 拦截 API 调用 | 能针对文件/网络 API 做细粒度记录 | 实现复杂，必须处理注入、线程、安全边界 |

## WorkBuddySandbox 映射

| 能力 | 项目路径 |
|------|----------|
| CLI 入口 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/bin/sandbox_cli.rs` |
| 会话管理 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/manager.rs` |
| 执行命令 Handler | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/sandbox/handlers/run_executable.rs` |
| 跨平台执行入口 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/executor/sandbox_executor/mod.rs` |
| macOS 执行 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/executor/sandbox_executor/mac.rs` |
| Windows 执行 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/executor/sandbox_executor/win.rs` |

## 学完本模块你应该能写什么

- 一个能启动命令、设置 `cwd/env`、收集 `stdout/stderr`、处理超时的进程启动器。
- 一个最小 `sandbox-exec` profile，并验证文件访问被拒绝。
- 一个 Windows API Hook 的概念模型：知道 Hook 点在哪里、为什么能拦截、拦截记录如何回传。

## 延伸阅读

- [IPC 协议与服务进程](../03-IPC协议与服务进程/README.md)
- [底层日志与排障方法](../07-调试与系统权限/底层日志与排障方法.md)
