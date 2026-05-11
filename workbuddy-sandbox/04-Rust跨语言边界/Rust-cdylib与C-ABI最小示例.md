# 🟡 Rust cdylib 与 C ABI 最小示例

> 📋 前置知识：理解函数调用、指针、字符串内存。

## 1. 为什么需要 C ABI

Rust 的函数名、结构体布局、错误处理都不是天然给其他语言稳定调用的。Swift/C#/C++ 要调用 Rust，最通用的公共语言是 `C ABI（C Application Binary Interface）`。

所以 Rust 要做三件事：

1. 编译成 `cdylib`；
2. 导出 `extern "C"` 函数；
3. 明确内存由谁分配、谁释放。

## 2. 解决方案概览

| 方案 | 简述 | 适用场景 |
|------|------|----------|
| `cdylib + C ABI` | 导出 C 风格函数 | 多语言通用 |
| `cbindgen` | 生成 C 头文件 | 接口较多时 |
| IPC | 通过协议调用 | 复杂业务命令 |
| UniFFI | 自动生成绑定 | 生态适配时 |

WorkBuddySandbox 的 `sandbox-ffi` 是统一动态库，导出 sandbox 和 projfs 两类 C ABI。

## 3. 方案对比

| 维度 | C ABI | IPC |
|------|-------|-----|
| 调用开销 | 低 | 中 |
| 崩溃隔离 | 差 | 好 |
| 内存复杂度 | 高 | 低 |
| 异步事件 | 不方便 | 方便 |

所以：业务命令优先 IPC；Extension 中需要直接枚举/合并 overlay 时，可以用 FFI。

## 4. 如何写一个最小版本

`Cargo.toml`：

```toml
[lib]
crate-type = ["cdylib"]
```

Rust 导出函数：

```rust
use std::ffi::{CStr, CString};
use std::os::raw::c_char;

#[no_mangle]
pub extern "C" fn hello(name: *const c_char) -> *mut c_char {
    let name = unsafe { CStr::from_ptr(name) }.to_string_lossy();
    CString::new(format!("hello {name}")).unwrap().into_raw()
}

#[no_mangle]
pub extern "C" fn free_string(ptr: *mut c_char) {
    if !ptr.is_null() {
        unsafe { let _ = CString::from_raw(ptr); }
    }
}
```

关键：`into_raw()` 把字符串所有权交给宿主，宿主必须调用 `free_string()` 还回来。

## 5. 常见坑

- **Rust panic 穿过 FFI 边界**：不要让 panic 直接跨语言传播。
- **字符串释放不配对**：Rust 分配的内存必须 Rust 释放。
- **结构体布局不稳定**：跨 FFI 的结构体要 `#[repr(C)]`。
- **bool/usize 等类型差异**：优先使用明确宽度类型，如 `i32/u64`。
- **last error 线程安全**：错误信息通常要用 thread-local 或显式 out 参数。

## 6. 回到 WorkBuddySandbox

| 机制 | 项目位置 |
|------|----------|
| `cdylib` 配置 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox-ffi/Cargo.toml` |
| 统一 re-export | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox-ffi/src/lib.rs` |
| sandbox FFI | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/sandbox/src/ffi.rs` |
| projfs FFI | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/src/projfs/src/ffi.rs` |
| C 结构体头文件 | `/Users/huangfangyi/Tencent/wb/WorkBuddySandbox/example/mac/FileExtension/projfs_types.h` |

读 `src/projfs/src/ffi.rs` 时重点看：handle 如何创建/释放、list 如何返回、last error 如何保存。

## 7. 练习题：你要亲手实现什么

1. 写一个 Rust `cdylib`，导出 `add(a,b) -> i32`。
2. 导出 `hello(name) -> char*`，并提供 `free_string`。
3. 故意传入 null 指针，返回错误码而不是 panic。
4. 给一个 `#[repr(C)] struct Item { path: *const c_char, size: u64 }`，解释它如何释放。

## 8. 延伸阅读

- [宿主动态加载动态库](./宿主动态加载动态库.md)
- [macOS FileProvider 最小 Provider](../02-虚拟文件系统与投影/macOS-FileProvider最小Provider.md)
