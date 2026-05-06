# macOS 网络机制

## 概览

macOS 的网络栈基于 BSD 套接字层，上层通过 Network.framework（现代）和 URLSession 提供高级 API。NetworkExtension 框架允许应用深度介入网络流量的处理。

## 核心机制

| 机制 | 用途 | 说明 |
|------|------|------|
| NetworkExtension | 网络扩展框架 | VPN、内容过滤、透明代理 |
| Network.framework | 现代网络 API | 替代 BSD socket 的高层接口 |
| URLSession | HTTP/HTTPS 客户端 | 系统级网络请求 |
| mDNSResponder | DNS 解析 | 系统统一 DNS 服务 |
| pf (Packet Filter) | 防火墙 | BSD 包过滤器 |

## 目录

- [NetworkExtension 网络拦截](./NetworkExtension网络拦截.md) 🟡
- [Network.framework 基础](./Network框架基础.md) 🟢
- [DNS 解析机制](./DNS解析机制.md) 🟡
- [BSD Socket 编程](./BSD-Socket编程.md) 🟢
