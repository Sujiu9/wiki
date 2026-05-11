# 🟡 Unix Domain Socket 服务端

> 📋 前置知识：[NDJSON 协议从零实现](./NDJSON协议从零实现.md)

## 1. 为什么需要 Unix Domain Socket

宿主和 `sandbox-cli` 在同一台机器上通信，不一定需要 TCP 端口。`Unix Domain Socket（UDS）` 是 Unix/macOS 上的本机 IPC：

- 通过文件路径寻址，如 `/tmp/WorkBuddy_Manager_123.sock`；
- 不暴露网络端口；
- 可利用文件权限控制访问；
- 和普通 socket 一样能读写字节流。

## 2. 解决方案概览

| 方案 | 简述 | 适用场景 |
|------|------|----------|
| stdin/stdout | 父子进程管道 | 单客户端、简单交互 |
| UDS | 本机 socket 文件 | 本机多语言 IPC |
| TCP localhost | 本机端口 | 需要浏览器/HTTP 调试 |
| XPC | macOS 原生服务机制 | App/系统服务集成 |

WorkBuddySandbox 选择 UDS 作为 macOS/Linux 主 IPC，是因为它简单、跨语言、权限边界清晰。

## 3. 方案对比

| 维度 | UDS | TCP localhost | stdin/stdout |
|------|-----|---------------|--------------|
| 端口占用 | 无 | 有 | 无 |
| 地址发现 | 文件路径 | host:port | 子进程句柄 |
| 多客户端 | 可支持 | 可支持 | 弱 |
| 权限控制 | 文件权限 | 防火墙/绑定地址 | 父子进程关系 |

## 4. 如何写一个最小版本

Rust 服务端最小模型：

```rust
use std::os::unix::net::UnixListener;
use std::io::{BufRead, BufReader, Write};

let path = "/tmp/mini-sandbox.sock";
let _ = std::fs::remove_file(path); // 清理旧 socket 文件
let listener = UnixListener::bind(path)?;
println!("IPC_ADDRESS={}", path);

let (mut stream, _) = listener.accept()?;
let mut reader = BufReader::new(stream.try_clone()?);
let mut line = String::new();
while reader.read_line(&mut line)? > 0 {
    stream.write_all(b"{\"success\":true}\n")?;
    line.clear();
}
```

这个模型已经包含了 `sandbox-cli` 的关键启动动作：绑定地址、打印地址、接受连接、按行读写。

## 5. 常见坑

- **旧 socket 文件未删除**：上次异常退出后路径残留，bind 会失败。
- **地址过长**：部分系统对 UDS 路径长度有限制。
- **stdout 顺序**：宿主必须等到 `IPC_ADDRESS=` 后再连接。
- **断连清理**：客户端断开后服务端要释放 session、投影、子进程等资源。
- **半包问题**：UDS 仍是字节流，业务层还是要 NDJSON framing。

## 6. 回到 WorkBuddySandbox

| 机制 | 项目位置 |
|------|----------|
| UDS Server | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/ipc/unix.rs` |
| CLI 打印地址 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/bin/sandbox_cli.rs` |
| 主读循环 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/manager.rs` |
| macOS 宿主通道 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/SandboxMain/Sources/SandboxMain/SandboxChannel.swift` |
| SandboxExt 反向通道 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/SandboxExt/Sources/IPC/SandboxExtIPCServer.swift` |

注意：WorkBuddySandbox 除了主 IPC，还有 macOS 扩展反向 IPC，用来让 Rust 查询 FileProvider/NetworkExtension 状态。

## 7. 练习题：你要亲手实现什么

1. 写一个 UDS 服务端，启动后打印 `IPC_ADDRESS=`。
2. 写一个客户端连接它，发送一行 JSON，读取一行响应。
3. 模拟客户端异常断开，服务端打印 `client disconnected` 并退出。
4. 加入 60 秒无人连接自动退出。

## 8. 延伸阅读

- [NDJSON 协议从零实现](./NDJSON协议从零实现.md)
- [底层日志与排障方法](../07-调试与系统权限/底层日志与排障方法.md)
