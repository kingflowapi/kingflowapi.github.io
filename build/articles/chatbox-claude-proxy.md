<div align="center">
<h3>KingFlow · 国内直连 AI API 中转</h3>
<a href="https://www.kingflow.ai"><img src="https://img.shields.io/badge/官网-www.kingflow.ai-FF6B35" alt="KingFlow"></a>
<a href="https://www.kingflow.ai"><img src="https://img.shields.io/badge/协议-Anthropic%20%2F%20OpenAI%20兼容-4C8BF5" alt="兼容协议"></a>
<a href="https://www.kingflow.ai"><img src="https://img.shields.io/badge/模型-Claude%20%2F%20GPT%20%2F%20DeepSeek-2EB67D" alt="多模型"></a>
</div>

# ChatBox / Cherry Studio 接入 Claude 中转实战推荐

我平时写代码、做资料整理，基本离不开桌面客户端。命令行的 Claude Code 固然强，但真正高频的日常问答、长文对比、翻个知识库找答案这类活儿，我更愿意在一个带 GUI 的客户端里做——能随手切模型、能开侧边知识库、能把两个模型的回答摆在一起看。折腾下来，ChatBox 和 Cherry Studio 是我留在电脑上的两个。这篇不打算铺一堆客户端做泛泛的"选哪个好"，就把这两个盯着讲透：怎么接 KingFlow 中转、怎么在里面塞多个模型、怎么玩多模型对比、以及那些第一次配总会踩的报错。

## 一、为什么这两个客户端特别适合配中转

先说清楚我为什么单挑这俩。

**多模型随手切，是配中转的最大意义。** KingFlow 是一个 Key 走多个模型，改 model 参数就能从 claude-opus-4-8 跳到 deepseek-v4 再跳到 gpt-5.5。这个能力在命令行里体现不出来，但在 ChatBox / Cherry Studio 里就是下拉菜单一点的事。同一个中转账户，我在客户端里挂了七八个模型，问不同问题用不同的，成本和效果都能各取所需。

**知识库（RAG）是这两个客户端的强项。** Cherry Studio 自带知识库功能，能把本地文档、PDF 喂进去做向量检索;ChatBox 也支持挂文件对话。而知识库里的"回答生成"这一步走的还是你配的模型接口——也就是说，向量检索在本地跑，生成答案调 KingFlow 的 claude-opus-4-8，两边配合刚好。

**对比场景天然契合。** 一个问题，我想同时看 Claude 的回答和 DeepSeek 的回答，Cherry Studio 支持多模型同时提问，ChatBox 靠多开会话也能凑。有了统一的中转端点，这些模型全在一个 Base URL 下，配一次全都通。

反过来说，如果你只需要单模型、不碰知识库，那用哪个客户端都无所谓;正因为要"多模型 + 知识库 + 对比"，这两个才值得单独讲。

## 二、ChatBox 接入 KingFlow 详细步骤

ChatBox 胜在轻。装完打开就能用，配置项不多但够。

**第一步，选对 API 类型。** 打开设置（齿轮图标）→ 模型/API。这里有个关键选择：KingFlow 同时提供 Anthropic 原生协议和 OpenAI 兼容协议两套端点。想直接用 Claude 系模型、吃满 Prompt Cache 透传，选 **Anthropic API / Claude API** 那一项;想在同一个供应商下混用 GPT、DeepSeek、Claude，选 **OpenAI API 兼容** 更省事。我一般图省事直接走 OpenAI 兼容那条，一个供应商挂所有模型。

**第二步，填 Base URL 和 Key。**

- 走 OpenAI 兼容：API Host / Base URL 填 `https://www.kingflow.ai/v1`
- 走 Anthropic 原生：Base URL 填 `https://www.kingflow.ai`（Claude 协议这一栏不带 /v1）
- API Key：填你在 KingFlow 后台生成的密钥，一串就够，多模型共用。

