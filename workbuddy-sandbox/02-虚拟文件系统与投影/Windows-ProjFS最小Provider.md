# 🟡 Windows ProjFS 最小 Provider

> 📋 前置知识：[覆盖层文件系统从零实现](./覆盖层文件系统从零实现.md)

## 1. 为什么需要 ProjFS

Overlay 模型可以管理变更，但用户还需要在文件管理器和命令行里“看到一个真实目录”。如果你把所有文件都复制出来，成本很高。

`ProjFS（Projected File System）` 的价值是：

- 先展示目录树和占位文件；
- 真正读取文件内容时，再由 Provider 按需供给；
- 文件变化由 Provider 路由到 source/overlay；
- 用户体验像普通 NTFS 目录。

## 2. 解决方案概览

| 方案 | 简述 | 适用场景 |
|------|------|----------|
| 真实复制目录 | 所有文件提前落地 | 小目录 |
| Symlink/Junction | 指向原目录 | 不能表达 overlay 删除/修改 |
| ProjFS Provider | 系统回调按需供给 | 大目录、虚拟工作区 |

WorkBuddySandbox Windows 侧用 ProjFS 展示投影根，再通过 session 路由找到对应 source 和 overlay。

## 3. 方案对比

| 维度 | 复制目录 | Symlink | ProjFS |
|------|----------|---------|--------|
| 启动速度 | 慢 | 快 | 快 |
| 表达删除 marker | 可 | 弱 | 可由 Provider 控制 |
| 大仓库性能 | 差 | 好 | 好 |
| 实现复杂度 | 低 | 低 | 高 |

ProjFS 的复杂度来自回调模型：系统问你“这个目录有什么”“这个文件内容是什么”，你必须及时回答。

## 4. 如何写一个最小 Provider

概念模型：

```text
启动 Provider(root)
  注册虚拟根
  回调 GetDirectoryEnumeration(path): 返回子项列表
  回调 GetPlaceholderInfo(path): 返回文件大小/属性
  回调 GetFileData(path, offset, length): 返回文件内容
```

Provider 不应该直接只读 source，它要先合并 overlay：

```text
enumerate(path):
  items = list(source/path)
  items += list(overlay/modified/path)
  items -= deletions_under(path)
  return merged(items)
```

## 5. 常见坑

- **回调必须快**：文件管理器枚举目录时会触发大量回调。
- **路径路由**：虚拟路径要映射到 session，再映射到 source/overlay。
- **大小写/规范化**：Windows 路径格式很多，要统一处理。
- **删除和重命名**：不能只处理读文件，写操作会影响 overlay 状态。
- **Provider 生命周期**：虚拟根注册、停止、清理要和 session 生命周期一致。

## 6. 回到 WorkBuddySandbox

| 机制 | 项目位置 |
|------|----------|
| Windows provider 入口 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/file/provider_windows.rs` |
| 全局投影管理 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/projfs/src/projection_manager.rs` |
| ProjFS 回调实现 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/projfs/src/projfs_provider.rs` |
| session 路由 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/projfs/src/projfs_router.rs` |
| overlay 合并 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/projfs/src/overlay.rs` |

理解顺序建议：先读 `overlay.rs`，再读 `projfs_router.rs`，最后读 `projfs_provider.rs`。不要一开始陷入 Windows API 细节。

## 7. 练习题：你要亲手实现什么

1. 不接 ProjFS，先写一个 `enumerate_virtual(path)`，合并 source 和 overlay。
2. 写一个内存版 Provider：给定虚拟路径，返回文件大小和内容。
3. 再阅读 ProjFS 回调，标出每个回调对应你内存版 Provider 的哪个函数。
4. 设计一个测试：source 有 `a.txt`，overlay 修改 `a.txt`，删除 `b.txt`，枚举结果应该是什么？

## 8. 延伸阅读

- [macOS FileProvider 最小 Provider](./macOS-FileProvider最小Provider.md)
- [从文件拷贝到 Git 语义快照](../05-快照与变更管理/从文件拷贝到Git语义快照.md)
