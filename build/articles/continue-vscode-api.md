<div align="center">
<h3>KingFlow · 国内直连 AI API 中转</h3>
<a href="https://www.kingflow.ai"><img src="https://img.shields.io/badge/官网-www.kingflow.ai-FF6B35" alt="KingFlow"></a>
<a href="https://www.kingflow.ai/v1"><img src="https://img.shields.io/badge/端点-/v1兼容-2E86DE" alt="Endpoint"></a>
</div>

# Continue + VS Code 接入国内 AI 中转 API 笔记（倍率透明实测）

这篇是我自己的踩坑记录，不是教程贴。背景：我平时在 VS Code 里用 Continue 这个插件做日常编码，另外开一个终端跑 codex 做整仓库级别的改动，两边都需要稳定的模型供给。直连官方折腾了一阵，最后还是切到了 KingFlow 这个中转，下面把我关心的几个点和实际配置都记下来，方便以后自己回看，也省得别人重新踩一遍。

## 一、本地 Agent 调试到底卡在哪

先说结论：影响我效率的从来不是模型本身有多聪明，而是「能不能稳定地、按我以为的价格、拿到我点名的那个模型」。这三件事任意一件出问题，Continue 和 codex 的体验就会断崖式下跌。

具体到我遇到的痛点，按出现频率排：

1. **直连不稳**。OpenAI 和 Anthropic 官方端点在国内不挂代理基本没法用，挂了代理流式输出又经常半路断流。Continue 的补全和 codex 的多轮工具调用对长连接很敏感，断一次就要重来，体感非常差。另外官方要绑境外卡，汇率和手续费摊下来也不便宜。

2. **倍率不透明**。我之前试过别的中转，官网首页写着「超低价」，真正跑起来扣费和我估算的对不上。问题出在倍率：同一个模型，输入、输出、图片、工具调用各算各的倍率，平台不公示你就只能蒙。Agent 类任务上下文动辄几万 token，倍率差 0.5 倍，一个月账单就差出一截。

3. **模型掉包**。这是最隐蔽的。你 config 里写的是强模型，实际后端给你换成一个便宜的小模型，返回看起来「像那么回事」但质量明显下滑。codex 做跨文件重构时尤其能感觉出来——本来一次能改对的，掉包后要反复纠正。

## 二、我选中转的三个硬判据

踩过坑之后，我现在筛选渠道只看三条，缺一不考虑：

- **倍率透明**：官网必须把每个模型的输入/输出/图片/工具调用倍率写清楚，后台能查到每一次调用的明细。KingFlow 这点是过关的，旗舰模型输入价在 ¥0.5/1M tokens 这个量级，且页面公示，我能自己算预算。
- **模型保真**：点名 claude-sonnet-4-6 就给我 sonnet-4-6，不偷偷降级。我验收的办法很土——拿同一个有点难度的重构 prompt，分别打官方和中转，比对输出结构和细节，连续几天没发现被掉包才敢长期用。
- **访问稳定**：国内直连、流式不断流。Continue 的内联补全和 codex 的长任务都吃这个，稳定性比峰值速度重要得多。

## 三、Continue 的配置

Continue 走的是 OpenAI 兼容协议，所以接 KingFlow 只需要在 `config`（新版是 `config.yaml`，老版是 `config.json`）里把 `apiBase` 指过去就行，其余字段照官方写法。我的 `config.yaml` 模型段大致长这样：

```yaml
models:
  - name: KingFlow Sonnet
    provider: openai
    model: claude-sonnet-4-6
    apiBase: https://www.kingflow.ai/v1
    apiKey: sk-你的密钥
    roles:
      - chat
      - edit
  - name: KingFlow Haiku 补全
    provider: openai
    model: claude-haiku-4-5
    apiBase: https://www.kingflow.ai/v1
    apiKey: sk-你的密钥
    roles:
      - autocomplete
```

