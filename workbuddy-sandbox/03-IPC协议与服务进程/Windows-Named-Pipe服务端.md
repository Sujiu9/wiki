# 🟡 Windows Named Pipe 服务端

> 📋 前置知识：[NDJSON 协议从零实现](./NDJSON协议从零实现.md)

## 1. 为什么 Windows 使用 Named Pipe

Windows 没有 Unix Domain Socket 作为传统主路径，本机进程间通信常用 `命名管道（Named Pipe）`。

它的典型地址形如：

```text
\\.\pipe\WorkBuddy_Manager_12345
```

Named Pipe 适合做 `sandbox-cli` 这种本机服务：宿主启动服务进程，读取 IPC 地址，然后用管道连接并按行收发 NDJSON。

## 2. 解决方案概览

| 方案 | 简述 | 适用场景 |
|------|------|----------|
| Anonymous Pipe | 父子进程单向/双向管道 | 简单父子通信 |
| Named Pipe | 有名字的本机管道 | 本机 IPC 服务 |
| TCP localhost | 本机网络端口 | 需要统一网络模型 |
| COM/RPC | Windows 组件通信 | 系统服务/复杂接口 |

WorkBuddySandbox 选择 Named Pipe，是为了和 macOS UDS 在抽象上保持一致：都是“地址 + 字节流 + NDJSON”。

## 3. 方案对比

| 维度 | Named Pipe | TCP localhost | Anonymous Pipe |
|------|------------|---------------|----------------|
| 地址发现 | `\\.\pipe\name` | host:port | 进程句柄 |
| 本机权限 | Windows ACL | 网络绑定/防火墙 | 父子关系 |
| 多客户端 | 可支持 | 可支持 | 弱 |
| 跨语言 | 好 | 好 | 一般 |

## 4. 如何写一个最小版本

概念流程：

```text
CreateNamedPipe("\\.\\pipe\\mini-sandbox")
ConnectNamedPipe()
循环：
  ReadFile() 读字节
  按 \n 切分 NDJSON
  WriteFile() 写响应 + \n
客户端断开：
  DisconnectNamedPipe()
  CloseHandle()
```

C# 客户端最小模型：

```csharp
using var pipe = new NamedPipeClientStream(".", "mini-sandbox", PipeDirection.InOut);
pipe.Connect(3000);
using var writer = new StreamWriter(pipe) { AutoFlush = true };
using var reader = new StreamReader(pipe);

writer.WriteLine("{\"command\":\"Ping\",\"messageId\":\"1\"}");
var line = reader.ReadLine();
Console.WriteLine(line);
```

真实服务端通常需要 overlapped I/O 或异步读写，避免一个客户端阻塞整个服务。

## 5. 常见坑

- **地址表现不一致**：有的实现内部使用逻辑名，客户端再拼成 `\\.\pipe\name`。
- **阻塞等待连接**：服务端要能处理启动超时和退出信号。
- **编码问题**：NDJSON 建议统一 UTF-8。
- **半包/粘包**：Named Pipe 仍可能需要业务层按行解析。
- **句柄泄漏**：断连后必须关闭 pipe handle。

## 6. 回到 WorkBuddySandbox

| 机制 | 项目位置 |
|------|----------|
| Windows IPC | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/ipc/win.rs` |
| sandbox-cli 入口 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/bin/sandbox_cli.rs` |
| IPC 协议结构 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/ipc/protocol.rs` |
| Windows 测试客户端 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox-test/src/common/transport_win.rs` |

实现细节提醒：`src/sandbox/src/ipc/win.rs` 的 `address()` 返回的是逻辑名，测试客户端会拼接为 `\\.\pipe\<name>`。写宿主代码时要以实际接入文档和当前实现为准。

## 7. 练习题：你要亲手实现什么

1. 写一个 Named Pipe echo server：收到一行，返回一行。
2. 在 C# 客户端发送 `Ping`，读取 `{success:true}`。
3. 加入 `messageId`，并发发送 3 个请求，确认响应可匹配。
4. 模拟服务端提前退出，客户端给出清晰错误。

## 8. 延伸阅读

- [Windows DLL 注入与 API Hook 最小实现](../01-进程与沙盒/Windows-DLL注入与API-Hook最小实现.md)
- [Rust cdylib 与 C ABI 最小示例](../04-Rust跨语言边界/Rust-cdylib与C-ABI最小示例.md)
