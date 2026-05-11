# 🟢 NDJSON 协议从零实现

> 📋 前置知识：能读写 JSON，理解请求/响应。

## 1. 为什么需要 NDJSON

本机 IPC 传输的是字节流，不会自动告诉你“一个消息从哪里开始，到哪里结束”。如果连续发送两个 JSON：

```json
{"command":"Ping"}{"command":"OpenSession"}
```

接收端很难直接切分。

`NDJSON（Newline-Delimited JSON）` 的规则很简单：**一行就是一个完整 JSON 消息**。换行符 `\n` 就是消息边界。

## 2. 解决方案概览

| 方案 | 消息边界 | 适用场景 |
|------|----------|----------|
| 固定长度头 | 先读长度，再读 body | 高性能二进制协议 |
| 分隔符 | 用 `\n` 或特殊字符分隔 | 文本协议、调试友好 |
| HTTP | 请求/响应天然有边界 | 跨进程/跨机器服务 |
| Protobuf stream | 自定义 framing | 强类型二进制协议 |

WorkBuddySandbox 使用 NDJSON，因为它足够简单，Swift/C#/Node/Rust 都容易实现，也方便日志排查。

## 3. 方案对比

| 维度 | NDJSON | 长度头二进制 | HTTP |
|------|--------|--------------|------|
| 可读性 | 高 | 低 | 高 |
| 实现复杂度 | 低 | 中 | 中 |
| 性能 | 中 | 高 | 中 |
| 本机 IPC 适配 | 好 | 好 | 一般 |

沙盒命令不是高频交易协议，可读性和兼容性比极致性能更重要。

## 4. 如何写一个最小版本

最小请求：

```json
{"sessionId":"sess-1","messageId":"req-1","command":"Ping","data":{}}
```

最小响应：

```json
{"sessionId":"sess-1","messageId":"req-1","success":true,"data":{}}
```

关键规则：

1. 请求必须有 `messageId`；
2. 响应原样带回 `messageId`；
3. 响应通过 `success` 字段区分成功/失败；
4. 通知没有 `success` 字段；
5. 接收端要忽略未知字段，便于协议升级。

Rust 读写循环的最小模型：

```rust
for line in reader.lines() {
    let msg: serde_json::Value = serde_json::from_str(&line?)?;
    let id = msg["messageId"].clone();

    let resp = serde_json::json!({
        "messageId": id,
        "success": true,
        "data": {"pong": true}
    });

    writer.write_all(resp.to_string().as_bytes())?;
    writer.write_all(b"\n")?;
}
```

## 5. 常见坑

- **忘记换行**：对端一直等不到完整消息。
- **一次 read 当成一条消息**：字节流可能半包/粘包，必须按行切分。
- **messageId 不唯一**：并发请求会串响应。
- **把错误结构强绑定**：错误文本应主要给人看，不要依赖解析错误字符串。
- **不兼容未知字段**：协议一升级，旧宿主就崩。

## 6. 回到 WorkBuddySandbox

| 机制 | 项目位置 |
|------|----------|
| 协议说明 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/bin/README.md` |
| 协议结构 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/ipc/protocol.rs` |
| 命令路由 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/ipc/command_route.rs` |
| 协议 proto | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/proto/sandbox_ipc.proto` |

`sandbox-cli` 启动后先在 stdout 打印 `IPC_ADDRESS=...`，这不是 NDJSON 业务消息，而是给宿主发现 IPC 地址的启动握手。

## 7. 练习题：你要亲手实现什么

1. 写一个 `mini-ipc-server`：收到 `Ping` 返回 `{success:true}`。
2. 加入 `messageId` 映射：客户端能并发发 3 个请求并正确匹配响应。
3. 加入通知：服务端主动发送 `{command:"processExit"}`，客户端能区分它不是响应。
4. 增加一个未知字段，验证旧客户端不会崩。

## 8. 延伸阅读

- [Unix Domain Socket 服务端](./Unix-Domain-Socket服务端.md)
- [Windows Named Pipe 服务端](./Windows-Named-Pipe服务端.md)
