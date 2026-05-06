# 🟡 macOS NetworkExtension 网络拦截机制

> 📋 前置知识：[macOS 基础概念](../01-fundamentals/README.md)、[进程沙盒机制](../03-security/进程沙盒机制.md)

## 为什么需要网络拦截？

AI Agent 在沙盒中执行任务时，其网络请求需要审计和管控：

- Agent 可能访问恶意 URL 或被诱导将数据外泄
- 某些 API 调用可能产生费用（如调用付费 LLM 服务）
- 内网资源的访问需要权限控制
- 用户需要知道 Agent 访问了哪些外部服务

**关键约束**：`sandbox-exec` 只能做全有/全无的网络控制，无法按**域名、端口、协议**精细拦截。需要更高层的方案。

---

## 解决方案概览

| 方案 | 简述 | 适用场景 |
|------|------|----------|
| NetworkExtension 透明代理 | 系统级内核网络栈拦截 | macOS 应用级精细管控 |
| HTTP Proxy (环境变量注入) | 设置 `HTTP_PROXY` 使进程走代理 | 通用降级方案 |
| sandbox-exec 网络规则 | SBPL deny network-outbound | 全量禁止，无法按域名控制 |
| 防火墙规则 (pf) | BSD packet filter | 按 IP/端口，需 root |
| DLL 注入 Hook (Windows) | Hook socket API | Windows 平台方案 |

---

## 方案对比

| 维度 | NE 透明代理 | HTTP Proxy | sandbox-exec |
|------|------------|------------|--------------|
| 粒度 | 域名/IP/端口/协议 | 域名/端口 | 全有/全无 |
| 对进程透明 | ✅ 完全透明 | ⚠️ 需要进程遵循 proxy 变量 | ✅ |
| HTTPS 可见性 | SNI 域名（不解密内容） | 需要 MITM 证书 | 无 |
| 覆盖范围 | 所有 TCP 连接 | 仅 HTTP/HTTPS | 所有网络 |
| 性能 | 中（用户态中继） | 中 | 高（内核直接拒绝） |
| 进程识别 | ✅ 通过 PID/审计令牌 | ❌ 无法区分进程来源 | 不适用 |

**最佳实践**：NE 透明代理为主方案，HTTP Proxy 为降级方案（NE 不可用时自动切换）。

---

## NETransparentProxyProvider 架构

### 系统层次

```
┌─────────────────────────────────────────────────────────────┐
│  Agent 子进程                                                 │
│  执行: curl https://api.example.com                          │
└───────────────────────┬─────────────────────────────────────┘
                        │ TCP connect()
┌───────────────────────▼─────────────────────────────────────┐
│  macOS 内核网络栈                                             │
│  检测到 NETransparentProxy 已注册 → 拦截流量                   │
└───────────────────────┬─────────────────────────────────────┘
                        │ NEAppProxyTCPFlow
┌───────────────────────▼─────────────────────────────────────┐
│  NE Extension (独立进程)                                      │
│  NETransparentProxyProvider                                   │
│  ├── handleNewFlow(_:) → 决定接管还是放行                      │
│  ├── 提取域名（DNS 缓存 / SNI / Host 头）                     │
│  └── 决策：放行（双向中继）或拦截（close flow）                 │
└─────────────────────────────────────────────────────────────┘
```

### 核心回调流程

```swift
class TransparentProxyProvider: NETransparentProxyProvider {
    
    override func handleNewFlow(_ flow: NEAppProxyFlow) -> Bool {
        guard let tcpFlow = flow as? NEAppProxyTCPFlow else {
            return false  // UDP flow: return false = 丢弃（不是放行！）
        }
        
        // 1. 获取来源进程 PID
        let pid = flow.metaData.sourceAppAuditToken.pid
        
        // 2. 沿 PPID 链追溯，判断是否为沙盒子进程
        guard isSandboxedProcess(pid) else {
            return false  // 非沙盒进程 → 不接管 → 系统正常路由
        }
        
        // 3. 接管该 flow
        tcpFlow.open(withLocalEndpoint: nil) { error in
            // 4. 读取首包，提取域名
            tcpFlow.readData { data, error in
                let domain = self.extractDomain(from: data, flow: tcpFlow)
                
                // 5. 决策
                if self.shouldAllow(domain: domain) {
                    self.relay(flow: tcpFlow, initialData: data)  // 双向中继
                } else {
                    tcpFlow.closeReadWithError(nil)
                    tcpFlow.closeWriteWithError(nil)
                }
            }
        }
        return true  // 表示已接管
    }
}
```

---

## 关键技术细节

### 1. 进程识别：PPID 链追溯

