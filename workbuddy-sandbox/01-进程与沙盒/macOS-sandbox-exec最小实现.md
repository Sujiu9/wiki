# 🟡 macOS sandbox-exec 最小实现

> 📋 前置知识：[从 fork/exec 到进程隔离](./从fork-exec到进程隔离.md)

## 1. 为什么需要 sandbox-exec

普通子进程只能控制“怎么启动”，不能控制“它能访问什么”。Agent 如果能随意读写用户目录，就无法形成可审计边界。

`沙盒执行（sandbox execution）` 的目标是：

- 明确允许哪些路径；
- 拒绝敏感路径；
- 在拒绝时留下日志，方便提示用户选择；
- 不让业务代码自己拦每一次文件调用。

## 2. 解决方案概览

| 方案 | 简述 | 适用场景 |
|------|------|----------|
| 应用沙盒 entitlements | App 级别权限 | GUI App 长期运行 |
| `sandbox-exec` | 单次命令使用 profile 限权 | 子进程隔离、临时策略 |
| 自己 Hook 文件 API | 拦截 open/read/write | macOS 上实现复杂，稳定性差 |

WorkBuddySandbox 在 macOS 执行命令时采用 `sandbox-exec` 作为主要隔离手段，并配合日志监听收集 deny 记录。

## 3. 方案对比

| 维度 | `sandbox-exec` | App Sandbox | API Hook |
|------|----------------|-------------|----------|
| 粒度 | 每个子进程 | 每个 App/Extension | 每个 API 调用 |
| 实现成本 | 中 | 低到中 | 高 |
| 动态规则 | profile 可生成 | 较弱 | 强 |
| 可审计性 | 依赖系统 deny 日志 | 较弱 | 可自定义 |

`WorkBuddySandbox` 需要按 session 动态生成规则，所以更适合 `sandbox-exec profile`。

## 4. 如何写一个最小版本

SBPL（Sandbox Profile Language）可以描述 allow/deny 规则。最小思路：默认拒绝写入，只允许写 `/tmp/sandbox-out`。

```scheme
(version 1)
(deny default)
(allow process*)
(allow file-read*)
(allow file-write* (subpath "/tmp/sandbox-out"))
```

运行：

```bash
mkdir -p /tmp/sandbox-out
cat > /tmp/demo.sb <<'EOF'
(version 1)
(deny default)
(allow process*)
(allow file-read*)
(allow file-write* (subpath "/tmp/sandbox-out"))
EOF

sandbox-exec -f /tmp/demo.sb /bin/bash -c 'echo ok > /tmp/sandbox-out/a; echo bad > /tmp/a'
```

预期：写 `/tmp/sandbox-out/a` 成功，写 `/tmp/a` 被拒绝。

## 5. 常见坑

- **默认 allow 太大**：如果 `(allow default)` 再补 deny，很容易漏掉敏感路径。
- **路径转义错误**：profile 是文本语言，路径必须正确转义，否则可能产生规则注入。
- **系统日志延迟**：deny 日志不一定和进程退出完全同步。
- **GUI 环境 PATH 不完整**：通过 App 启动的子进程可能找不到 `/opt/homebrew/bin`。
- **不是所有系统能力都适合 sandbox-exec**：网络透明拦截还需要 NetworkExtension 或代理配合。

## 6. 回到 WorkBuddySandbox

| 机制 | 项目位置 |
|------|----------|
| SBPL profile 生成 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/platform/macos/sandbox_profile.rs` |
| sandbox-exec 启动 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/platform/macos/process.rs` |
| macOS 执行编排 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/executor/sandbox_executor/mac.rs` |
| deny 日志监听 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/platform/macos/sandbox_exec_monitor.rs` |
| 平台规则翻译 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/permission/platform_rules.rs` |

读这些文件时重点看两条链路：

1. `PermissionManager` 如何变成 SBPL 文本；
2. 系统 deny 日志如何变成 `FileBlockRecord`。

## 7. 练习题：你要亲手实现什么

1. 写一个 profile：允许读当前目录，禁止写当前目录。
2. 写一个 Rust/Swift 脚本：动态生成 profile，并调用 `sandbox-exec -f profile command`。
3. 使用 `log stream` 观察 deny 日志，记录被拒绝的路径。
4. 把 deny 日志转换成结构体：`{path, action, raw}`。

做到这一步，你就不是“知道项目用了 sandbox-exec”，而是能自己写一个最小 macOS 命令沙盒。

## 8. 延伸阅读

- [macOS 代码签名与 entitlements 实战](../07-调试与系统权限/macOS代码签名与entitlements实战.md)
- [代理与透明代理基础](../06-网络拦截/代理与透明代理基础.md)
