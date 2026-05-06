# 跨平台 IPC 通信机制：Unix Domain Socket vs Named Pipe

## 为什么需要进程间通信？

在现代软件架构中，一个功能往往由**多个进程**协作完成：

- GUI 主程序 + 后台计算引擎（不同语言实现）
- Extension/Plugin 进程 + 宿主进程
- Agent 子进程 + 沙盒管理器

进程之间的内存空间是**完全隔离**的（虚拟内存机制保证），需要专门的通信通道。

> 在 AI Agent 沙盒场景中：Swift/C# 宿主 ↔ Rust 沙盒核心 ↔ Agent 子进程，三者需要高效、可靠的 IPC 机制。

---

## 解决方案概览

| 方案 | 平台 | 传输方式 | 适用场景 |
|------|------|----------|----------|
| Unix Domain Socket | macOS/Linux | 本地字节流/数据报 | 本机进程通信，类似网络 socket |
| Named Pipe | Windows | 命名管道 | Windows 进程通信 |
| Mach Ports | macOS | 内核消息队列 | XPC、底层系统通信 |
| ALPC | Windows | 内核消息传递 | Windows 内部系统调用 |
| 共享内存 + 信号量 | 跨平台 | 直接内存映射 | 超大数据量、低延迟 |
| TCP Loopback | 跨平台 | 127.0.0.1 TCP | 简单但有端口冲突风险 |

---

## 方案对比

| 维度 | Unix Domain Socket | Named Pipe | TCP Loopback | 共享内存 |
|------|-------------------|------------|--------------|----------|
| 性能 | 高（无网络栈开销） | 高 | 中（经过完整网络栈） | 极高 |
| 安全性 | 文件权限控制 | ACL 控制 | 任何进程可连接 | 需额外同步 |
| 双向通信 | ✅ 天然全双工 | ⚠️ 需两个管道或 DUPLEX 模式 | ✅ | ✅ 但需要信号通知 |
| 连接管理 | accept/connect 模型 | ConnectNamedPipe | accept/connect | 无连接概念 |
| 多客户端 | ✅ | ✅ | ✅ | 复杂 |
| 跨机器 | ❌ 仅本机 | ⚠️ 支持但少用 | ✅ | ❌ |

---

## Unix Domain Socket（macOS/Linux）

### 原理

Unix Domain Socket 在内核中创建一个通信通道，通过**文件系统路径**标识（而非 IP:Port）：

```
/tmp/my-app.sock          ← 路径就是"地址"
```

与 TCP socket 使用相同的 API（`socket/bind/listen/accept/connect`），但：
- 不经过网络栈（不拆包、不路由、不做校验和）
- 使用文件权限控制访问
- 性能比 TCP loopback 高约 2x

### 代码示例

```rust
// Rust 服务端（tokio 异步）
use tokio::net::UnixListener;

let socket_path = format!("/tmp/WorkBuddy_Manager_{}.sock", std::process::id());
let listener = UnixListener::bind(&socket_path)?;

loop {
    let (stream, _) = listener.accept().await?;
    tokio::spawn(async move {
        handle_connection(stream).await;
    });
}
```

```swift
// Swift 客户端
import Foundation

let socket = socket(AF_UNIX, SOCK_STREAM, 0)
var addr = sockaddr_un()
addr.sun_family = sa_family_t(AF_UNIX)
let path = "/tmp/WorkBuddy_Manager_\(pid).sock"
path.withCString { ptr in
    withUnsafeMutablePointer(to: &addr.sun_path.0) { dest in
        strcpy(dest, ptr)
    }
}
connect(socket, &addr, socklen_t(MemoryLayout<sockaddr_un>.size))
```

### 路径命名约定

```bash
/tmp/WorkBuddy_Manager_<pid>.sock    # 包含 PID 避免冲突
```

---

## Named Pipe（Windows）

### 原理

Windows Named Pipe 是内核对象，通过特殊路径 `\\.\pipe\<name>` 标识：

```
\\.\pipe\WorkBuddy_Manager_12345     ← 管道名称
```

### 代码示例

```rust
// Rust 服务端（Windows）
use tokio::net::windows::named_pipe::ServerOptions;

let pipe_name = format!(r"\\.\pipe\WorkBuddy_Manager_{}", std::process::id());
let server = ServerOptions::new()
    .first_pipe_instance(true)
    .create(&pipe_name)?;

server.connect().await?;  // 等待客户端连接
```

