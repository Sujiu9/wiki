# macOS 配置文件

## 概览

macOS 的配置体系与 Windows 完全不同，没有"注册表"这样的集中式配置存储，而是采用分散式的文件配置方案。

## 核心配置机制

| 机制 | 用途 | 文件格式 |
|------|------|----------|
| Property List (plist) | 应用/系统偏好设置 | XML / Binary plist |
| launchd | 守护进程/定时任务管理 | plist (XML) |
| defaults | 命令行读写用户偏好 | - |
| Configuration Profile | MDM/企业设备管理 | mobileconfig (XML) |
| shell 配置 | 环境变量、别名 | .zshrc / .bash_profile |

## 目录

- [plist 文件详解](./plist.md) 🟢
- [launchd 与守护进程](./launchd.md) 🟡
- [defaults 命令](./defaults-command.md) 🟢
- [Shell 配置文件加载机制](./shell-config-loading.md) 🟡
- [环境变量配置](./environment-variables.md) 🟢
- [MDM 与配置描述文件](./mdm-profiles.md) 🔴

## plist 文件的常见位置

```
~/Library/Preferences/              # 用户级应用偏好
/Library/Preferences/               # 系统级应用偏好
~/Library/LaunchAgents/             # 用户级启动代理
/Library/LaunchAgents/              # 系统级启动代理（用户登录后运行）
/Library/LaunchDaemons/             # 系统级守护进程（系统启动时运行）
/System/Library/LaunchDaemons/      # Apple 系统守护进程（SIP 保护）
```

## 快速上手

```bash
# 读取某个应用的配置
defaults read com.apple.finder

# 修改 Finder 显示隐藏文件
defaults write com.apple.finder AppleShowAllFiles -bool true

# 查看某个 plist 文件内容
plutil -p ~/Library/Preferences/com.apple.finder.plist
```
