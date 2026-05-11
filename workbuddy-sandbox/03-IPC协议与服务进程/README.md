# 🟢 IPC 协议与服务进程

> 📋 前置知识：[从 fork/exec 到进程隔离](../01-进程与沙盒/从fork-exec到进程隔离.md)

本模块学习一个核心能力：**如何把一个沙盒核心做成独立服务进程，让任意语言宿主通过协议调用它**。

## 为什么需要服务进程

如果 Swift、C#、Node.js 都直接链接 Rust 库，会遇到：

- ABI 和动态库加载复杂；
- 崩溃会拖垮宿主；
- 多语言都要写一套绑定；
- 异步事件不好统一。

所以 WorkBuddySandbox 推荐 `sandbox-cli`：宿主只需要启动一个子进程，然后通过 IPC 发 JSON 消息。

## 学习顺序

1. [NDJSON 协议从零实现](./NDJSON协议从零实现.md)：先理解“一行 JSON 一个消息”。
2. [Unix Domain Socket 服务端](./Unix-Domain-Socket服务端.md)：macOS/Linux 下如何做本机 IPC。
3. [Windows Named Pipe 服务端](./Windows-Named-Pipe服务端.md)：Windows 下如何做等价通信。

## 方案对比

| 方案 | 优点 | 缺点 |
|------|------|------|
| 直接 FFI | 调用开销低 | ABI、内存、崩溃边界复杂 |
| HTTP 本地服务 | 调试方便 | 端口管理、安全暴露面更大 |
| UDS / Named Pipe | 本机 IPC，权限边界清晰 | 双平台要分别实现 |
| stdin/stdout JSON | 最简单 | 生命周期和并发能力弱 |

`sandbox-cli` 选择 UDS / Named Pipe + NDJSON，是在跨语言、可调试、可扩展之间的折中。

## WorkBuddySandbox 映射

| 能力 | 项目路径 |
|------|----------|
| 接入文档 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/bin/README.md` |
| CLI 入口 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/bin/sandbox_cli.rs` |
| 协议结构体 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/ipc/protocol.rs` |
| Unix Socket | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/ipc/unix.rs` |
| Named Pipe | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/ipc/win.rs` |
| 命令路由 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/ipc/command_route.rs` |

## 学完本模块你应该能写什么

- 一个最小 NDJSON 请求/响应协议。
- 一个 UDS 服务端和客户端。
- 一个 Named Pipe 服务端和客户端。
- 一个支持 `messageId` 关联响应、未知字段兼容、异步通知的协议模型。

## 延伸阅读

- [Rust cdylib 与 C ABI 最小示例](../04-Rust跨语言边界/Rust-cdylib与C-ABI最小示例.md)
- [底层日志与排障方法](../07-调试与系统权限/底层日志与排障方法.md)
