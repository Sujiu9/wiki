# 🟡 网络拦截

> 📋 前置知识：[从 fork/exec 到进程隔离](../01-进程与沙盒/从fork-exec到进程隔离.md)

本模块学习 Agent 网络访问如何被识别、放行、拒绝和审计。

## 为什么需要网络拦截

AI Agent 执行命令时可能访问网络：下载依赖、调用 API、连接未知主机。沙盒要回答：

- 访问了哪个 host/port？
- 是 DNS、TCP 连接还是 HTTP CONNECT？
- 当前规则允许还是拒绝？
- 被拒绝后如何记录并让用户选择？

## 学习顺序

1. [代理与透明代理基础](./代理与透明代理基础.md)：先理解显式代理、CONNECT、透明代理和降级。
2. [macOS NetworkExtension 理解路径](./macOS-NetworkExtension理解路径.md)：再理解 macOS 为什么需要系统扩展参与。

## 方案对比

| 方案 | 优点 | 缺点 |
|------|------|------|
| 环境变量代理 | 简单 | 只对尊重代理的程序有效 |
| 本地 HTTP Proxy | 可记录 HTTP/CONNECT | 对原生 TCP 覆盖不足 |
| Windows API Hook | 能拦 connect/send 等 | 实现复杂 |
| macOS NetworkExtension | 系统级透明代理 | 权限、签名、生命周期复杂 |

## WorkBuddySandbox 映射

| 能力 | 项目路径 |
|------|----------|
| 网络模块入口 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/network/mod.rs` |
| LocalProxy | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/network/proxy.rs` |
| 网络访问控制 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/network/access_control.rs` |
| macOS NE 查询 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/network/macos.rs` |
| NE Bridge | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/platform/macos/ne_bridge.rs` |
| NE Provider | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/networkextension/Sources/TransparentProxyProvider.swift` |

## 学完本模块你应该能写什么

- 一个只处理 HTTP CONNECT 的本地代理。
- 一个 host/port allow/deny 规则判断器。
- 一个网络拦截记录结构：`{host, port, scheme, action, source}`。
- 能解释 NE 已就绪时为什么跳过 LocalProxy，未就绪时为什么降级。

## 延伸阅读

- [macOS 代码签名与 entitlements 实战](../07-调试与系统权限/macOS代码签名与entitlements实战.md)
- [Windows DLL 注入与 API Hook 最小实现](../01-进程与沙盒/Windows-DLL注入与API-Hook最小实现.md)
