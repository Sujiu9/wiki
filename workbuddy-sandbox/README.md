# 🟢 WorkBuddySandbox 底层能力学习专区

> 面向正在开发 `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox` 的开发者。目标不是“看懂别人写的代码”，而是逐步具备自己写出最小版本的能力。

## 这个专区解决什么问题

WorkBuddySandbox 涉及沙盒、文件投影、IPC、FFI、快照、网络拦截、代码签名和系统日志。直接读项目源码很容易变成“翻译变量名”：知道某个文件做了什么，但不知道为什么必须这么做，更不知道离开项目后怎么自己写。

本专区按能力训练组织，每个主题都遵循：

```text
为什么需要 -> 有哪些方案 -> 方案优劣 -> 如何手写最小版本 -> 回到 WorkBuddySandbox -> 练习验证
```

## 推荐阅读顺序

1. [00-学习路线](./00-学习路线.md)：先建立完整地图。
2. [进程与沙盒](./01-进程与沙盒/README.md)：从最小进程启动器开始。
3. [IPC 协议与服务进程](./03-IPC协议与服务进程/README.md)：学会写 `sandbox-cli` 这种服务进程模型。
4. [虚拟文件系统与投影](./02-虚拟文件系统与投影/README.md)：理解 source/overlay/deletion marker。
5. [Rust 跨语言边界](./04-Rust跨语言边界/README.md)：理解 `cdylib`、C ABI、动态加载。
6. [快照与变更管理](./05-快照与变更管理/README.md)：从复制快照进阶到 Git 语义快照。
7. [网络拦截](./06-网络拦截/README.md)：从本地代理理解到 NetworkExtension。
8. [调试与系统权限](./07-调试与系统权限/README.md)：建立权限、签名、日志证据链。

## 学完后你应该能写什么

| 能力 | 验证方式 |
|------|----------|
| 最小进程启动器 | 输入 JSON，执行命令，输出 exitCode/stdout/stderr |
| 最小 NDJSON IPC | 支持 Ping、messageId、异步通知 |
| 最小 overlay | 支持 read/write/delete/list_changes/sync/discard |
| 最小 Rust FFI | 导出 `extern "C"` 函数，宿主动态加载调用 |
| 最小快照系统 | 支持 commit/log/checkout/diff |
| 最小 CONNECT 代理 | 解析 host:port，按规则 allow/deny |
| macOS 权限排障 | 能检查 codesign、entitlements、App Group、日志路径 |

## 和 WorkBuddySandbox 的关系

这里不是项目源码注释。每篇文档会先让你脱离项目写一个最小模型，再映射回真实源码路径。这样读源码时你能区分：

- 哪些是底层原理；
- 哪些是平台差异；
- 哪些是工程化边界处理；
- 哪些只是 WorkBuddySandbox 的项目约定。

## 文档规范

- 每个主题一个文件。
- 单文件控制在 300 行以内。
- 代码示例只展示关键逻辑。
- 专有名词首次出现给出中英文对照。
- 每篇文档必须有“练习题：你要亲手实现什么”。