```csharp
// C# 客户端
using var client = new NamedPipeClientStream(
    ".", $"WorkBuddy_Manager_{pid}", PipeDirection.InOut, PipeOptions.Asynchronous);
await client.ConnectAsync(timeout: 5000);
```

---

## NDJSON 协议层

在传输层（UDS/Named Pipe）之上，使用 **NDJSON**（Newline Delimited JSON）作为消息格式：

### 为什么选择 NDJSON？

| 对比 | NDJSON | Protobuf | MessagePack |
|------|--------|----------|-------------|
| 可调试性 | ✅ 人类可读 | ❌ 二进制 | ❌ 二进制 |
| 无需 schema | ✅ | ❌ 需 .proto | ⚠️ |
| 多语言支持 | ✅ 任何语言原生 JSON | ⚠️ 需代码生成 | ⚠️ 需库 |
| 性能 | 中 | 高 | 高 |
| 帧分界 | `\n` 即边界 | 需长度前缀 | 需长度前缀 |

对于 IPC 场景（非高频数据流），可调试性 > 极致性能。

### 消息格式

```json
{"sessionId":"s1","messageId":"m001","command":"runExecutable","data":{"cmd":"ls","args":["-la"]}}
{"sessionId":"s1","messageId":"m001","success":true,"data":{"exitCode":0,"stdout":"..."}}
```

### 区分请求与响应

```
请求：没有 "success" 字段
响应：有 "success" 字段（true/false）
```

### 反向命令（Rust → 宿主）

IPC 是**双向**的，Rust 核心也会主动发请求给宿主：

| 方向 | 示例命令 | 用途 |
|------|----------|------|
| 宿主 → Rust | `runExecutable` | 执行命令 |
| 宿主 → Rust | `core.merge.apply` | 提交变更 |
| Rust → 宿主 | `StartProjection` | 请求启动文件投影 |
| Rust → 宿主 | `host.trash.move_batch` | 请求移入回收站 |
| Rust → 宿主 | `host.audit.request_audit` | 请求用户审批 |

---

## 异步架构设计

### 避免死锁：长耗时操作的处理

某些操作（如用户审批）可能耗时 30 分钟，如果在 IPC 读循环中同步等待会导致整个通道阻塞。

解决方案：**异步 Worker + 延迟响应**

```
1. IPC read loop 收到 "core.merge.apply" 请求
2. 准备阶段持锁构造必要数据
3. spawn 独立 worker task 后立即释放锁
4. IPC read loop 不返回响应（继续处理其他消息）
5. worker 完成后用原 messageId 自发响应
```

```rust
// 伪代码
async fn cmd_merge_apply(msg: Message, ipc: IpcHandle) {
    let data = {
        let lock = state.lock();
        lock.prepare_merge_data()  // 持锁准备数据
    };  // 锁释放
    
    // spawn worker，不阻塞 read loop
    tokio::spawn(async move {
        let result = do_merge(data).await;  // 可能耗时很长
        ipc.send_response(msg.message_id, result).await;  // 用原 ID 回复
    });
    
    // 不返回响应，read loop 继续
}
```

### IpcHandle 注入模式

执行层通过 `IpcHandle` trait 与 IPC 通信，不直接持有 manager 引用：

```rust
/// 执行层的 IPC 接口
trait IpcHandle: Send + Sync {
    async fn send_request_and_wait(&self, command: &str, data: Value) -> Result<Value>;
    async fn send_response(&self, message_id: &str, result: Result<Value>);
}
```

好处：
- 执行层可独立单元测试（mock IpcHandle）
- 避免循环引用
- 解耦消息路由与业务逻辑

---

## 三层分发架构

```
┌──────────────────────────────────────────────┐
│  IPC Layer (manager.rs)                       │
│  • 纯消息收发与分发                            │
│  • 不含业务逻辑                               │
├──────────────────────────────────────────────┤
│  Coordination Layer (sandbox.rs)              │
│  • 统一协调 permission/executor/overlay        │
│  • 状态管理                                   │
├──────────────────────────────────────────────┤
│  Execution Layer (executor/)                  │
│  • 所有命令执行的具体实现                       │
│  • 拦截事件解析                               │
└──────────────────────────────────────────────┘
```

---

## 延伸阅读

- [macOS 进程沙盒机制](../../macos/03-security/进程沙盒机制.md)
- [跨平台对比：文件投影技术](./文件投影技术对比.md)
- [操作系统整体架构对比](./操作系统整体架构对比.md)
