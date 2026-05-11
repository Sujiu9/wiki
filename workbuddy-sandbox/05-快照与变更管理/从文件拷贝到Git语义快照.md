# 🟡 从文件拷贝到 Git 语义快照

> 📋 前置知识：[覆盖层文件系统从零实现](../02-虚拟文件系统与投影/覆盖层文件系统从零实现.md)

## 1. 为什么需要快照

Overlay 解决“当前改了什么”，快照解决“历史上发生过什么”。

一个最小快照系统应该回答：

- 当前文件提交成哪个版本？
- 历史版本有哪些？
- 能不能读某个历史版本？
- 能不能把当前文件恢复到某个版本？
- 旧版本如何清理？

## 2. 解决方案概览

| 方案 | 存储方式 | 适用场景 |
|------|----------|----------|
| 版本号复制 | `v1/a.txt`, `v2/a.txt` | 入门、文件少 |
| 内容哈希 | 按文件内容 hash 去重 | 重复内容多 |
| Git 对象库 | blob/tree/commit/ref | 需要 diff、历史语义 |

WorkBuddySandbox 的 `snapfile` 抽象允许不同后端：简单复制易理解，libgit2 后端更接近真实工程。

## 3. 方案对比

| 维度 | 复制快照 | Git 语义快照 |
|------|----------|--------------|
| 实现难度 | 低 | 中高 |
| 空间效率 | 差 | 好 |
| diff 能力 | 需要自写 | 现成 |
| 历史管理 | 自写 | refs/commits |

建议先手写复制版，因为它能逼你明确接口；再学 Git 后端只是替换存储实现。

## 4. 如何写一个最小版本

最小接口：

```rust
trait SnapFile {
    fn commit(&self, path: &str, message: &str) -> VersionId;
    fn log(&self, path: &str) -> Vec<VersionId>;
    fn read_version(&self, path: &str, version: VersionId) -> Vec<u8>;
    fn checkout(&self, path: &str, version: VersionId);
}
```

复制版目录结构：

```text
.snap/
└── a.txt/
    ├── v1.data
    ├── v2.data
    └── meta.json
```

`commit` 的最小逻辑：

```python
def commit(path):
    n = next_version(path)
    copy_file(workdir/path, snap/path/f"v{n}.data")
    append_meta(path, version=n, time=now())
    return n
```

## 5. 常见坑

- **只记录内容不记录元数据**：mtime、权限、删除状态可能也重要。
- **checkout 覆盖当前改动**：恢复前要确认或先备份。
- **删除文件的快照**：删除本身也是一种版本状态。
- **并发 commit**：版本号分配要原子。
- **gc 误删**：还被引用的版本不能清理。

## 6. 回到 WorkBuddySandbox

| 机制 | 项目位置 |
|------|----------|
| 快照 trait | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/snapfile/src/snapfile.rs` |
| 复制后端 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/snapfile/src/simple_copy.rs` |
| 快照类型 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/snapfile/src/types.rs` |
| 文件层快照管理 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/file/snapshot_manager.rs` |

读 `simple_copy.rs` 时不要急着看所有错误处理，先找 `commit` 如何保存版本、`checkout` 如何恢复版本。

## 7. 练习题：你要亲手实现什么

1. 写一个复制版单文件快照系统。
2. 支持 `log` 输出所有版本号和提交说明。
3. 支持 `checkout(vN)` 恢复内容。
4. 支持 `diff(v1, v2)`，哪怕只是逐行比较。
5. 设计删除文件的快照表达方式。

## 8. 延伸阅读

- [libgit2 最小快照系统](./libgit2最小快照系统.md)
- [覆盖层文件系统从零实现](../02-虚拟文件系统与投影/覆盖层文件系统从零实现.md)
