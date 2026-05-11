# 🟢 调试与系统权限

> 📋 前置知识：已阅读任一平台能力文档。

底层开发最容易卡住的不是代码逻辑，而是权限、签名、系统缓存、Extension 生命周期和日志分散。

## 为什么单独学习调试和权限

WorkBuddySandbox 涉及：

- macOS App Group；
- FileProvider Extension；
- NetworkExtension；
- Hardened Runtime；
- 动态库签名；
- Rust xlog；
- 系统 deny 日志；
- Windows Hook 审计日志。

如果不建立排障模型，你会看到“没有效果”，但不知道是代码没跑、权限不够、Extension 没启动，还是日志看错地方。

## 学习顺序

1. [macOS 代码签名与 entitlements 实战](./macOS代码签名与entitlements实战.md)：先理解权限为什么必须声明和签名。
2. [底层日志与排障方法](./底层日志与排障方法.md)：再建立从现象到证据的排查路径。

## 方案对比

| 问题 | 错误学习方式 | 正确学习方式 |
|------|--------------|--------------|
| 权限失败 | 猜代码 bug | 查 entitlement、签名、系统日志 |
| Extension 不工作 | 重启 App | 查 domain、App Group、Extension 日志 |
| 沙盒拒绝 | 只看 stderr | 查 deny 日志和 `FileBlockRecord` |
| 网络不通 | 只看 curl | 区分 LocalProxy、NE、tsbx 来源 |

## WorkBuddySandbox 映射

| 能力 | 项目路径 |
|------|----------|
| Rust 日志 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/logging/src/lib.rs` |
| xlog C++ 引擎 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/logging/xlog-src/xlog_bridge.cpp` |
| Swift 日志 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/SandboxExt/Sources/Shared/XLog.swift` |
| 权限管理 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/permission/manager.rs` |
| 平台规则生成 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/permission/platform_rules.rs` |
| macOS 构建签名 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/scripts/build-mac.sh` |

## 学完本模块你应该能写什么

- 一个检查 entitlements 和 codesign 的排障清单。
- 一个最小日志闭环：初始化、写日志、flush、定位文件。
- 一个问题定位流程：先确认进程是否启动，再确认权限，再确认规则，再确认日志。

## 延伸阅读

- [macOS FileProvider 最小 Provider](../02-虚拟文件系统与投影/macOS-FileProvider最小Provider.md)
- [macOS NetworkExtension 理解路径](../06-网络拦截/macOS-NetworkExtension理解路径.md)
