---
title: 在 Linux 服务器上搭建 OpenClaw AI 助手
description: 详细教程：在 Linux（CentOS/RHEL 或 Ubuntu/Debian）服务器上安装、配置和运行 OpenClaw，打造你的私人 AI 助手。涵盖预构建二进制、Docker、源码编译三种方式。
keywords: OpenClaw, AI 助手, Linux 安装, Docker, 自部署, 智能助手
categories:
  - 技术笔记
  - AI
---

## 0 前言：什么是 OpenClaw？

OpenClaw 是一个开源的 AI 助手框架，让你能在自己的服务器上运行一个类似 Claude Desktop 的智能助手。它可以：

- 读取你的文件、日历、邮件等上下文
- 通过工具调用与外部系统交互（shell、浏览器、GitHub、各种 API）
- 支持多通道接入（Telegram、Discord、微信、Web 等）
- 完全本地可控，数据私密

本文假设你有一台 Linux 服务器（CentOS/RHEL 8+ 或 Ubuntu/Debian），手上有 root 或 sudo 权限。准备在服务器上部署一个生产可用的 OpenClaw 实例。

> **目标**：安装完成后，你可以通过 `openclaw` CLI 运行命令，并通过 Web UI 或消息通道与助手交互。

---

## 1 环境要求

| 组件 | 要求 |
|------|------|
| **Node.js** | v22+（OpenClaw 依赖新版 Node）|
| **内存** | ≥ 2GB（建议 4GB+，如果使用本地的 LLM 模型） |
| **磁盘** | ≥ 5GB（模型文件可能占用几 GB）|
| **操作系统** | Linux（CentOS/RHEL 8+, Ubuntu 20.04+, Debian 11+） |
| **网络** | 能访问 external APIs（OpenRouter、Anthropic 等）|
| **Docker**（可选）| Podman v3+ 或 Docker v20+（如果用容器方式） |

如果你的机器没有 Node 22，OpenClaw 官方的安装脚本会自动帮你装一个独立版本（不会影响系统自带的 Node）。

---

## 2 三种安装方式

官方推荐的安装方式有三种，按场景选择：

| 方式 | 适用场景 | 复杂度 |
|------|----------|--------|
| **安装脚本** | 个人服务器、快速上手 | ⭐ |
| **npm/pnpm** | 你已熟悉 Node 版本管理 | ⭐⭐ |
| **源码/Docker** | 贡献代码或需要环境隔离 | ⭐⭐⭐ |

下面分别介绍。

---

### 2.1 方式一：安装脚本（最推荐）

安装脚本会：

1. 检测系统类型
2. 下载预构建的 OpenClaw 二进制（或编译源码）
3. 安装到 `$HOME/.openclaw/bin` 或系统目录
4. 可选运行 `onboard` 向导，配置网关和一些基础设置

**CentOS/RHEL（及类系统）**：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

**Ubuntu/Debian**：

同上，脚本会自动适配。如果要跳过 Onboarding 向导，只安装二进制：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --no-onboard
```

安装完成后，脚本会提示你将 OpenClaw 加入 PATH。通常自动操作后重启终端即可。

验证安装：

```bash
openclaw version
openclaw doctor
```

---

### 2.2 方式二：npm/pnpm（适合开发者）

如果你已用 nvm/fnm/asdf 管理 Node 22+，可以用包管理器全局安装：

**npm**：

```bash
npm install -g openclaw@latest
# 可能会遇到 sharp 编译问题，见下方排错
openclaw onboard --install-daemon
```

**pnpm**（推荐用于开发）：

```bash
pnpm add -g openclaw@latest
pnpm approve-builds -g  # 批准 openclaw、sharp、node-llama-cpp 等包
openclaw onboard --install-daemon
```

> **排错：sharp 编译错误**  
> - macOS 常见，如果系统装了 libvips，强制使用预编译二进制：`SHARP_IGNORE_GLOBAL_LIBVIPS=1 npm install -g openclaw@latest`  
> - Linux 需要 build-essential、python、make、g++ 等。CentOS：`sudo yum groupinstall "Development Tools"` 和 `sudo yum install python3`  
> - 或使用安装脚本避免编译

---

### 2.3 方式三：Docker/Podman（隔离环境）

生产环境推荐容器化部署。你可以用 Docker Compose 或直接用 `docker run`。

官方提供两个镜像：

- `openclaw/gateway:latest`：主程序（包含 CLI 和 daemon）
- `openclaw/dashboard:latest`：Web 管理界面

快速启动示例（需映射配置目录）：

```bash
mkdir -p ~/.openclaw
docker run -d \
  --name openclaw-gateway \
  -p 3000:3000 \
  -v ~/.openclaw:/app/.openclaw \
  -e OPENCLAW_CHANNELS=web \
  openclaw/gateway:latest
