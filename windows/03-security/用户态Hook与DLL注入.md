# 🔴 Windows 用户态 Hook 与 DLL 注入

> 📋 前置知识：[NT 内核架构](../01-fundamentals/README.md)

## 为什么需要用户态 Hook？

Windows 与 macOS 不同，没有提供类似 `sandbox-exec` 的系统级进程沙盒工具。要实现 AI Agent 子进程的文件访问控制，有两条路：

| 方案 | 层级 | 优点 | 缺点 |
|------|------|------|------|
| 内核驱动 (minifilter) | Ring 0 | 不可绕过，覆盖全面 | 需要签名、蓝屏风险、部署复杂 |
| 用户态 DLL 注入 + Hook | Ring 3 | 无需驱动签名、部署简单、风险低 | 理论上可被绕过 |

AgentOS 选择了**用户态方案**（tsbx），通过 inline-hook ntdll 中的系统调用存根来实现文件访问控制。

---

## Windows 系统调用架构

理解 Hook 的前提是理解 Windows 的系统调用路径：

```
┌─────────────────────────────────────────────────────────────────┐
│  用户态 (Ring 3)                                                 │
│                                                                 │
│  应用程序                                                        │
│  ┌──────────┐                                                   │
│  │ CreateFileW("test.txt")                                      │
│  └────┬─────┘                                                   │
│       │ Win32 API                                               │
│       ▼                                                         │
│  ┌──────────────────────┐                                       │
│  │ kernel32.dll          │                                       │
│  │ CreateFileW() →       │                                       │
│  │   NtCreateFile()      │ ← 转发到 ntdll                       │
│  └────┬─────────────────┘                                       │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ ntdll.dll — 系统调用存根 (syscall stub)                    │   │
│  │                                                          │   │
│  │ NtCreateFile:                                            │   │
│  │   mov r10, rcx           ; 保存参数                       │   │
│  │   mov eax, 0x55          ; 系统调用号                     │   │
│  │   syscall                ; 陷入内核 ←── Hook 点！          │   │
│  │   ret                                                    │   │
│  └────┬─────────────────────────────────────────────────────┘   │
│       │                                                         │
├───────┼─────────────────────────────────────────────────────────┤
│       │ syscall 指令                                             │
│       ▼                                                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ ntoskrnl.exe — 内核                                       │   │
│  │                                                          │   │
│  │ NtCreateFile() 内核实现                                    │   │
│  │   → IoCreateFile()                                       │   │
│  │     → ObOpenObjectByName()                               │   │
│  │       → 文件系统驱动                                      │   │
│  └──────────────────────────────────────────────────────────┘   │
│                    内核态 (Ring 0)                                │
└─────────────────────────────────────────────────────────────────┘
```

**Hook 的最佳位置**：ntdll.dll 中的系统调用存根函数。这是用户态的"最后一关"——所有文件操作最终都会经过这里。

---

## Inline Hook 原理

### ntdll 函数的原始形态

```asm
; NtCreateFile 在 ntdll 中的原始代码（x64）
NtCreateFile:
    4C 8B D1        mov r10, rcx        ; 3 字节
    B8 55 00 00 00  mov eax, 0x55       ; 5 字节  ← syscall 号
    0F 05           syscall             ; 2 字节
    C3              ret                 ; 1 字节
; 总共约 11 字节的简单存根
```

### Hook 后的形态

```asm
; Hook 后的 NtCreateFile（前 N 字节被替换为跳转指令）
NtCreateFile:
    FF 25 00 00 00 00   jmp qword ptr [rip+0]   ; 6 字节（间接跳转）
    <8 bytes address>   ; Hook 函数的绝对地址     ; 8 字节
    ; 共 14 字节，覆盖了原始存根的前部分

; 跳转到：
hook_NtCreateFile:
    ; 1. 检查访问规则
    ; 2. 如果允许 → 调用原始函数（trampoline）
    ; 3. 如果拒绝 → 返回 STATUS_ACCESS_DENIED
```

