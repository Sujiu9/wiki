# 🟡 Shell 配置文件的加载机制

> 📋 前置知识：[macOS 配置文件概览](./README.md)

## 为什么有这么多配置文件？

打开 macOS 的 Home 目录，你可能会看到：

```
~/.zshrc
~/.zshenv
~/.zprofile
~/.zlogin
~/.zlogout
~/.bash_profile
~/.bashrc
~/.profile
```

这些文件**不是重复发明轮子**，而是为了覆盖不同的 Shell 启动场景。核心问题是：

> Shell 的启动方式不止一种，不同方式需要执行不同的初始化逻辑。

如果没有这种分层设计，所有配置挤在一个文件里，会导致：
- 非交互式脚本（如 cron job）加载了不必要的 UI 配置（如 prompt 主题），拖慢执行
- SSH 登录时缺少必要的环境变量
- 子 Shell 重复执行只需一次的初始化逻辑（如 PATH 拼接导致重复）

---

## 两个关键维度

Shell 的启动方式由两个独立维度决定：

| 维度 | 含义 | 判断方式 |
|------|------|----------|
| **Login vs Non-login** | 是否是用户"登入"系统的第一个 Shell | 是否需要读取用户身份环境 |
| **Interactive vs Non-interactive** | 是否有人在键盘前交互 | 是否有 TTY / prompt |

组合起来就是 4 种情况：

| 场景 | Login? | Interactive? | 典型例子 |
|------|:------:|:------------:|----------|
| 打开 Terminal.app | ✅ | ✅ | 日常使用终端 |
| SSH 远程登录 | ✅ | ✅ | `ssh user@host` |
| 运行脚本 | ❌ | ❌ | `zsh script.sh` |
| 在终端里开子 Shell | ❌ | ✅ | 在终端输入 `zsh` |

---

## Zsh 的加载顺序（macOS 默认 Shell）

从 macOS Catalina (10.15) 起，默认 Shell 从 bash 切换到了 **Zsh**。Zsh 的配置文件加载顺序如下：

```
┌─────────────────────────────────────────────────────────┐
│                    所有情况都加载                          │
│  /etc/zshenv  →  ~/.zshenv                              │
├─────────────────────────────────────────────────────────┤
│                  仅 Login Shell 加载                      │
│  /etc/zprofile  →  ~/.zprofile                          │
├─────────────────────────────────────────────────────────┤
│               仅 Interactive Shell 加载                   │
│  /etc/zshrc  →  ~/.zshrc                                │
├─────────────────────────────────────────────────────────┤
│             仅 Login + Interactive 加载                   │
│  /etc/zlogin  →  ~/.zlogin                              │
├─────────────────────────────────────────────────────────┤
│                  Login Shell 退出时                       │
│  ~/.zlogout  →  /etc/zlogout                            │
└─────────────────────────────────────────────────────────┘
```

### 完整时序（Login + Interactive）

```
1. /etc/zshenv       ← 系统级，总是执行
2. ~/.zshenv         ← 用户级，总是执行
3. /etc/zprofile     ← 系统级，Login 时
4. ~/.zprofile       ← 用户级，Login 时
5. /etc/zshrc        ← 系统级，Interactive 时
6. ~/.zshrc          ← 用户级，Interactive 时
7. /etc/zlogin       ← 系统级，Login 时（在 rc 之后）
8. ~/.zlogin         ← 用户级，Login 时（在 rc 之后）
```

---

## 每个文件该放什么？

| 文件 | 加载条件 | 推荐放置内容 |
|------|----------|--------------|
| `~/.zshenv` | **总是** | 最基础的环境变量（如 `EDITOR`、`LANG`），不能有输出 |
| `~/.zprofile` | Login | PATH 拼接、一次性初始化（如 homebrew、nvm） |
| `~/.zshrc` | Interactive | 别名、prompt 主题、插件、补全、bindkey |
| `~/.zlogin` | Login（rc 之后） | 登录欢迎信息（很少使用） |
| `~/.zlogout` | 退出 Login Shell | 清理操作（很少使用） |

### 黄金法则

```
环境变量 → ~/.zshenv 或 ~/.zprofile
交互配置 → ~/.zshrc
其他很少需要动
```

---

## macOS Terminal.app 的特殊行为

⚠️ **关键：macOS 的 Terminal.app 每次打开新窗口/Tab 都视为 Login Shell**。

这与 Linux 不同（Linux 下新开 Terminal 通常是 Non-login Interactive Shell）。

```bash
# 验证当前是否为 Login Shell
echo $0
# 输出 "-zsh" 表示 Login Shell（前缀有 -）
# 输出 "zsh" 表示 Non-login Shell
```

这意味着在 macOS 上：
- `~/.zprofile` 和 `~/.zshrc` **每次开新终端窗口都会执行**
- 你不需要像 Linux 那样纠结"到底是 login 还是 non-login"

---

## Bash 的对照（了解即可）

如果你还在用 bash 或需要维护旧配置：

| Bash 文件 | 对应的 Zsh 文件 | 加载条件 |
|-----------|-----------------|----------|
| `~/.bash_profile` | `~/.zprofile` | Login |
| `~/.bashrc` | `~/.zshrc` | Interactive (Non-login) |
| `~/.profile` | `~/.zprofile` | Login（bash 没有 bash_profile 时的 fallback） |

⚠️ Bash 的一个坑：**Login Shell 不会自动加载 `.bashrc`**，所以很多人在 `.bash_profile` 里手动 `source ~/.bashrc`。Zsh 没有这个问题。

---

## 实际配置建议

```bash
# ~/.zshenv — 极简，只放必须全局生效的变量
export EDITOR="nvim"
export LANG="en_US.UTF-8"
```

```bash
# ~/.zprofile — PATH 和一次性工具初始化
eval "$(/opt/homebrew/bin/brew shellenv)"  # Homebrew
export PATH="$HOME/.local/bin:$PATH"
```

```bash
# ~/.zshrc — 日常交互配置
# 别名
alias ll="ls -lah"
alias gs="git status"

# Prompt（使用 starship 为例）
eval "$(starship init zsh)"

# 插件管理（以 zinit 为例）
source ~/.zinit/zinit.zsh
zinit light zsh-users/zsh-autosuggestions
```

---

## 排查"配置不生效"的方法

```bash
# 1. 确认当前 Shell 类型
echo $0           # -zsh = login, zsh = non-login
echo $ZSH_VERSION # 确认是 zsh 不是 bash

# 2. 追踪加载顺序（在每个文件开头加调试行）
echo ">>> loading ~/.zshenv" >> /tmp/shell-debug.log

# 3. 以特定模式启动来测试
zsh -l            # 模拟 Login Shell
zsh -i            # 模拟 Interactive Shell
zsh -c "echo \$PATH"  # 模拟 Non-interactive（脚本模式）
```

---

## 延伸阅读

- [环境变量配置](./环境变量配置.md)
- [跨平台对比：配置体系](../../cross-platform/comparison/配置体系对比.md)
