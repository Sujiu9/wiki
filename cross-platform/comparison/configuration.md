# 配置体系对比：macOS vs Windows

## 设计哲学

| 方面 | macOS | Windows |
|------|-------|---------|
| 核心思想 | 分散式文件配置 | 集中式注册表 |
| 配置格式 | plist (XML/Binary) | 注册表 (二进制 hive) |
| 用户配置位置 | ~/Library/Preferences/ | HKCU\ |
| 系统配置位置 | /Library/, /System/Library/ | HKLM\ |
| 命令行工具 | `defaults`, `plutil` | `reg`, PowerShell |
| 配置权限 | Unix 文件权限 + SIP | ACL on Registry Keys |
| 备份/迁移 | 复制 plist 文件 | 导出 .reg 文件 |

## 应用偏好设置

### macOS
```bash
# 存储位置
~/Library/Preferences/com.company.appname.plist

# 编程接口 (Objective-C / Swift)
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
[defaults setObject:@"value" forKey:@"key"];
```

### Windows
```powershell
# 存储位置
HKCU\Software\Company\AppName

# 编程接口 (C++)
RegSetValueEx(hKey, L"key", 0, REG_SZ, (BYTE*)L"value", size);
```

## 后台服务/守护进程

### macOS — launchd
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" ...>
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.company.mydaemon</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/mydaemon</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

### Windows — 服务 (SCM)
```powershell
# 创建服务
New-Service -Name "MyService" -BinaryPathName "C:\Program Files\MyApp\service.exe"

# 或通过 sc 命令
sc create MyService binPath= "C:\Program Files\MyApp\service.exe" start= auto
```

## 定时任务

| 方面 | macOS | Windows |
|------|-------|---------|
| 机制 | launchd (StartCalendarInterval) | 任务计划程序 (Task Scheduler) |
| 配置格式 | plist | XML |
| 命令行 | `launchctl` | `schtasks` |
| GUI 工具 | 无官方（第三方 LaunchControl） | 任务计划程序 MMC |

## 环境变量

### macOS
```bash
# 用户级（当前 shell）
export PATH="/usr/local/bin:$PATH"    # 写入 ~/.zshrc

# 系统级（所有用户）
/etc/paths                             # 路径配置
/etc/paths.d/                          # 分文件路径配置
launchctl setenv VAR value             # GUI 应用可见
```

### Windows
```powershell
# 用户级
[Environment]::SetEnvironmentVariable("VAR", "value", "User")

# 系统级（需管理员）
[Environment]::SetEnvironmentVariable("VAR", "value", "Machine")

# 存储位置
# 用户: HKCU\Environment
# 系统: HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment
```

## 开发建议

1. **跨平台应用**：使用统一的配置抽象层，底层分别对接 plist 和注册表
2. **配置迁移**：设计时考虑配置的导出/导入格式（如 JSON/YAML 中间格式）
3. **权限管理**：两个平台都区分用户级和系统级配置，注意权限提升的不同方式
4. **配置监听**：macOS 用 `FSEvents` / `KVO`，Windows 用 `RegNotifyChangeKeyValue`
