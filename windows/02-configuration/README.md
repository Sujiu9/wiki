# Windows 配置体系

## 概览

Windows 采用以**注册表（Registry）**为核心的集中式配置管理，辅以各类配置文件。

## 核心配置机制

| 机制 | 用途 | 存储形式 |
|------|------|----------|
| 注册表 (Registry) | 系统/应用/用户配置的中心存储 | 二进制 hive 文件 |
| 组策略 (Group Policy) | 企业环境下的集中配置管理 | .admx / .adml / Registry |
| INI 文件 | 传统应用配置（已较少使用） | 纯文本 |
| XML 配置 | .NET 应用、任务计划等 | XML |
| 环境变量 | 路径、临时目录等系统变量 | 注册表中存储 |

## 目录

- [注册表详解](./注册表详解.md) 🟢
- [注册表编程](./注册表编程.md) 🟡
- [组策略](./组策略.md) 🟡
- [环境变量](./环境变量.md) 🟢
- [Windows 服务配置](./Windows服务配置.md) 🟡

## 注册表核心结构

```
HKEY_LOCAL_MACHINE (HKLM)    # 本机系统配置
├── SOFTWARE                  # 已安装软件配置
├── SYSTEM                    # 系统启动/驱动配置
├── HARDWARE                  # 硬件信息（动态生成）
└── SAM                       # 安全账户管理器

HKEY_CURRENT_USER (HKCU)     # 当前用户配置
├── SOFTWARE                  # 用户级软件配置
├── Environment               # 用户环境变量
└── Control Panel             # 用户界面偏好

HKEY_CLASSES_ROOT (HKCR)     # 文件关联、COM 注册
HKEY_USERS (HKU)             # 所有用户配置
HKEY_CURRENT_CONFIG (HKCC)   # 当前硬件配置
```

## 快速上手

```powershell
# 读取注册表值
Get-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer"

# 设置注册表值
Set-ItemProperty -Path "HKCU:\Software\MyApp" -Name "Setting1" -Value "Hello"

# 命令行工具
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion" /v ProductName
```

```c
// C/C++ 读取注册表
HKEY hKey;
RegOpenKeyEx(HKEY_LOCAL_MACHINE, L"SOFTWARE\\MyApp", 0, KEY_READ, &hKey);
DWORD value, size = sizeof(DWORD);
RegQueryValueEx(hKey, L"Setting1", NULL, NULL, (LPBYTE)&value, &size);
RegCloseKey(hKey);
```
