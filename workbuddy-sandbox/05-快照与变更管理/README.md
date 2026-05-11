# 🟡 快照与变更管理

> 📋 前置知识：[覆盖层文件系统从零实现](../02-虚拟文件系统与投影/覆盖层文件系统从零实现.md)

本模块学习“如何让文件变更可回溯”。

## 为什么需要快照

Overlay 只能表达当前会话的改动，但开发工具还需要：

- 给某个时刻打点；
- 查看历史版本；
- 回滚某个文件；
- 对比两个版本；
- 清理不再需要的历史。

这就是 `snapfile` 的问题域。

## 学习顺序

1. [从文件拷贝到 Git 语义快照](./从文件拷贝到Git语义快照.md)：先理解快照系统的最小抽象。
2. [libgit2 最小快照系统](./libgit2最小快照系统.md)：再理解 Git 对象模型如何高效表达版本。

## 方案对比

| 方案 | 优点 | 缺点 |
|------|------|------|
| 完整复制 | 最容易实现 | 空间浪费 |
| 增量 diff | 空间较省 | 应用补丁复杂 |
| Git 对象库 | 去重、历史、diff 成熟 | 学习成本高 |

## WorkBuddySandbox 映射

| 能力 | 项目路径 |
|------|----------|
| SnapFile trait | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/snapfile/src/snapfile.rs` |
| 简单复制实现 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/snapfile/src/simple_copy.rs` |
| Git/libgit2 实现 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/snapfile/src/git_snapfile.rs` |
| sandbox 快照管理 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/file/snapshot_manager.rs` |

## 学完本模块你应该能写什么

- 一个单文件快照接口：`commit/log/checkout/revert/reset/gc`。
- 一个复制版快照后端。
- 一个理解 Git `blob/tree/commit/ref` 的最小快照模型。
- 能解释 WorkBuddySandbox 为什么同时有 simple copy 和 libgit2 后端。

## 延伸阅读

- [覆盖层文件系统从零实现](../02-虚拟文件系统与投影/覆盖层文件系统从零实现.md)
- [底层日志与排障方法](../07-调试与系统权限/底层日志与排障方法.md)