```

开启 Web UI（另一容器）：

```bash
docker run -d \
  --name openclaw-dashboard \
  -p 8080:8080 \
  --link openclaw-gateway \
  -e OPENCLAW_GATEWAY_URL=http://openclaw-gateway:3000 \
  openclaw/dashboard:latest
```

更完整的 `docker-compose.yml` 参考官方文档：

```yaml
version: "3.8"
services:
  gateway:
    image: openclaw/gateway:latest
    ports:
      - "3000:3000"
    volumes:
      - ~/.openclaw:/app/.openclaw
    environment:
      - OPENCLAW_CHANNELS=web,telegram
    restart: unless-stopped

  dashboard:
    image: openclaw/dashboard:latest
    ports:
      - "8080:8080"
    environment:
      - OPENCLAW_GATEWAY_URL=http://gateway:3000
    depends_on:
      - gateway
    restart: unless-stopped
```

---

## 3 首次运行与 Onboarding

安装完成后，运行：

```bash
openclaw onboard --install-daemon
```

这个命令会：

1. 询问是否安装并启动 Gateway 服务（后台守护进程）
2. 询问是否启用 Web 通道（浏览器访问）
3. 询问是否配置 Telegram 通道（可选）
4. 生成初始配置文件 `~/.openclaw/config.json`

**跳过向导**，直接安装后台服务：

```bash
openclaw install-d
```

之后可以用 `openclaw gateway start` 启动服务。

---

## 4 目录结构与配置文件

OpenClaw 默认将配置和状态放在 `~/.openclaw/`，结构如下：

```
~/.openclaw/
├── config.json           # 主配置文件
├── workspace/            # 助手的工作区（临时文件、缓存）
├── logs/                 # 运行日志
├── state.json            # 持久化状态（会话、技能配置）
├── skills/               # 已安装的技能（AgentSkills）
└── channels/             # 通道专用配置（如 Telegram token）
```

**config.json 示例片段**：

```json
{
  "model": {
    "provider": "openrouter",
    "id": "anthropic/claude-3-haiku"
  },
  "tools": {
    "allowlist": ["*"]
  },
  "channels": {
    "web": {
      "enabled": true,
      "port": 3000
    }
  }
}
```

详细配置项参考 `openclaw config help`。

---

## 5 启动服务

### 5.1 使用内置 service manager

OpenClaw 自带一个轻量的系统服务管理器：

```bash
# 安装为系统服务（systemd 或 launchd）
openclaw install-service

# 启动/停止/重启
openclaw service start
openclaw service status
openclaw service logs -f

# 卸载服务
openclaw uninstall-service
```

### 5.2 手动运行（调试）

```bash
openclaw gateway --verbose
```

---

## 6 访问助手

- **Web UI**：打开浏览器 `http://你的服务器IP:3000`（默认端口，可在配置中改）
- **命令行**：`openclaw chat "你好"`
- **消息通道**：Telegram、Discord、微信等（需单独配置 token 和 webhook）

Web 界面是单页应用，首次访问会用浏览器本地存储保存会话。登录后可以：

