# 🔴 macOS NetworkExtension 理解路径

> 📋 前置知识：[代理与透明代理基础](./代理与透明代理基础.md)、[macOS 代码签名与 entitlements 实战](../07-调试与系统权限/macOS代码签名与entitlements实战.md)

## 1. 为什么需要 NetworkExtension

LocalProxy 只能拦截“愿意走代理”的程序。Agent 执行的命令可能直接创建 TCP 连接，完全绕过 `HTTP_PROXY`。

`NetworkExtension（NE）` 的价值是把网络拦截下沉到系统层：

- 程序不一定知道自己被代理；
- 可以按 PID/session 关联流量；
- 能记录被拒绝的 host/port；
- 适合作为 macOS 的透明网络拦截方案。

## 2. 解决方案概览

| 方案 | 简述 | 适用场景 |
|------|------|----------|
| LocalProxy | 通过环境变量配置代理 | 兜底、简单场景 |
| NE App Proxy | 系统网络扩展处理流量 | 透明代理、规则拦截 |
| PF 防火墙 | 包过滤 | 低层网络控制，不适合 App 分发 |

WorkBuddySandbox 使用 `NETransparentProxyProvider`，宿主负责安装/启动配置，Rust 侧通过 App Group 与 NE 交换规则和拦截记录。

## 3. 方案对比

| 维度 | LocalProxy | NetworkExtension |
|------|------------|------------------|
| 覆盖范围 | 中 | 高 |
| 权限要求 | 低 | 高，需要 entitlement |
| 调试难度 | 低 | 高 |
| 生命周期 | 普通进程 | 系统管理 Extension |

所以 NE 是能力更强但更难调试的方案，文档学习时应先理解代理，再理解 NE。

## 4. 如何建立最小理解模型

NE 不是普通函数调用，而是三方协作：

```text
主 App
  保存/启动 NE 配置
  写入 App Group 共享规则

NetworkExtension
  被系统启动
  读取 App Group 规则
  接收流量并做 allow/deny
  写入 blocked records

Rust sandbox-cli
  注册当前沙盒 PID/session
  查询 blocked records
  同步 session 网络规则
```

最小数据流：

```text
runExecutable -> 注册 PID -> Agent 发起连接 -> NE 匹配 PID/session -> 判断规则 -> 放行或记录 block
```

## 5. 常见坑

- **entitlement 不匹配**：主 App 和 Extension 必须有正确 NetworkExtension 权限。
- **App Group 不一致**：规则和记录无法共享。
- **Extension 启动慢**：Rust 侧要能查询 NE 是否 ready，并决定是否降级 LocalProxy。
- **PID 归因困难**：子进程链、短生命周期进程都可能影响归因。
- **日志分散**：主 App、NE、Rust 日志在不同位置。

## 6. 回到 WorkBuddySandbox

| 机制 | 项目位置 |
|------|----------|
| NE Provider | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/networkextension/Sources/TransparentProxyProvider.swift` |
| NE 管理器 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/SandboxExt/Sources/Manager/NetworkExtensionManager.swift` |
| 共享模型 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/SandboxExt/Sources/Shared/NESharedModels.swift` |
| Rust NE Bridge | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/platform/macos/ne_bridge.rs` |
| Objective-C Bridge | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/platform/macos/ne_bridge.m` |
| NE ready 查询 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/network/macos.rs` |

读这些文件时先画出“谁写 App Group、谁读 App Group、谁发 Darwin notification”，不要一开始追所有 Swift 细节。

## 7. 练习题：你要亲手实现什么

1. 不写 NE，先设计一个 App Group JSON：保存 session 网络规则。
2. 写两个进程：一个写规则，一个读规则并判断 host 是否允许。
3. 模拟 NE blocked record：写入 `{pid,host,port,decision}`。
4. Rust/脚本侧读取记录并转换为 `NetworkBlockRecord`。
5. 最后再回项目看 `TransparentProxyProvider.swift` 如何把系统流量接进这套模型。

## 8. 延伸阅读

- [底层日志与排障方法](../07-调试与系统权限/底层日志与排障方法.md)
- [macOS sandbox-exec 最小实现](../01-进程与沙盒/macOS-sandbox-exec最小实现.md)