Agent 子进程可能再 fork 出子子进程，需要沿 PPID 链向上查找：

```swift
/// 判断 PID 是否属于沙盒进程树
func isSandboxedProcess(_ pid: pid_t) -> Bool {
    var current = pid
    let registeredPids = loadRegisteredPids()  // 从 App Group 读取
    
    while current > 1 {
        if registeredPids.contains(current) {
            return true
        }
        current = getParentPid(current)  // 通过 proc_pidinfo 获取
    }
    return false
}
```

### 2. 三级域名提取策略

由于透明代理拿到的是原始 TCP 流，需要从中提取目标域名：

```
优先级从高到低：
┌─────────────────────────────────────────────────────────────┐
│ Level 1: DNS 缓存命中                                        │
│ 系统 DNS 查询时缓存 IP→Domain 映射，TTL=60s                   │
│ 优势：零延迟，连接建立前就知道域名                              │
├─────────────────────────────────────────────────────────────┤
│ Level 2: TLS ClientHello SNI                                 │
│ HTTPS 握手首包中明文携带目标域名                                │
│ 优势：不需要解密流量                                          │
├─────────────────────────────────────────────────────────────┤
│ Level 3: HTTP Host 头                                        │
│ 明文 HTTP 请求中的 Host 字段                                  │
│ 优势：简单直接                                                │
└─────────────────────────────────────────────────────────────┘
```

### 3. App Group 共享机制

NE Extension 与宿主 App 是**不同进程**，通过 App Group 共享数据：

```swift
// 宿主 App 侧：注册沙盒进程 PID
let defaults = UserDefaults(suiteName: "group.com.tencent.workbuddy.sandbox")
defaults?.set(registeredPids, forKey: "sandbox_pids")

// NE Extension 侧：读取已注册的 PID
let defaults = UserDefaults(suiteName: "group.com.tencent.workbuddy.sandbox")
let pids = defaults?.array(forKey: "sandbox_pids") as? [Int32] ?? []
```

共享数据包括：
- 沙盒进程 PID 注册表
- 域名黑/白名单规则
- 连接日志
- IP→Domain DNS 缓存映射

### 4. 为什么不能拦截 UDP/DNS？

这是一个关键的坑：

```
NETransparentProxyProvider 中：
  return false = 丢弃流量（不是放行！）
  return true  = 接管流量
```

如果将 UDP 纳入拦截范围：
1. macOS 的 DNS 查询由 `mDNSResponder` 系统进程统一发出
2. `mDNSResponder` 的 PID 不在沙盒进程列表中
3. `handleNewFlow` 对其返回 `false` → DNS 查询被丢弃
4. **全系统 DNS 解析失败**

因此 NE 配置中必须只包含 TCP：

```swift
let settings = NETransparentProxyNetworkSettings(tunnelRemoteAddress: "127.0.0.1")
// 只纳入 TCP 流量
settings.includedNetworkRules = [
    NENetworkRule(
        remoteNetwork: nil,       // 所有远程地址
        remotePrefix: 0,
        localNetwork: nil,
        localPrefix: 0,
        protocol: .TCP,           // 仅 TCP！
        direction: .outbound
    )
]
```

---

## 降级方案：HTTP Proxy

当 NE Extension 不可用时（用户未授权、系统异常等），自动降级到 HTTP Proxy：

```
Rust sandbox-core:
  QueryNetworkExtensionReady（500ms 超时）
    ├── ready = true  → NE 模式（透明拦截）
    └── ready = false → 启动本地 LocalProxyServer
                        注入 HTTP_PROXY/HTTPS_PROXY 环境变量到子进程
```

降级的代价：
- 只能拦截遵循 proxy 环境变量的 HTTP/HTTPS 流量
- 直接建立 TCP 连接的程序（如 `nc`、自定义 socket）不受控
- 无法按进程区分流量

---

## NE Extension 生命周期

| 事件 | 触发时机 | 注意事项 |
|------|----------|----------|
| `startProxy` | 首次配置加载或系统重启后 | 需在此设置 `NETransparentProxyNetworkSettings` |
| `handleNewFlow` | 每个新 TCP 连接建立时 | 必须快速返回，耗时操作异步处理 |
| `stopProxy` | 用户关闭或 Extension 异常 | 清理所有 pending flows |
| 系统杀死 | 内存压力 / 10s 无新 flow | Extension 会被按需重新唤起 |

---

## 延伸阅读

- [进程沙盒机制](../03-security/进程沙盒机制.md)
- [FileProvider 文件投影](../04-filesystem/FileProvider文件投影.md)
- [跨平台对比：IPC 通信机制](../../cross-platform/comparison/IPC通信机制.md)
