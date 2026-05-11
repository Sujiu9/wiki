# 🟡 libgit2 最小快照系统

> 📋 前置知识：[从文件拷贝到 Git 语义快照](./从文件拷贝到Git语义快照.md)

## 1. 为什么用 libgit2

复制版快照容易写，但会浪费空间，也缺少成熟 diff 和对象管理能力。Git 的底层模型天然适合快照：

- `blob` 存文件内容；
- `tree` 存目录结构；
- `commit` 指向某个 tree 和父 commit；
- `ref` 给某条历史起名字。

`libgit2` 是 Git 核心能力的库形式，Rust 中通常通过 `git2` crate 调用。

## 2. 解决方案概览

| 方案 | 简述 | 适用场景 |
|------|------|----------|
| 调用 `git` 命令 | 简单，依赖外部命令 | 脚本/工具 |
| 使用 `libgit2` | 内嵌库，无需 shell | 应用内快照 |
| 自写对象库 | 可控 | 成本极高 |

WorkBuddySandbox 使用 libgit2，是为了在 Rust 进程内管理快照，而不是依赖外部 git 命令。

## 3. 方案对比

| 维度 | git 命令 | libgit2 |
|------|---------|---------|
| 集成难度 | 低 | 中 |
| 错误处理 | 解析 stdout/stderr | 结构化错误 |
| 分发依赖 | 依赖 git 可执行 | 依赖动态库/构建 |
| 性能控制 | 较弱 | 较强 |

## 4. 如何写一个最小版本

Git 语义快照的最小流程：

```text
open repository
write blob(file content)
create tree(blob path)
create commit(tree, parent)
update ref refs/snapfile/a.txt/v1 -> commit
```

伪代码：

```rust
let repo = git2::Repository::open_bare(".snap.git")?;
let blob_id = repo.blob(file_bytes)?;

let mut builder = repo.treebuilder(None)?;
builder.insert("a.txt", blob_id, 0o100644)?;
let tree_id = builder.write()?;
let tree = repo.find_tree(tree_id)?;

let sig = repo.signature()?;
let commit_id = repo.commit(
    Some("refs/snapfile/a.txt/v1"),
    &sig,
    &sig,
    "snapshot v1",
    &tree,
    &[],
)?;
```

真实实现会处理父 commit、版本递增、diff、checkout、gc。

## 5. 常见坑

- **bare repo 概念**：没有工作区，所有内容都在对象库里。
- **ref 命名**：需要设计稳定命名，避免不同文件冲突。
- **路径编码**：不同平台路径分隔符要统一。
- **libgit2 分发**：macOS/Windows 都要处理动态库构建和签名。
- **checkout 安全**：恢复版本时要避免覆盖用户未保存变更。

## 6. 回到 WorkBuddySandbox

| 机制 | 项目位置 |
|------|----------|
| git feature 配置 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/snapfile/Cargo.toml` |
| libgit2 后端 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/snapfile/src/git_snapfile.rs` |
| 构建 libgit2 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/scripts/build_libgit2.sh` |
| 测试 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/snapfile/tests/git_snapfile.rs` |

`git_snapfile.rs` 使用 `refs/snapfile/<relative_path>/vN` 管理单文件版本。理解这个 ref 命名，就能理解它如何把 Git 历史映射到文件快照。

## 7. 练习题：你要亲手实现什么

1. 用 `git2` 创建一个 bare repo。
2. 把字符串 `hello` 写成 blob，并创建 commit。
3. 用 ref `refs/snapfile/demo.txt/v1` 指向 commit。
4. 再提交 v2，读取 v1/v2 的 blob 内容。
5. 实现一个最小 `diff(v1, v2)`。

## 8. 延伸阅读

- [底层日志与排障方法](../07-调试与系统权限/底层日志与排障方法.md)
- [Rust cdylib 与 C ABI 最小示例](../04-Rust跨语言边界/Rust-cdylib与C-ABI最小示例.md)
