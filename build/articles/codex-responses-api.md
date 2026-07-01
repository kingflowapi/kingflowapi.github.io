<div align="center">
<h3>KingFlow · 国内直连 AI API 中转</h3>
<a href="https://www.kingflow.ai"><img src="https://img.shields.io/badge/官网-www.kingflow.ai-FF6B35" alt="KingFlow"></a>
<a href="https://www.kingflow.ai/v1"><img src="https://img.shields.io/badge/端点-%2Fv1%20兼容-2E8B57" alt="Endpoint"></a>
<a href="https://www.kingflow.ai"><img src="https://img.shields.io/badge/协议-Responses%20%2F%20ChatCompletions-1E90FF" alt="Protocol"></a>
</div>

# Codex Responses API 桥接 KingFlow：协议差异与排障清单

如果你已经把桥接服务跑起来、Codex 也指向了本地端口，却还是时不时冒出 `model_not_found`、流式卡半截、或者上游莫名 401，那问题几乎都不在"能不能连通"，而在于 **Responses API 和 Chat Completions 两套协议的形态差异没被桥接层正确抹平**。这篇不讲怎么从零搭桥（那是另一篇的活），只集中做两件事：把两套协议逐字段对照讲清楚，再给一份能照着排的排障 checklist。

上游统一用 KingFlow 的兼容端点 `https://www.kingflow.ai/v1`，模型走在售款（`gpt-5.5` / `gpt-5.4` / `claude-opus-4-8` 经桥接）。

## 一、Responses API vs Chat Completions 差在哪

Codex 新客户端默认走 OpenAI 的 **Responses API**，而绝大多数兼容网关（包括 KingFlow 的 `/v1`）对外提供的是成熟稳定的 **Chat Completions**。两者不是换个路径那么简单，请求体结构、字段命名、流式事件模型、工具调用回传都不一样。先看对照表：

| 维度 | Responses API（Codex 侧发出） | Chat Completions（KingFlow 上游） | 桥接层要做的转换 |
|------|------|------|------|
| 路径 | `POST /v1/responses` | `POST /v1/chat/completions` | 改写 URL 路径 |
| 输入字段 | `input`（字符串或 message 数组） | `messages`（role/content 数组） | `input` → `messages` 结构展开 |
| 系统提示 | `instructions` 顶层字段 | `messages[0]` 的 `role:"system"` | 提取 `instructions` 拼进 messages 头 |
| 输出字段 | `output` / `output_text` | `choices[].message.content` | 反向映射回 `output` |
| 流式协议 | SSE，事件带 `type`（如 `response.output_text.delta`） | SSE，`choices[].delta.content` 增量 | 逐帧重组事件类型与结构 |
| 结束标志 | `response.completed` 事件 | `data: [DONE]` | 转换终止信号 |
| 工具调用 | `tools` + `tool_choice`，结果走 `function_call` 输出项 | `tools` + `tool_calls`（delta 分片） | 拼接分片 → 组装 Responses 格式的函数调用项 |
| 用量统计 | `usage.input_tokens` / `output_tokens` | `usage.prompt_tokens` / `completion_tokens` | 字段改名 |

一句话总结：**同一个语义，两套字段名和两套流式事件模型**。桥接层的价值就是把 Codex 说的"Responses 方言"翻译成 KingFlow 听得懂的"Chat Completions 方言"，再把回包翻译回去。

## 二、桥接层到底要转换什么

具体落到代码里，桥接层至少要处理下面四类转换，缺一类就会在对应场景翻车：

1. **请求体拆装**。把 Responses 的 `input` 和 `instructions` 合并展开成一个标准 `messages` 数组。`instructions` 必须落成 system 消息，否则模型丢失系统设定，表现为"答非所问"。
2. **流式事件重组**。Chat Completions 的 SSE 是一串 `delta.content` 增量，而 Codex 期待的是带 `type` 的结构化事件（`response.created` → 若干 `response.output_text.delta` → `response.completed`）。桥接层要边收上游增量、边生成 Codex 认得的事件序列。这一步最容易出 bug，也是"能回复但流式异常"的根源。
3. **工具调用拼接**。上游把 `tool_calls` 按 SSE 分片吐出来（函数名一帧、参数 JSON 分多帧），桥接层要先在内存里把分片攒完整，再组装成 Responses 的 `function_call` 输出项。半路截断会导致 Codex 拿到非法 JSON 参数直接报错。
4. **用量与错误透传**。`usage` 字段改名要做，同时上游返回的 4xx/5xx 错误体最好原样透传，别吞掉——排障时全靠它定位是本地问题还是上游问题。

## 三、接 KingFlow 的关键配置

桥接层的上游三件套只有三个值要对：**base_url、Key、model**。

`.env` 里指向 KingFlow：

