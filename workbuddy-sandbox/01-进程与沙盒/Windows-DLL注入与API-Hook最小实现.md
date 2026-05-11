# 🟡 Windows DLL 注入与 API Hook 最小实现

> 📋 前置知识：[从 fork/exec 到进程隔离](./从fork-exec到进程隔离.md)

## 1. 为什么 Windows 需要 Hook 思路

Windows 没有一个和 macOS `sandbox-exec` 完全等价、能用文本 profile 临时限制任意子进程的机制。要知道子进程访问了哪些文件、连接了哪些网络，常见做法是让受控进程加载一段可信代码，在关键 API 调用处做判断和记录。

这就是 `API Hook` 的基本动机：

- 程序调用 `CreateFileW` 打开文件；
- Hook 层先看到参数路径；
- 根据规则决定允许、拒绝或记录；
- 再调用原始 API 或返回错误。

> 本文只讨论**自己创建的受控子进程**中的防护/审计 Hook，不讨论绕过安全软件、隐蔽注入或未授权控制其他进程。

## 2. 解决方案概览

| 方案 | 简述 | 适用场景 |
|------|------|----------|
| JobObject | 管进程树、资源限制 | 生命周期、kill 子进程树 |
| AppContainer | 系统沙盒容器 | UWP/特定权限模型 |
| API Hook | 拦截文件/网络 API | 需要细粒度审计和动态规则 |
| 代理 | 拦截 HTTP/HTTPS 流量 | 能配置代理的进程 |

WorkBuddySandbox 的 Windows 侧使用 `tsbx` Hook 引擎，核心是“创建受控进程 + 注入/加载 Hook DLL + 根据规则记录文件和网络访问”。

## 3. 方案对比

| 维度 | JobObject | AppContainer | API Hook |
|------|-----------|--------------|----------|
| 进程生命周期 | 强 | 中 | 依赖宿主管理 |
| 文件访问拦截 | 弱 | 中 | 强 |
| 网络访问拦截 | 弱 | 中 | 强 |
| 实现复杂度 | 低 | 中 | 高 |
| 动态策略 | 弱 | 中 | 强 |

所以 Hook 不是为了“炫技”，而是为了拿到系统没有直接暴露的细粒度审计点。

## 4. 如何写一个概念级最小版本

最小模型分三层：

1. **宿主层**：创建子进程，并确保它加载 Hook DLL；
2. **Hook DLL**：在进程内替换目标 API 入口；
3. **规则/审计层**：判断路径或地址，把结果写回共享内存、管道或日志。

伪代码如下：

```cpp
// Hook 后的 CreateFileW 概念模型
HANDLE MyCreateFileW(LPCWSTR path, DWORD access, ...) {
    if (is_denied(path, access)) {
        record_block("file", path, access);   // 记录拦截事件
        SetLastError(ERROR_ACCESS_DENIED);
        return INVALID_HANDLE_VALUE;
    }

    return RealCreateFileW(path, access, ...); // 放行时调用原始 API
}
```

注意：真实 Hook 要处理线程安全、递归调用、宽字符路径、符号链接、短路径、网络路径、子进程继承等问题。

## 5. 常见坑

- **路径不规范**：`C:\Temp\a.txt`、`\\?\C:\Temp\a.txt`、短路径可能指向同一文件。
- **只 Hook 一个 API 不够**：文件可能通过 `CreateFileW`、硬链接、重命名、删除 API 间接变化。
- **递归调用**：Hook 内部写日志又触发文件 API，可能无限递归。
- **注入时机**：进程启动早期的访问可能发生在 Hook 初始化前。
- **规则同步**：宿主更新规则后，DLL 内部必须及时看到新规则。

## 6. 回到 WorkBuddySandbox

| 机制 | 项目位置 |
|------|----------|
| Windows 沙盒执行 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/executor/sandbox_executor/win.rs` |
| tsbx SDK 宿主侧 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/tsbx/tsbx_sdk/host.cpp` |
| DLL 注入逻辑 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/tsbx/tsbx_sdk/injector.cpp` |
| Hook DLL 入口 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/tsbx/tsbx/dllmain.cpp` |
| 文件 Hook | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/tsbx/tsbx/file_hooks.cpp` |
| 网络 Hook | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/tsbx/tsbx/network_hooks.cpp` |
| 进程 Hook | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/tsbx/tsbx/proc_hooks.cpp` |
| 规则解析 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/tsbx/shared/rules_parser.cpp` |

读 `tsbx` 时不要从所有 Hook 表开始。先只追一个问题：`CreateFileW("secret.txt")` 如何变成一条 `FileBlockRecord`？

## 7. 练习题：你要亲手实现什么

1. 写一个不注入的版本：包装自己的 `open_file(path)`，先判断 allow/deny，再调用系统 API。
2. 再写一个概念 Hook：把 `CreateFileW` 的路径打印出来，不做拒绝。
3. 加入规则：拒绝写入某个目录，返回 `ERROR_ACCESS_DENIED`。
4. 加入审计：把 `{path, action, decision}` 写到日志。

完成后再看 `file_hooks.cpp`，你会发现项目复杂度主要来自“覆盖所有边界情况”，核心思想仍是这个最小模型。

## 8. 延伸阅读

- [Windows Named Pipe 服务端](../03-IPC协议与服务进程/Windows-Named-Pipe服务端.md)
- [代理与透明代理基础](../06-网络拦截/代理与透明代理基础.md)
