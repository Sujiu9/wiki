# 🟢 从 fork/exec 到进程隔离

> 📋 前置知识：理解命令行、环境变量、退出码。

## 1. 为什么需要进程启动器

沙盒的第一步不是“限制权限”，而是**可控地启动一个进程**。

如果只是手动执行：

```bash
npm test
```

你拿不到完整的控制能力：无法统一设置工作目录、无法限制超时、无法稳定采集输出、无法把这次执行和一个会话 ID 绑定。WorkBuddySandbox 要管理 Agent 子进程，所以必须先把“执行命令”抽象成一个可编程对象。

## 2. 解决方案概览

| 方案 | 说明 | 适用场景 |
|------|------|----------|
| Shell 执行 | `sh -c "cmd"` 或 `cmd.exe /c` | 方便，但转义复杂 |
| 直接 exec | 指定 `executable + args` | 更安全，可控性更强 |
| PTY 执行 | 分配伪终端（Pseudo-Terminal） | 交互式命令，如 REPL |
| Pipe 执行 | stdin/stdout/stderr 管道 | 长时间运行、增量输出 |

WorkBuddySandbox 的 `runExecutable` 同时支持 `sync`、`pty`、`pipe`，本质就是这几种模式的工程化封装。

## 3. 方案对比

| 维度 | Shell 执行 | 直接 exec | PTY/Pipe |
|------|------------|-----------|----------|
| 易用性 | 高 | 中 | 中 |
| 参数安全 | 低，容易转义错误 | 高 | 高 |
| 交互能力 | 弱 | 弱 | 强 |
| 审计清晰度 | 命令字符串更直观 | 参数结构更清楚 | 可记录每段输出 |

结论：底层实现应优先使用“可执行文件 + 参数数组”，把 `displayCommand` 只作为展示和审计字段。

## 4. 如何写一个最小版本

一个最小进程启动器至少要控制五件事：

1. `executable`：启动哪个程序；
2. `args`：传什么参数；
3. `cwd`：在哪个目录运行；
4. `env`：注入哪些环境变量；
5. `timeout`：超时后如何杀掉。

Rust 最小模型：

```rust
use std::process::{Command, Stdio};
use std::time::Duration;

let mut child = Command::new("/bin/bash")
    .arg("-c")
    .arg("echo hello && pwd")
    .current_dir("/tmp")
    .env("WB_SESSION_ID", "sess-1")
    .stdout(Stdio::piped())
    .stderr(Stdio::piped())
    .spawn()?;

// 真实实现还要加超时等待、kill、stdout/stderr 收集
let output = child.wait_with_output()?;
println!("exit={:?}", output.status.code());
```

这段代码不是沙盒，但它是沙盒执行的地基。

## 5. 常见坑

- **把命令拼成字符串**：`"git " + user_input` 会引入转义和注入问题，应优先使用参数数组。
- **只读 stdout 不读 stderr**：子进程可能因管道缓冲区写满而卡死。
- **只 kill 父进程**：子进程可能继续运行，真实实现要考虑进程组或 JobObject。
- **忽略 cwd**：相同命令在不同目录下结果完全不同。
- **忽略 PATH**：GUI App 启动的环境变量经常和终端不同。

## 6. 回到 WorkBuddySandbox

| 机制 | 项目位置 |
|------|----------|
| `runExecutable` 请求处理 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/sandbox/handlers/run_executable.rs` |
| sync/pty/pipe 类型 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/executor/types.rs` |
| 原生执行兜底 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/executor/native_executor.rs` |
| macOS 进程启动细节 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/platform/macos/process.rs` |

理解这些文件时，不要先看全部代码。先问自己：它在设置 `cwd`、`env`、`stdio`、`timeout`、`kill` 中的哪一项？

## 7. 练习题：你要亲手实现什么

1. 写一个 `mini-runner`：输入 JSON `{executable,args,cwd,env,timeoutMs}`，执行后输出 `{exitCode,stdout,stderr}`。
2. 加入超时：超过 3 秒自动终止进程。
3. 加入 `displayCommand`：展示给用户，但实际执行仍使用 `executable + args`。
4. 故意运行一个会不断输出 stderr 的命令，验证不会卡死。

完成后再读 WorkBuddySandbox 的 `runExecutable`，你会发现它只是把这些基础能力扩展成跨平台沙盒版本。

## 8. 延伸阅读

- [macOS sandbox-exec 最小实现](./macOS-sandbox-exec最小实现.md)
- [NDJSON 协议从零实现](../03-IPC协议与服务进程/NDJSON协议从零实现.md)