- 创建/切换会话
- 查看工具调用历史
- 管理技能
- 修改全局配置（部分）

---

## 7 配置 AI 模型

OpenClaw 支持多种提供商：

| provider | 示例 model id | 备注 |
|----------|---------------|------|
| openrouter | `anthropic/claude-3.7-sonnet` | 需 OPENROUTER_API_KEY |
| openai | `gpt-4o` | 需 OPENAI_API_KEY |
| ollama | `llama3.3:70b` | 本地模型，需运行 Ollama 服务 |
| anthropic | `claude-3-opus-20240229` | 需 ANTHROPIC_API_KEY |
| gemini | `gemini-2.0-flash` | 需 GOOGLE_API_KEY |

在 `config.json` 中配置：

```json
{
  "model": {
    "provider": "openrouter",
    "id": "anthropic/claude-3-haiku"
  }
}
```

环境变量方式（覆盖配置）：

```bash
export OPENROUTER_API_KEY="你的 key"
export OPENCLAW_MODEL="anthropic/claude-3-haiku"
```

---

## 8 添加技能（AgentSkills）

OpenClaw 的核心是可扩展的技能系统。安装新技能：

```bash
# 从 ClawHub 搜索安装
openclaw skills search "github"
openclaw skills install gh-issues

# 列出已安装技能
openclaw skills list

# 更新所有技能
openclaw skills update all

# 卸载技能
openclaw skills uninstall <skill-id>
```

每个技能自带独立的配置，安装后可以通过 Web UI 或 CLI 进一步配置。

---

## 9 常见排错

### 9.1 `openclaw: command not found`

安装脚本默认安装在 `$HOME/.openclaw/bin`，确认该目录在 PATH：

```bash
echo $PATH | grep -q "$HOME/.openclaw/bin" || export PATH="$HOME/.openclaw/bin:$PATH"
# 加入 ~/.bashrc 或 ~/.zshrc
```

### 9.2 端口冲突

Web 通道默认使用 3000 端口。修改 `config.json`：

```json
{
  "channels": {
    "web": {
      "port": 8080
    }
  }
}
```

然后重启服务。

### 9.3 API Key 错误

确保对应 provider 的环境变量已设置，例如：

```bash
export OPENROUTER_API_KEY="sk-..."
```

可以放入 `~/.bashrc` 或系统服务的 EnvironmentFile（systemd 用 `EnvironmentFile=/etc/default/openclaw`）。

### 9.4 内存不足

本地模型（如 Ollama）需要大内存。如果 OOM，换小模型或添加 swap：

```bash
# CentOS/RHEL
sudo swapon -a  # 检查 swap
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab
```

### 9.5 Gateway 无法启动

查看日志：

```bash
openclaw gateway logs -f
openclaw doctor   # 自动检测配置问题
```

---

## 10 进阶：生产部署建议

1. **Use systemd service**：`openclaw install-service` 会生成 systemd unit，支持自动重启、资源限制。
2. **Firewall**：仅暴露必要端口（如 3000），或用反向代理（nginx）加 Basic Auth。
3. **TLS**：建议通过 nginx/Traefik 提供 HTTPS。
4. **Backup**：定期备份 `~/.openclaw/` 目录，包含配置、技能、日志。
5. **Monitoring**：用 `openclaw gateway status` 或 systemd `journalctl -u openclaw-gateway` 监控。

---

## 11 总结

本文覆盖了在 Linux 服务器上完整安装、配置、运行 OpenClaw 的流程。无论你是个人想智能化管理服务器，还是团队需要统一 AI 工作流，OpenClaw 都提供了一个灵活、可扩展的基础。

下一步可以：

- 探索 [文档](https://docs.openclaw.ai) 了解更多技能
- 在 [ClawHub](https://clawhub.com) 搜索社区技能
- 查看 `openclaw --help` 或 `openclaw gateway --help` 学习 CLI

祝你部署顺利，AI 玩得开心！🚀