### Trampoline（跳板）

被覆盖的原始指令需要保存，才能在"放行"时调用原函数：

```
┌──────────────────────────────────────────────────────────┐
│  Hook 函数入口                                            │
│  hook_NtCreateFile(args...):                             │
│    rule = check_rules(args.filename)                     │
│    if rule == ALLOW:                                     │
│      return trampoline_NtCreateFile(args)  ─────┐       │
│    else:                                         │       │
│      return STATUS_ACCESS_DENIED                 │       │
└──────────────────────────────────────────────────┼───────┘
                                                   │
┌──────────────────────────────────────────────────▼───────┐
│  Trampoline（跳板代码）                                    │
│                                                          │
│  ; 执行被覆盖的原始指令                                    │
│  mov r10, rcx                                            │
│  mov eax, 0x55                                           │
│  ; 跳回原函数被覆盖区域之后的位置                           │
│  jmp NtCreateFile + 11   ───────────────────────┐       │
└──────────────────────────────────────────────────┼───────┘
                                                   │
┌──────────────────────────────────────────────────▼───────┐
│  NtCreateFile + 11:                                      │
│  syscall                                                 │
│  ret                                                     │
│  ; 正常的系统调用                                          │
└──────────────────────────────────────────────────────────┘
```

---

## DLL 注入技术

### 为什么需要注入？

Hook 代码（tsbx.dll）需要被加载到**目标进程的地址空间**中才能 Hook 其 ntdll。注入方式取决于场景：

### 方案 1：入口点 Shellcode 注入（首选）

用于父进程启动子进程时注入：

```c
// 注入流程
PROCESS_INFORMATION pi;
// 1. 以 CREATE_SUSPENDED 创建目标进程
CreateProcessW(exe_path, cmdline, ..., CREATE_SUSPENDED, ..., &pi);

// 2. 在目标进程中分配内存
LPVOID remote_mem = VirtualAllocEx(pi.hProcess, NULL, 
    shellcode_size, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);

// 3. 写入 shellcode（加载 tsbx.dll 的代码）
WriteProcessMemory(pi.hProcess, remote_mem, shellcode, shellcode_size, NULL);

// 4. 修改入口点：替换为跳转到 shellcode
//    原始入口点会被 shellcode 在执行完毕后恢复
CONTEXT ctx;
ctx.ContextFlags = CONTEXT_FULL;
GetThreadContext(pi.hThread, &ctx);
// 保存原始 RIP
original_rip = ctx.Rip;
// 指向我们的 shellcode
ctx.Rip = (DWORD64)remote_mem;
SetThreadContext(pi.hThread, &ctx);

// 5. 恢复主线程
ResumeThread(pi.hThread);
// shellcode 会：加载 tsbx.dll → 执行 DllMain → 安装 Hook → 跳回原始入口点
```

### Shellcode 结构（x64，13 字节极简版）

```asm
; AgentOS tsbx 使用的 entry-point shellcode
FA                      cli                 ; 1 字节 - 占位/对齐
48 B8 XX XX XX XX XX XX XX XX  mov rax, <hook_entry_addr>  ; 10 字节
FF E0                   jmp rax             ; 2 字节
; 总共 13 字节
```

### 方案 2：rundll32 桥接（跨架构注入）

当 64 位进程需要注入 32 位子进程时：

```
x64 父进程
  │
  │ 无法直接操作 x86 进程的 ntdll
  ▼
启动 C:\Windows\SysWOW64\rundll32.exe (32-bit)
  │
  │ rundll32 加载 tsbx32.dll
  │ tsbx32.dll 在 32-bit 空间中完成 Hook
  ▼
目标 x86 进程被注入
```

---

## 子进程传播机制

### 问题

沙盒进程可能再启动子进程——如果不处理，子进程就逃逸了沙盒。

