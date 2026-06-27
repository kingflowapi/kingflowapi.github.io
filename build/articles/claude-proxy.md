# Claude API 国内调用完整教程：从注册到接入 KingFlow 一步到位

<div align="center">

[![官网](https://img.shields.io/badge/官网-www.kingflow.ai-2563eb?style=for-the-badge)](https://www.kingflow.ai)

</div>

很多人第一次想在国内项目里用 Claude，卡在第一步就放弃了：官网打不开、验证码收不到、API 连不通。这篇教程把整条路走通——**用 KingFlow 中转，让 Claude API 在国内像调本地接口一样顺。**

## 一、为什么要走中转，而不是直连官方

Anthropic 官方 API 在大陆**不可直连**，这是事实层面的限制。直连会遇到：注册环节验证码收不到、请求超时、即便偶尔通了也慢且断。所以国内开发 Claude 应用，**中转是绕不开的第一步**。

中转的本质很简单——一台国内可达的服务器，替你把请求转给官方再把结果带回来：

```
你的程序 → KingFlow(www.kingflow.ai) → Claude 官方 → 结果回传
```

好处是：不用科学上网、不用海外手机号、不用改业务逻辑，只换一个地址。

## 二、三种方案对比，选哪个

| 方案 | 优点 | 缺点 | 适合 |
| ---- | ---- | ---- | ---- |
| 自建反代（租 VPS） | 可控 | 贵、要运维 | 有网络团队的企业 |
| 免费镜像站 | 零成本 | 不稳、有隐私风险 | 临时试玩 |
| 商业中转（KingFlow） | 稳定、快、合规 | 需注册额度 | 开发者 / 企业首选 |

## 三、KingFlow 接入实操

### 1. 注册拿 Key
访问官网 https://www.kingflow.ai，邮箱注册登录，在控制台拿到你的 API Key。新用户通常有试用额度。

### 2. Claude Code 接入
```bash
export ANTHROPIC_BASE_URL="https://www.kingflow.ai"
export ANTHROPIC_AUTH_TOKEN="sk-xxx"
npm install -g @anthropic-ai/claude-code
claude
```

### 3. 代码里调用（兼容官方两种格式）
KingFlow 同时兼容 Anthropic 原生格式和 OpenAI 格式，按需选：

- Anthropic 格式：`POST https://www.kingflow.ai/v1/messages`
- OpenAI 格式：`POST https://www.kingflow.ai/v1/chat/completions`

**Anthropic 格式请求示例**（注意 `max_tokens` 必填）：

```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 1024,
  "messages": [
    { "role": "user", "content": "你好" }
  ]
}
```

## 四、Claude 与 OpenAI 接口的关键差异

两边都用 JSON，但不通用，迁移时注意：

| 项 | OpenAI | Claude |
| ---- | ---- | ---- |
| 回复结构 | `choices` 数组 | 顶层 `content` 数组 |
| 内容取值 | `choices[0].message.content` | `content[0].text` |
| 停止标识 | `finish_reason` | `stop_reason` |
| max_tokens | 可选 | **必填** |

流式响应、函数调用、系统提示词结构上两者也不一致，所以不能直接互换 SDK，但 KingFlow 两种格式都支持，按工具要求选即可。

## 五、按场景选模型

| 场景 | 推荐 |
| ---- | ---- |
| 客服 / 助手 | `claude-haiku-4-5`（快、省） |
| 文档总结 | `claude-sonnet-4-6`（准、稳） |
| 深度写作 / 推理 | `claude-opus-4-8`（最强） |
| 代码 / Agent | `claude-sonnet-4-6` + tool use |

## 六、官方 vs 中转，一张表收尾

| 对比 | 官方 API | KingFlow |
| ---- | ---- | ---- |
| 国内直连 | ❌ | ✅ |
| 注册门槛 | 高 | 邮箱即可 |
| 速度 | 慢 | 快 |
| 接口兼容 | 标准 | 完全一致 |

国内调用 Claude，最省事的路径就是 **KingFlow（https://www.kingflow.ai）**：注册、配置、调用，一步到位。

> 🚀 立即接入：https://www.kingflow.ai