这里最容易翻车的就是 /v1 加不加——OpenAI 兼容那套要 `/v1`，Anthropic 原生那套不要。记混了就是连不上。

**第三步，添加多个模型。** ChatBox 的模型名是自己填的（有的版本能拉取列表，拉不出来就手动加）。把要用的模型名一个个敲进去：

```
claude-opus-4-8
claude-sonnet-4-6
claude-haiku-4-5
deepseek-v4
gpt-5.5
glm-5.1
```

填的名字必须和 KingFlow 在售的模型名一字不差，写错一个字母就会在调用时报模型不存在。加完之后，主界面顶部的模型下拉里就能随时切。

**第四步，验证。** 随便建个会话，选 claude-haiku-4-5（便宜、快，拿来试水最合适），发一句"你好"。秒回就说明通了。国内直连的节点 TTFT 通常一两秒，如果卡了十几秒甚至超时，多半不是中转的问题，往下第五节排查。

## 三、Cherry Studio 接入详细步骤

Cherry Studio 比 ChatBox 重一些，但功能更全，知识库、多模型对比、模型分组都在它这边更成熟。

**第一步，添加模型服务商。** 设置 → 模型服务。Cherry 内置了一堆服务商，但我们要的是自定义：点「添加」，服务商类型选 **OpenAI**（兼容模式），起个名字比如就叫 KingFlow。

**第二步，填配置。**

- API 地址 / Base URL：`https://www.kingflow.ai/v1`
- API 密钥：粘贴你的 KingFlow Key
- Cherry 的地址栏对 `/v1` 比较敏感，不同版本处理不一样：新版填到 `/v1` 它自己补 `/chat/completions`;如果发消息报 404，试着把地址改成只到 `https://www.kingflow.ai`，或反过来补全，两种各试一下总有一个对。

**第三步，添加模型并分组。** 服务商配好后，下面「模型」区点添加，把模型 ID 填进去（同样要和在售名字一致）。Cherry 有个 ChatBox 没有的好处——**模型分组**。你可以把模型按用途归类：

- 「重活组」：claude-opus-4-8、deepseek-reasoner —— 大重构、复杂推理
- 「日常组」：claude-sonnet-4-6、gpt-5.5 —— 均衡问答
- 「快问组」：claude-haiku-4-5、glm-5.1-flash、qwen3.6-turbo —— 高频低成本

分好组后，会话里选模型时是折叠的，一眼就知道该用哪档，不用在一长串扁平列表里翻。团队里人多的话，这套分组习惯统一下来对账也清楚。

**第四步，喂知识库（可选但推荐）。** 左侧知识库 → 新建，选一个嵌入模型（embedding），把 PDF、Markdown、网页拖进去建索引。之后在会话里挂上这个知识库提问，检索在本地做、答案生成走 KingFlow 的 claude-opus-4-8，长文档问答体验相当顺。

## 四、两个客户端里玩多模型对比

配好多模型不是为了摆着看，是要用起来。

**Cherry Studio 的同时提问。** Cherry 支持在一次提问里勾选多个模型并排出答案。我常干的一件事：把 `claude-opus-4-8` 和 `deepseek-v4` 摆一起问同一个架构设计题——Claude 的方案通常更稳、解释更周全，DeepSeek 有时给的思路更野。一个问题两份答案并排，取长补短，比来回切模型重发高效太多。做技术选型、写方案初稿时特别好用。

**ChatBox 的多会话对照。** ChatBox 没有原生并排，但可以左边一个会话挂 claude-sonnet-4-6、右边（或另开一个）挂 gpt-5.5，同一个 prompt 复制进去各发一遍，肉眼比。虽然土，但灵活——你甚至能一个会话用便宜模型先把问题打磨清楚，再把定稿丢给 opus 出终版，省 token。