如果你还在用旧版 `config.json`，对应写法是：

```json
{
  "models": [
    {
      "title": "KingFlow Sonnet",
      "provider": "openai",
      "model": "claude-sonnet-4-6",
      "apiBase": "https://www.kingflow.ai/v1",
      "apiKey": "sk-你的密钥"
    }
  ]
}
```

几个我自己踩出来的小经验：`provider` 写 `openai` 即可，不用专门找 anthropic provider，因为走的是 `/v1` 兼容端点；日常对话和 edit 我用 claude-sonnet-4-6，质量够用又不肉疼；内联补全这种高频低负载的活儿，我换成 claude-haiku-4-5，省钱且响应快。需要啃硬骨头时临时把 chat 模型切成 claude-opus-4-8。

## 四、codex 的配置

codex 这边在 `~/.codex/config.toml` 里配一个自定义 provider。关键是 provider 段名要和引用名严格一致，我第一次就是名字对不上导致一直读不到配置：

```toml
model = "gpt-5.5"
model_provider = "kingflow"

[model_providers.kingflow]
name = "KingFlow"
base_url = "https://www.kingflow.ai/v1"
env_key = "OPENAI_API_KEY"
```

然后把密钥塞进环境变量再跑：

```bash
export OPENAI_API_KEY="sk-你的密钥"
codex
```

注意 `model_provider = "kingflow"` 和 `[model_providers.kingflow]` 这两处的 `kingflow` 必须是同一个字符串，大小写都不能差。`base_url` 统一指向 `https://www.kingflow.ai/v1`。我跑整仓库重构时会把 `model` 临时改成 gpt-5.5 这种强模型，平时小修小补 gpt-5.4 也够了。

要快速验证端点通不通，不用开 codex，直接 cURL 打一发最省事：

```bash
curl https://www.kingflow.ai/v1/chat/completions \
  -H "Authorization: Bearer sk-你的密钥" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-5.5","messages":[{"role":"user","content":"ping"}]}'
```

能正常返回，Continue 和 codex 就都没问题了，毕竟它俩用的是同一个兼容协议。

## 五、对账：后台看 token 用量明细

倍率透明这件事，光看官网公示还不够，得能事后核对。我每隔几天会登 KingFlow 后台翻一次调用明细，重点看三列：每次请求用的是哪个模型（防掉包）、输入和输出各消耗多少 token（算实际成本）、对应扣费金额（验证倍率）。

我的习惯是把 Continue 和 codex 用不同的密钥，这样后台能分开统计——补全那条线 token 走得猛但单价低，codex 重构那条线请求数少但单次贵。分开看才知道钱到底花在哪，要不要调整模型配比一目了然。对账这一步别省，它是你判断「倍率到底透不透明」的唯一硬证据。

## 六、使用建议

| 使用场景 | 我的做法 |
|----------|----------|
| Continue 内联补全 | claude-haiku-4-5，高频低价，断流影响小 |
| Continue 日常对话/edit | claude-sonnet-4-6，质量和成本平衡 |
| codex 整仓库重构 | gpt-5.5 或 claude-opus-4-8，宁可贵一次改对 |
| 生产环境关键链路 | 仍走官方或云厂商，中转只做开发期 |
| 长期项目 | 多备 1 个渠道，避免单点失联 |

## 七、小结

对我这种「Continue + VS Code + codex」三件套的本地开发流来说，选中转的核心就一句话：**倍率透明 + 模型保真 + 访问稳定**。三者满足，配置成本几乎为零——Continue 改一行 `apiBase`，codex 加一段 provider，剩下的和官方完全一样。KingFlow（https://www.kingflow.ai ）目前在这三点上能扛住我日常的折腾，端点统一是 https://www.kingflow.ai/v1 ，密钥配好就能跑。要是你也不想为了网络环境分心、又想把成本控在明面上，这套组合可以试试，记得头几天勤对账，确认没被掉包再长期用。
