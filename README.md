# OS 开发学习仓库

> 面向 macOS 和 Windows 平台的操作系统底层开发学习笔记与实践代码。

## 📁 仓库结构

```
.
├── macos/                  # macOS 平台相关
│   ├── 01-fundamentals/    # 基础概念（内核、进程、内存等）
│   ├── 02-configuration/   # 配置文件（plist、launchd、defaults）
│   ├── 03-filesystem/      # 文件系统（APFS、权限、扩展属性）
│   ├── 04-ipc/             # 进程间通信（XPC、Mach ports、Unix sockets）
│   ├── 05-security/        # 安全机制（SIP、Gatekeeper、Keychain、Notarization）
│   ├── 06-networking/      # 网络（Network.framework、socket）
│   └── 07-system-services/ # 系统服务（launchd、IOKit、Core Foundation）
│
├── windows/                # Windows 平台相关
│   ├── 01-fundamentals/    # 基础概念（NT 内核、进程、内存等）
│   ├── 02-configuration/   # 配置文件（注册表、INI、组策略）
│   ├── 03-filesystem/      # 文件系统（NTFS、ACL、符号链接）
│   ├── 04-ipc/             # 进程间通信（Named Pipes、COM、WCF）
│   ├── 05-security/        # 安全机制（UAC、DPAPI、Windows Defender）
│   ├── 06-networking/      # 网络（Winsock、WinHTTP、WinINet）
│   └── 07-system-services/ # 系统服务（SCM、WMI、ETW）
│
├── cross-platform/         # 跨平台对比与通用知识
│   ├── comparison/         # macOS vs Windows 对比文档
│   └── patterns/           # 跨平台开发模式与最佳实践
│
└── examples/               # 实践代码示例
    ├── macos/
    └── windows/
```

## 🎯 学习路径建议

1. **先看 fundamentals** — 理解各平台的内核架构和核心概念
2. **对比学习** — 通过 `cross-platform/comparison/` 理解两平台的异同
3. **深入专题** — 根据实际开发需要，深入 IPC、安全、文件系统等模块
4. **动手实践** — 在 `examples/` 中运行和修改示例代码

## 📝 文档规范

- 每个主题一个 Markdown 文件，包含：概念说明 → 原理图解 → 代码示例 → 常见问题
- 代码示例尽量可独立运行
- 专有名词首次出现时给出中英文对照和简要解释

## 🏷️ 标签说明

| 标签 | 含义 |
|------|------|
| 🟢 基础 | 入门级，建议优先阅读 |
| 🟡 进阶 | 需要一定基础知识 |
| 🔴 深入 | 涉及底层实现细节 |