### 解决方案：Hook CreateProcessInternalW

```c
// tsbx Hook 的 CreateProcessInternalW
BOOL WINAPI hook_CreateProcessInternalW(
    HANDLE hToken,
    LPCWSTR lpApplicationName,
    LPWSTR lpCommandLine,
    ...
    DWORD dwCreationFlags,
    ...
    LPPROCESS_INFORMATION lpProcessInformation
) {
    // 1. 强制添加 CREATE_SUSPENDED 标志
    dwCreationFlags |= CREATE_SUSPENDED;
    
    // 2. 调用原始 CreateProcessInternalW
    BOOL result = original_CreateProcessInternalW(
        hToken, lpApplicationName, lpCommandLine, ...,
        dwCreationFlags, ..., lpProcessInformation
    );
    
    if (result) {
        // 3. 在新进程中注入 tsbx.dll
        inject_tsbx_dll(lpProcessInformation->hProcess, lpProcessInformation->hThread);
        
        // 4. 通过共享内存传递规则集
        setup_shared_memory_rules(lpProcessInformation->dwProcessId);
        
        // 5. 如果调用者原本没有 SUSPENDED 标志，恢复线程
        if (!(original_flags & CREATE_SUSPENDED)) {
            ResumeThread(lpProcessInformation->hThread);
        }
    }
    
    return result;
}
```

### 传播链路

```
Agent 进程 (已注入 tsbx)
  │
  │ CreateProcessW("cmd.exe /c build.bat")
  │ ← 被 Hook 拦截，强制 SUSPENDED
  ▼
cmd.exe (SUSPENDED, 已注入 tsbx)
  │
  │ CreateProcessW("msbuild.exe ...")
  │ ← 同样被 Hook 拦截
  ▼
msbuild.exe (SUSPENDED, 已注入 tsbx)
  │
  │ CreateProcessW("cl.exe ...")
  ▼
cl.exe (已注入 tsbx)

// 整棵进程树都在沙盒控制下
```

---

## 规则分发：共享内存协议

### 为什么用共享内存？

| 方案 | 延迟 | 适用性 |
|------|------|--------|
| Named Pipe | 微秒级 | 需要连接建立 |
| 文件 | 毫秒级 | 太慢 |
| **共享内存** | **纳秒级** | ✅ 零拷贝直读 |

### 共享内存布局

```
命名: "Local\LiteSandbox_<PID>"

┌────────────────────────────────────────────────────┐
│  Header (固定大小)                                   │
│  ┌──────────────────────────────────────────────┐  │
│  │ magic: u32 = 0x54534258 ("TSBX")            │  │
│  │ version: u32 = 1                            │  │
│  │ rules_offset: u32                           │  │
│  │ rules_size: u32                             │  │
│  │ audit_offset: u32                           │  │
│  │ audit_size: u32                             │  │
│  └──────────────────────────────────────────────┘  │
│                                                    │
│  Rules Section (JSON)                              │
│  ┌──────────────────────────────────────────────┐  │
│  │ {                                            │  │
│  │   "default_action": "deny",                 │  │
│  │   "rules": [                                │  │
│  │     {"pattern": "C:\\project\\*", "action": "allow"},  │
│  │     {"pattern": "C:\\Windows\\*", "action": "read-only"},│
│  │     {"pattern": "*.exe",         "action": "deny"}     │
│  │   ]                                         │  │
│  │ }                                            │  │
│  └──────────────────────────────────────────────┘  │
│                                                    │
│  Audit Ring Buffer                                 │
│  ┌──────────────────────────────────────────────┐  │
│  │ 环形缓冲区：记录每次 deny 事件                  │  │
│  │ {timestamp, pid, path, operation, result}    │  │
│  └──────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────┘
```

---

## Hook 决策引擎

