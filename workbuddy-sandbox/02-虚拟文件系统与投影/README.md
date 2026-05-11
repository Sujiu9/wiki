# 🟡 虚拟文件系统与投影

> 📋 前置知识：[从 fork/exec 到进程隔离](../01-进程与沙盒/从fork-exec到进程隔离.md)

本模块学习 WorkBuddySandbox 最容易“看起来很玄”的部分：文件投影。

## 为什么需要文件投影

Agent 需要修改项目文件，但用户希望修改可审计、可丢弃、可选择性落盘。如果直接让 Agent 写源目录，就很难做到：

- 查看这次会话改了哪些文件；
- 丢弃某些改动；
- 恢复删除；
- 多个会话互不影响；
- 对用户展示“像真实目录一样”的工作区。

所以核心思路是：**源目录只读，写入覆盖层，最终再选择同步回源目录**。

## 学习顺序

1. [覆盖层文件系统从零实现](./覆盖层文件系统从零实现.md)：先用普通目录实现 `source + overlay + deletion marker`。
2. [Windows ProjFS 最小 Provider](./Windows-ProjFS最小Provider.md)：理解 Windows 系统投影回调模型。
3. [macOS FileProvider 最小 Provider](./macOS-FileProvider最小Provider.md)：理解 macOS domain、identifier、materialized。

## 方案对比

| 方案 | 优点 | 缺点 |
|------|------|------|
| 直接复制目录 | 简单 | 大仓库慢，占空间 |
| Git worktree/branch | Git 语义强 | 非 Git 目录支持差 |
| 自研 overlay | 能精确控制变更 | 需要自己处理删除、同步、冲突 |
| 系统投影 FS | 用户体验像真实目录 | 平台差异大，调试复杂 |

WorkBuddySandbox 结合了 overlay 和平台投影：底层变更层由 Rust 管理，Windows 用 ProjFS，macOS 用 FileProvider 展示。

## WorkBuddySandbox 映射

| 能力 | 项目路径 |
|------|----------|
| 文件能力入口 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/file/file_manager.rs` |
| 投影抽象 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/file/projection.rs` |
| overlay 核心 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/projfs/src/overlay.rs` |
| Windows Provider | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/projfs/src/projfs_provider.rs` |
| macOS FileProvider | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/FileExtension/Sources/FileProviderExtension.swift` |
| FileProvider 枚举 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/FileExtension/Sources/FileProviderEnumerator.swift` |

## 学完本模块你应该能写什么

- 一个普通目录版 overlay：读 source，写 overlay，用 deletion marker 表示删除。
- 一个 `list_changes`：列出新增、修改、删除。
- 一个 `sync`：把 overlay 变更应用回 source。
- 一个 `discard`：丢弃 overlay 变更。
- 能解释 ProjFS / FileProvider 只是把这个模型投影给用户看的平台外壳。

## 延伸阅读

- [snapfile 快照系统](../05-快照与变更管理/从文件拷贝到Git语义快照.md)
- [macOS 代码签名与 entitlements 实战](../07-调试与系统权限/macOS代码签名与entitlements实战.md)