```bash
UPSTREAM_BASE_URL=https://www.kingflow.ai/v1
UPSTREAM_API_KEY=sk-你的KingFlow密钥
UPSTREAM_MODEL=gpt-5.5
PROXY_AUTH_KEY=sk-proxy-local-replace-with-48-char-hex
PORT=4000
```

Codex 侧（`~/.codex/config.toml`）指向本地桥接，注意 `model_provider` 的值必须和下面段名 `[model_providers.kingflow]` **逐字一致**，大小写都不能差：

```toml
model = "gpt-5.5"
model_provider = "kingflow"

[model_providers.kingflow]
name = "KingFlow via bridge"
base_url = "http://127.0.0.1:4000/v1"
env_key = "PROXY_AUTH_KEY"
```

要点三条：Codex 的 `base_url` 指向**本地桥接**（4000），桥接的 `UPSTREAM_BASE_URL` 才指向 **KingFlow**（`https://www.kingflow.ai/v1`）；`env_key` 对应的值要等于桥接的 `PROXY_AUTH_KEY`；`model` 两处都从 KingFlow 后台模型列表复制在售款，别手打。

## 四、排障清单（照着逐条定位）

按"从近到远"的顺序排，能省掉一大半瞎猜：

- **本地请求不到（连接被拒 / 超时）**：桥接进程没起，或 4000 端口被占。`lsof -i:4000` 看谁占着，`curl http://127.0.0.1:4000/v1/models` 看是否有响应。先确认本地这一跳通了再往上游查。
- **本地 401**：Codex 侧的 Key ≠ 桥接的 `PROXY_AUTH_KEY`。这是"本地代理认证"这一层，和 KingFlow 密钥无关。核对 `env_key` 指向的环境变量值和 `.env` 是否一致。
- **上游 401（本地能进、上游拒）**：桥接日志里能看到请求转发出去但上游返回 401，说明 `UPSTREAM_API_KEY` 不对或不完整——多半是复制时漏了字符或带了空格。回 KingFlow 后台密钥页重新整段复制。
- **model_not_found**：模型名拼错或用了已下线款。从 KingFlow 模型列表重新复制，确认是在售款（`gpt-5.5` / `gpt-5.4` / `claude-opus-4-8`）。注意 `config.toml` 的 `model` 和 `.env` 的 `UPSTREAM_MODEL` 要指同一个。
- **能回复但流式异常（吞字 / 卡住 / Codex 界面不刷新）**：桥接层的 SSE 事件重组没做对。检查两点：是否把上游 `delta.content` 转成了 `response.output_text.delta`，以及收到 `[DONE]` 时是否补发了 `response.completed`。缺终止事件，Codex 会一直等。
- **工具调用报参数非法**：`tool_calls` 分片没攒完整就转出去了。在桥接层加日志打印组装后的完整 JSON，确认是合法 JSON 再发。

## 五、验证是否打通

不要直接在 Codex 里试错，先用 curl 打本地桥接的 `/v1/chat/completions`，把桥接和上游这条链路验通，再让 Codex 接手：

```bash
curl http://127.0.0.1:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-proxy-local-replace-with-48-char-hex" \
  -d '{
    "model": "gpt-5.5",
    "messages": [{"role": "user", "content": "只回复 OK"}]
  }'
```

拿到正常 JSON 回复，说明"本地认证 + 桥接转换 + KingFlow 上游"三段全通。想连流式一起验，加 `"stream": true`，观察是否有连续的 `data:` 增量帧和结尾的 `[DONE]`——这一步过了，Codex 的流式基本就稳了。全通之后再回 Codex 发一句话，界面能流式刷新即为打通。

## 六、FAQ

**Q1：为什么 Codex 端非要走桥接，不能直接填 KingFlow 地址？**
因为 Codex 客户端发的是 Responses API 请求（`/v1/responses`、`input` 字段、结构化 SSE 事件），而 KingFlow 提供的是标准 Chat Completions。字段和流式模型对不上，桥接层就是那层翻译器。

**Q2：本地 401 和上游 401 怎么快速区分？**
看桥接日志有没有"请求已转发上游"这条。没有就是本地认证挂了（查 `PROXY_AUTH_KEY`）；有、且上游回 401，就是 `UPSTREAM_API_KEY` 的问题（查 KingFlow 密钥）。

**Q3：流式老是卡最后一下不结束？**
上游发完 `[DONE]` 后，桥接层要主动补一个 `response.completed` 事件给 Codex，否则客户端会一直等待终止信号。

**Q4：模型名从哪里拿最保险？**
一律从 KingFlow 后台的模型列表复制，别凭记忆手打。`config.toml` 与 `.env` 两处保持一致，用在售款 `gpt-5.5` / `gpt-5.4` / `claude-opus-4-8`，过时模型名会直接 `model_not_found`。

---

搞定协议差异这一层，Codex 接 KingFlow（`https://www.kingflow.ai/v1`）剩下的就是把配置抄对。遇到问题按上面的清单从本地往上游逐跳排，基本不用猜。
