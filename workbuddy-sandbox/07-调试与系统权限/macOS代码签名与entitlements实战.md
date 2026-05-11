# 🟢 macOS 代码签名与 entitlements 实战

> 📋 前置知识：理解 App、Extension、动态库的基本概念。

## 1. 为什么需要代码签名和 entitlements

macOS 的很多底层能力不是“调用 API 就能用”。系统会检查：

- 这个二进制是谁签名的；
- 它声明了哪些权限；
- App 和 Extension 是否属于同一个 Team；
- 是否开启 Hardened Runtime；
- 动态库签名是否可信。

`entitlements` 是权限声明，`codesign` 是把声明绑定到二进制的过程。

## 2. 解决方案概览

| 机制 | 解决什么 |
|------|----------|
| Code Signing | 证明二进制身份和完整性 |
| Entitlements | 声明系统能力，如 App Group、NetworkExtension |
| App Group | App 与 Extension 共享容器 |
| Hardened Runtime | 限制运行时动态行为，提高安全性 |
| TCC | 用户隐私权限，如 Full Disk Access |

WorkBuddySandbox 的 macOS 侧同时涉及 FileProvider、NetworkExtension、App Group 和动态库签名，所以签名不是发布阶段才关心，而是开发阶段就必须理解。

## 3. 方案对比

| 场景 | ad-hoc 签名 | Developer ID/Team 签名 |
|------|-------------|------------------------|
| 本地简单测试 | 可用 | 可用 |
| NetworkExtension | 通常不够 | 需要正确 Team/entitlement |
| 分发 | 不适合 | 必需 |
| App/Extension 协作 | 容易不一致 | 可统一 Team |

## 4. 如何验证最小闭环

查看签名：

```bash
codesign -dv --verbose=4 /path/to/App.app
```

查看 entitlements：

```bash
codesign -d --entitlements :- /path/to/App.app
```

验证动态库：

```bash
codesign --verify --deep --strict /path/to/libsandbox_ffi.dylib
```

你要确认三件事：

1. 主 App、FileProvider、NetworkExtension 的 Team ID 一致；
2. App Group 名称一致；
3. 需要的 NetworkExtension / FileProvider 权限确实在 entitlements 中。

## 5. 常见坑

- **cargo build 覆盖签名产物**：重新构建动态库后签名可能失效。
- **App Group 拼写不一致**：主 App 和 Extension 看似都能运行，但共享数据为空。
- **Extension entitlements 缺失**：Extension 启动失败或能力不可用。
- **Hardened Runtime 下加载未签名 dylib**：`dlopen` 失败。
- **TCC 权限误判为代码 bug**：全盘投影或敏感目录读取需要用户授权。

## 6. 回到 WorkBuddySandbox

| 机制 | 项目位置 |
|------|----------|
| 主 App entitlements | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/SandboxMain/SandboxMain.entitlements` |
| FileProvider entitlements | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/FileExtension/SandboxFileProvider.entitlements` |
| NetworkExtension entitlements | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/networkextension/SandboxNetworkExtension.entitlements` |
| 构建签名脚本 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/scripts/build-mac.sh` |
| 代码签名规则 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/.codebuddy/rules/mac-codesigning.md` |
| Full Disk Access 检查 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/SandboxMain/Sources/SandboxMain/DiskAccessChecker.swift` |

## 7. 练习题：你要亲手实现什么

1. 对一个 `.app` 执行 `codesign -d --entitlements :-`，找出 App Group。
2. 对一个 `.dylib` 做签名校验。
3. 故意改错 App Group，推演 FileProvider/NE 会出现什么现象。
4. 写一份排障清单：`签名 -> entitlements -> App Group -> 日志`。

## 8. 延伸阅读

- [底层日志与排障方法](./底层日志与排障方法.md)
- [macOS NetworkExtension 理解路径](../06-网络拦截/macOS-NetworkExtension理解路径.md)
