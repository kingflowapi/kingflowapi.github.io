# Codex CLI 接入国内中转 API 实战：KingFlow 配置与避坑

<div align="center">

[![官网](https://img.shields.io/badge/官网-www.kingflow.ai-2563eb?style=for-the-badge)](https://www.kingflow.ai)
[![Codex](https://img.shields.io/badge/工具-OpenAI%20Codex-16a34a?style=for-the-badge)](https://www.kingflow.ai)

</div>

OpenAI 的 **Codex CLI** 是个能直接读代码、改文件、跑命令的终端编程助手。但它默认连官方，国内用起来网络是个坎。这篇用最短路径把它接到 **KingFlow** 中转上，并把新手最容易踩的坑一次说清。

## 准备工作

- Node.js **22+**、npm **10+**
- 一个 KingFlow 账户和 API Key（官网 https://www.kingflow.ai 注册获取）

> Windows 用户：官方对 Windows 支持偏实验性，更稳的是 WSL 环境。

## 一、安装 Codex CLI

```bash
# macOS / Linux
npm install -g @openai/codex
codex --version
```

Windows 在 CMD / PowerShell 执行同样的 `npm install -g @openai/codex`；Linux 若报权限错误前面加 `sudo`。

## 二、把 KingFlow 配成 Codex 的接口

Codex 读用户目录下的 `~/.codex/`（Windows 同理）。需要两个文件：

**`~/.codex/auth.json`**（Key 换成你的）：

```json
{"OPENAI_API_KEY": "sk-xxx"}
```

**`~/.codex/config.toml`**：

```toml
model_provider = "kingflow"
model = "gpt-5-codex"
model_reasoning_effort = "high"
disable_response_storage = true
preferred_auth_method = "apikey"

[model_providers.kingflow]
name = "kingflow"
base_url = "https://www.kingflow.ai/v1"
wire_api = "responses"
```

macOS / Linux 快速创建：

```bash
mkdir -p ~/.codex
touch ~/.codex/auth.json ~/.codex/config.toml
```

⚠️ 改完**一定要重启终端**，配置才生效。

## 三、跑起来

```bash
cd your-project-folder
codex
# 或直接带任务：
codex "先扫描这个仓库结构，列出你打算改的文件"
```

交互界面里输入 `/` 打开命令菜单（`/model` 切模型、`/new` 开新会话、`/status` 看状态），输入 `!` 可直接跑终端命令（如 `!git status`）。配完 `.codex` 后，VS Code 装 codex 插件即可共用同一套配置。

## 四、最容易踩的 6 个坑

**1. `codex: command not found`**
npm 全局路径没进 PATH，或没装成功。先 `codex --version` 验证，再重装。

**2. 配了 Key 仍报 401**
确认 `auth.json` 是 `{"OPENAI_API_KEY": "sk-xxx"}`，且**重启了终端**。

**3. 一直超时 / 连不上**
检查 `base_url` 是否完全等于 `https://www.kingflow.ai/v1`；公司/校园网可能要放行域名。

**4. `model not found`**
用 KingFlow 实际可用的模型名，名字不匹配会直接失败。

**5. `config.toml` 不生效**
最常见——`model_provider = "kingflow"` 和 `[model_providers.kingflow]` **段名必须一致**；以及忘了重启终端。

**6. Windows 找不到 `.codex` 文件夹**
资源管理器里打开"显示隐藏的项目"，`.codex` 是隐藏目录。

## 五、升级

```bash
npm i -g @openai/codex@latest
```

## 小结

把 Codex CLI 接到国内中转，核心就两件事：`auth.json` 放 Key、`config.toml` 写对 `base_url` 和一致的 provider 名。用 **KingFlow（https://www.kingflow.ai）** 做这层接口，国内直连、低延迟、按量计费，配好就能让 Codex 在你日常开发里干活。

> 🚀 立即接入：https://www.kingflow.ai
