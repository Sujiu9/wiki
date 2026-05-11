# 🟡 Rust 跨语言边界

> 📋 前置知识：[NDJSON 协议从零实现](../03-IPC协议与服务进程/NDJSON协议从零实现.md)

本模块学习 Rust 核心如何被 Swift、C#、C/C++ 等宿主调用。

## 为什么需要跨语言边界

WorkBuddySandbox 的核心能力在 Rust，但平台 GUI 和 Extension 分别使用 Swift、C#、C/C++。跨语言边界有两条路：

1. **进程边界**：宿主启动 `sandbox-cli`，通过 IPC 调用；
2. **动态库边界**：宿主加载 `sandbox-ffi`，通过 C ABI 调用。

IPC 更适合业务命令，FFI 更适合低层能力或 Extension 中需要直接调用的函数。

## 学习顺序

1. [Rust cdylib 与 C ABI 最小示例](./Rust-cdylib与C-ABI最小示例.md)：先学会导出一个 C 可调用函数。
2. [宿主动态加载动态库](./宿主动态加载动态库.md)：再学会 `dlopen/dlsym` 和 `LoadLibrary/GetProcAddress`。

## 方案对比

| 方案 | 优点 | 缺点 |
|------|------|------|
| IPC | 崩溃隔离好，协议清晰 | 序列化开销，异步处理复杂 |
| 静态链接 | 调用简单 | 分发和 ABI 复杂 |
| 动态库 + C ABI | 多语言通用，性能好 | 内存所有权和错误处理复杂 |

## WorkBuddySandbox 映射

| 能力 | 项目路径 |
|------|----------|
| FFI crate 配置 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox-ffi/Cargo.toml` |
| 统一导出层 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox-ffi/src/lib.rs` |
| SandboxManager C ABI | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/ffi.rs` |
| ProjFS/overlay C ABI | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/projfs/src/ffi.rs` |
| Swift 动态加载 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/FileExtension/Sources/ProjFSNative.swift` |

## 学完本模块你应该能写什么

- 一个 Rust `cdylib`，导出 `extern "C"` 函数。
- 一个安全返回字符串的 FFI 接口，并提供释放函数。
- 一个 Swift/C# 宿主动态加载动态库并调用函数。
- 能说明为什么 WorkBuddySandbox 大业务走 IPC，而部分底层能力走 FFI。

## 延伸阅读

- [IPC 协议与服务进程](../03-IPC协议与服务进程/README.md)
- [覆盖层文件系统从零实现](../02-虚拟文件系统与投影/覆盖层文件系统从零实现.md)