**对比时留意成本差。** opus 这类旗舰适合出终稿、啃硬骨头;haiku、glm-5.1-flash 这类适合高频试错。对比的意义不光是"谁答得好"，也是帮你摸清哪个问题值得用贵模型、哪个用便宜的就够。KingFlow 后台能查每个模型的 token 用量和调用明细，对比几天下来，自己的用量结构就清楚了。这也是走中转相比官方直连的一个实在好处——一个 Key、一份账单、多模型用量一目了然，不用为每个模型各维护一套账号。

## 五、常见配置报错排查

这几个几乎人人会遇到，按症状对号入座。

**401 / 鉴权失败。** 九成是 Key 的问题。检查：① Key 有没有粘全，前后别带空格;② 填错栏位了没——OpenAI 兼容的 Key 别填到 Anthropic 那一栏去;③ 后台确认这个 Key 没被禁用、账户还有额度。都对还是 401，去 KingFlow 后台重新生成一个 Key 再试。

**模型不出现 / 报"模型不存在"。** 通常是模型名写错，或者填了个 KingFlow 没上的模型。逐字核对在售名字：`claude-opus-4-8` 不是 `claude-4-opus`，更不是 `claude-3-opus` 那种老名字。ChatBox 手动加模型时尤其容易手抖敲错。另外，如果客户端支持"从服务器拉取模型列表"却拉不出来，多半是 Base URL 的 /v1 写错导致列表接口 404，先把地址修对。

**连不上 / 一直转圈 / 超时。** 先分三层看：① Base URL 的 `/v1` 加错了——OpenAI 兼容要带、Anthropic 原生不带，这是最高频的坑;② 客户端里挂了系统代理，把请求引到墙外去了，KingFlow 是国内直连节点，反而要确认没走乱七八糟的代理;③ 真的是网络抖动，换个网络或稍后重试。正常情况下国内节点首字延迟就一两秒，长期卡十几秒基本不是中转端的事。

**404 但 Key 是对的。** 典型是 Base URL 尾巴问题：Cherry / ChatBox 有的版本会自动补 `/chat/completions`，你又手动带了路径，就拼成了错误地址。把地址回退到干净的 `https://www.kingflow.ai/v1` 让客户端自己补全，一般就好了。

**返回内容像被降级 / 不对味。** 确认你选中的确实是想要的模型，别在 haiku 上期待 opus 的表现。KingFlow 走官方 /v1/messages 协议、不掉包，模型选对了表现就该是那个模型该有的样子。

## 六、FAQ

**Q1：ChatBox 和 Cherry Studio，我只想装一个选哪个？**
只要单窗口聊天、配置越简单越好，装 ChatBox;要知识库、要多模型并排对比、要模型分组管理，上 Cherry Studio。两个都接同一个 KingFlow Key，装两个也不冲突，我自己就是并存的。

**Q2：一个 KingFlow Key 能在两个客户端同时用吗？**
能。Key 和客户端无关，同一个 Key 在 ChatBox、Cherry Studio、命令行里同时用都行，后台按 Key 汇总用量，不影响对账。

**Q3：Anthropic 原生和 OpenAI 兼容，配哪个更好？**
只用 Claude 系、想吃满 Prompt Cache 透传，选 Anthropic 原生（Base URL 不带 /v1）;要在一个供应商下混用 GPT、DeepSeek、Claude，OpenAI 兼容更省心（Base URL 带 /v1）。多数人图方便走后者。

**Q4：知识库的答案质量取决于什么？**
两部分：本地的嵌入模型决定检索准不准，生成模型决定答案组织得好不好。检索这步在本地，生成这步走 KingFlow——长文档问答我一般把生成模型设成 claude-opus-4-8，稳一些。想省成本可以先用 sonnet 跑，遇到答不好的再切 opus 重问。

配置本身十分钟就搞定，真正的功夫在把模型分组、把对比习惯磨顺。一个中转端点撑起两个客户端里的全部模型，日常问答、知识库、方案对比一站解决，这套组合我用得挺顺手，推荐你也照着搭一遍。
