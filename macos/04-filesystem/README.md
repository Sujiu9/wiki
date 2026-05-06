# macOS 文件系统

## 概览

macOS 使用 APFS（Apple File System）作为默认文件系统，具备快照、克隆、加密、空间共享等现代特性。文件投影技术（FileProvider）则允许应用创建虚拟文件系统视图。

## 核心机制

| 机制 | 用途 | 说明 |
|------|------|------|
| APFS | 默认文件系统 | 支持快照、克隆、卷加密 |
| FileProvider Extension | 文件投影/虚拟文件系统 | 按需加载、云存储同步 |
| FSEvents | 文件系统事件监听 | 实时监控目录变更 |
| Extended Attributes | 扩展属性 (xattr) | 存储文件元数据 |
| Sandbox Containers | 应用数据隔离 | 每个应用独立数据目录 |

## 目录

- [FileProvider 文件投影](./FileProvider文件投影.md) 🟡
- [APFS 文件系统特性](./APFS文件系统特性.md) 🟢
- [文件权限与 ACL](./文件权限与ACL.md) 🟢
- [FSEvents 文件监听](./FSEvents文件监听.md) 🟡
