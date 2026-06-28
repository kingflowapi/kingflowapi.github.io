<div align="center">

### KingFlow · 国内直连 AI API 中转

<a href="https://www.kingflow.ai"><img src="https://img.shields.io/badge/官网-www.kingflow.ai-FF6B35" alt="KingFlow"></a>
<a href="https://www.kingflow.ai/v1"><img src="https://img.shields.io/badge/端点-/v1-2EA44F" alt="Endpoint"></a>
<a href="https://www.kingflow.ai"><img src="https://img.shields.io/badge/倍率-后台可查-1E90FF" alt="Rate"></a>

</div>

# Cursor + Claude Code 中转接入：后台可查倍率的省钱方案

我现在的日常开发是两件套：编辑器里用 **Cursor** 写代码、补全、改文件，命令行里用 **Claude Code** 跑长任务——批量重构、读整个仓库、写测试、按 issue 落地改动。两个工具我都离不开，但它们有个共同的痛点：直接接官方 API，钱烧得又快又算不清。

这篇就讲讲我怎么把这两个工具一起挂到 KingFlow 上，配置各放各的，账单一处看清。

## 一、开发者的真实日常：两个工具一起烧 token

Cursor 适合"人在驾驶座"的活儿——你盯着代码,它边写边补，交互密集、调用频繁。Claude Code 适合"交给它自己干"的活儿——给个目标，它自己读文件、规划、改一堆地方，单次任务吃的 token 量级完全不同。

两个工具叠在一起，问题就来了：

- 官方计费按原始 token 走，单价不便宜，重度用一天的量很可观。
- Cursor 的额度和 Claude Code 的账单是两套体系，你很难把"今天到底花了多少"加在一起看。
- 月底来一张账单，哪个工具花得多、哪个模型贵，全靠猜。

我要的不是"最便宜"，而是**花得明白**：每一笔调用走的什么模型、什么倍率、扣了多少、余额还剩多少，能查。

## 二、KingFlow 的核心价值：倍率透明 + 后台可查

KingFlow 是一个 AI API 中转，对我最有用的是两点。

**第一是倍率透明。** GPT 系从 0.1 倍率起，Claude 系 0.6 倍率，具体每个模型对应的倍率在后台都列得清清楚楚。倍率乘以官方原价就是我实际付的钱，不藏在套餐里、不打包成"积分"，看一眼就知道这个模型贵不贵。

**第二是后台什么都能查。** 调用日志（哪个 key、哪个模型、什么时候、花多少 token）、实时余额、按模型/按时间的用量统计，全在控制台。Cursor 和 Claude Code 共用同一个 key 体系，两个工具的消耗最终汇总到一处——这才解决了我"加不起来"的烦恼。

| 对比项 | 官方 API / 各自接入 | KingFlow 中转 |
| --- | --- | --- |
| 单价 | 原价 | 倍率计费，GPT 0.1 起 / Claude 0.6 |
| 倍率可见 | 隐含在定价里 | 后台逐模型列出 |
| 用量统计 | 两个工具各看各的 | Cursor + Claude Code 汇总一处 |
| 余额/日志 | 颗粒度有限 | 实时余额 + 逐条调用日志 |
| 接入成本 | 各配各的官方密钥 | 改 Base URL + 换 key |

## 三、配置 Cursor

Cursor 走 OpenAI 兼容协议，在设置里覆盖端点即可。

打开 **Settings → Models**，找到 OpenAI API Key 区域：

1. 勾选 **Override OpenAI Base URL**，填：

   ```
   https://www.kingflow.ai/v1
   ```

2. **API Key** 填你在 KingFlow 后台生成的密钥。
3. 在模型列表里填上要用的模型名（如 `claude-sonnet-4-6`、`gpt-5.5`），点 Verify 验证通过即可。

验证成功后，Cursor 的对话、补全、Edit 都会走 KingFlow，这些调用会一条条出现在后台日志里。

## 四、配置 Claude Code

Claude Code 走 Anthropic 协议，靠两个环境变量切换上游。把下面两行写进 shell 配置（`~/.zshrc` 或 `~/.bashrc`）：

```bash
export ANTHROPIC_BASE_URL="https://www.kingflow.ai/v1"
export ANTHROPIC_AUTH_TOKEN="你的-KingFlow-Key"
```

`source` 一下让它生效，然后正常运行 `claude` 即可。想验证一下接没接对：

```bash
curl https://www.kingflow.ai/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: 你的-KingFlow-Key" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-haiku-4-5",
    "max_tokens": 64,
    "messages": [{"role": "user", "content": "ping"}]
  }'
```

能正常返回，说明 Claude Code 这边也通了。两个工具同一个 key、同一个端点，后台自然就并到一起算了。

## 五、两个工具怎么分工、怎么选模型

挂上之后，省钱的关键其实在"哪个活儿用哪个模型"。我的习惯是：

- **Cursor 日常编码 / 补全 / 小改**：用 `claude-haiku-4-5`。交互频繁、单次任务小，便宜又快，倍率低，量大也不肉疼。
- **Cursor 处理复杂逻辑 / 重要改动**：临时切到 `claude-sonnet-4-6`，质量更稳，改完再切回 haiku。
- **Claude Code 跑大任务**：默认 `claude-sonnet-4-6`，读多文件、做规划、写测试，均衡档够用且性价比好；只有真正难啃的架构级活儿才偶尔上更高档的 `claude-opus-4-8`。
- **批量、机械、确定性高的脚本任务**：直接 `claude-haiku-4-5`，能省一大截。

因为倍率在后台看得见，我可以非常直观地判断"这个活儿值不值得上贵模型"，而不是凭感觉乱用。

## 六、先测后充，留好备用方案

两条务实建议：

**先测后充。** 注册一般会送一点测试额度，别急着充钱。先拿你自己真实的任务跑几遍——让 Cursor 补几轮代码，让 Claude Code 做一个真实的重构，看速度、看返回质量、看后台扣的倍率合不合理，都满意了再充。

**生产留备用。** 中转再稳也是多一跳。如果是吃饭的生产业务，配置上做成可切换——环境变量抽出来、Base URL 参数化，必要时一行切回官方或别的通道，别把单点风险全压在一个端点上。

## FAQ

**Q1：Cursor 和 Claude Code 必须用同一个 key 吗？**
不必须，但建议用同一个 KingFlow key。这样两个工具的消耗会汇总到同一份后台统计里，余额、日志、用量看起来最清楚。

**Q2：倍率到底在哪看？**
登录 KingFlow 后台，模型列表里每个模型都标了对应倍率，GPT 系 0.1 起、Claude 系 0.6。实际花费 = 倍率 × 官方原价 × token 量，调用日志里也能逐条对账。

**Q3：Cursor 里 Verify 不过怎么办？**
先确认 Base URL 是 `https://www.kingflow.ai/v1`（带 `/v1`，别漏），key 没填错没带空格，模型名是后台支持的当前在售型号（如 `claude-sonnet-4-6`）。再不行就用第四节的 cURL 单测一下，定位是 key 问题还是模型名问题。

**Q4：Claude Code 设了环境变量还是走官方？**
多半是变量没生效或被别处覆盖。确认写进了正在用的 shell 配置文件、`source` 过、或干脆新开一个终端窗口；用 `echo $ANTHROPIC_BASE_URL` 看看值对不对。

---

一句话总结：Cursor 管交互写码、Claude Code 管自动跑任务，两个都改成 `https://www.kingflow.ai/v1` 这一个端点，倍率后台看得见、用量余额日志查得到——这才是我愿意长期用下去的省钱方式。官网 https://www.kingflow.ai 。