```c
// tsbx 中的文件访问决策流程
NTSTATUS hook_NtCreateFile(
    PHANDLE FileHandle,
    ACCESS_MASK DesiredAccess,
    POBJECT_ATTRIBUTES ObjectAttributes,  // 包含文件路径
    ...
) {
    // 1. 提取文件路径
    UNICODE_STRING *path = ObjectAttributes->ObjectName;
    
    // 2. 路径规范化（处理 \??\、\Device\HarddiskVolume 等前缀）
    wchar_t normalized[MAX_PATH];
    normalize_nt_path(path, normalized);
    
    // 3. LRU 缓存查找
    cache_entry_t *cached = lru_lookup(normalized, DesiredAccess);
    if (cached) return cached->result;
    
    // 4. 规则匹配（从共享内存读取规则集）
    rule_action_t action = match_rules(normalized, DesiredAccess);
    
    switch (action) {
        case ALLOW:
            // 放行，调用原始 NtCreateFile
            result = trampoline_NtCreateFile(FileHandle, DesiredAccess, ...);
            break;
            
        case READ_ONLY:
            // 剥离写权限
            DesiredAccess &= ~(FILE_WRITE_DATA | FILE_APPEND_DATA | DELETE);
            result = trampoline_NtCreateFile(FileHandle, DesiredAccess, ...);
            break;
            
        case DENY:
            // 直接返回拒绝
            result = STATUS_ACCESS_DENIED;
            // 写入审计日志
            audit_log(normalized, DesiredAccess, DENIED);
            break;
    }
    
    // 5. 写入 LRU 缓存
    lru_insert(normalized, DesiredAccess, result);
    
    return result;
}
```

---

## 安全加固

### 防硬链接攻击

```c
// 攻击向量：在允许写入的目录中创建硬链接指向受保护文件
// 例：mklink /H C:\project\steal.txt C:\Users\secret\keys.txt

// 防护：检查文件的硬链接数
FILE_STANDARD_INFORMATION info;
NtQueryInformationFile(handle, &info, ...);
if (info.NumberOfLinks > 1) {
    // 多链接文件 → 可能是硬链接攻击
    // 进行额外的目标路径验证
    validate_all_links(handle);
}
```

### 防直接系统调用绕过

用户态 Hook 的已知局限：恶意代码可以直接构造 `syscall` 指令绕过 ntdll：

```asm
; 直接系统调用（绕过 ntdll Hook）
mov r10, rcx
mov eax, 0x55    ; NtCreateFile 的 syscall 号
syscall          ; 直接陷入内核
```

**缓解措施**：
- 对于 AI Agent 场景，子进程是 python/node/bash 等解释器，不太可能执行直接系统调用
- 配合进程注入检测（监控是否有代码在 ntdll 外执行 syscall 指令）
- 如需更强保护 → 转向内核驱动 minifilter 方案

---

## 与 macOS sandbox-exec 的对比

| 维度 | macOS sandbox-exec | Windows tsbx (用户态 Hook) |
|------|-------------------|---------------------------|
| 实现层级 | 内核 MACF 钩子 | 用户态 ntdll inline-hook |
| 不可绕过性 | ✅ 内核级，无法从用户态绕过 | ⚠️ 理论上可直接 syscall 绕过 |
| 部署复杂度 | 零部署（系统自带） | 需要 DLL 注入基础设施 |
| 子进程继承 | 自动（内核保证） | 手动（Hook CreateProcess 传播） |
| 性能开销 | 极低（内核原生路径） | 低（inline-hook + LRU 缓存） |
| 灵活性 | SBPL 语法，不算灵活 | 完全自定义，任意逻辑 |
| 驱动签名要求 | 无 | 无（纯用户态） |

---

## 延伸阅读

- [跨平台进程生命周期管理](../../cross-platform/comparison/进程生命周期管理.md)
- [跨平台 IPC 通信机制](../../cross-platform/comparison/IPC通信机制.md)
- [macOS 进程沙盒机制](../../macos/03-security/进程沙盒机制.md) — 对比内核级方案
