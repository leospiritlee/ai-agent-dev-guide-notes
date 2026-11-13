# 第7章 上下文工程：token 是预算

**一句话**：`context engineering` 管的是「推理时该放哪些 token 进去」这整个集合——`system instructions`、工具定义、`tool_result`、`message history`、`MCP`、`external data` 全在其中；因为 `context rot` 存在，目标是找到最小的 `high-signal tokens` 集合。

**为什么需要它**：官方把这门手艺的转向说得很清楚：在 `prompt engineering` 成为焦点几年之后，`context engineering` 走到台前；用语言模型构建应用，越来越不是找对提示词里的措辞，而是回答「什么样的上下文配置最可能产生我们想要的行为」。定义也给了：`Context` 是采样时包含的那组 token，工程问题是针对 LLM 的固有限制优化这些 token 的效用；`Context engineering` 是在推理期间策划与维护最优 token 集合的策略，包括所有落在提示词之外的信息。与写提示词这种离散任务不同，它是迭代的——每次决定传给模型什么，都要重新策划一遍。对 Agent 尤其如此：一个在循环里运行的 Agent 不断产生「下一轮可能相关」的数据，这些信息必须被周期性地精炼。

**核心结构**：`context engineering` 被官方视作 `prompt engineering` 的自然演进，但管的范围大得多：`system instructions`、`tools`、`MCP`、`external data`、`message history` 等整个上下文状态。为什么必须当成预算？针尖寻草堆式基准研究揭示了 `context rot`：上下文窗口里 token 数增加，模型准确回忆其中信息的能力下降。因此上下文必须被视为 `finite resource with diminishing marginal returns`——像人类工作记忆有限一样，LLM 有一份解析大量上下文时支取的 `attention budget`，每引入一个新 token 都会消耗它一点；n 个 token 产生 `n² pairwise relationships`，上下文变长时模型捕捉这些成对关系的能力被摊薄，形成上下文长度与注意力聚焦之间的天然张力。结果是 `performance gradient rather than a hard cliff`：不是到某个长度突然崩，而是越塞越钝。结论因此是一句可操作的话：好的上下文工程 = 找到**尽可能小**的 `high-signal tokens` 集合，让期望结果的概率最大。

**关键设计**：**`system prompts` 要停在 `right altitude`**。官方把 `right altitude` 定义为两种常见失败模式之间的 `Goldilocks zone`，并建议把提示词组织成明确的分区（如 `<background_information>`、`<instructions>`、`## Tool guidance`、`## Output description`），用 `XML tagging` 或 `Markdown headers` 分隔；无论怎么组织，都应追求「能完整勾勒期望行为的最小信息集」——minimal 不等于 short，仍要给足让 Agent 遵守行为的信息。`few-shot prompting` 仍是强烈建议的最佳实践，但官方明确不推荐把一长串边界情况塞进提示词去穷举每条规则，而应策划一组 `diverse, canonical examples` 有效展示期望行为，「对 LLM 来说，例子是值一千个词的图片」。另一条主线是 `just-in-time`：让 Agent 按需探索环境取信息，而不是预先把所有可能有用的东西都装进来；官方把 `informative, yet tight` 与这套策略一起，归到「最小高信号 token 集」这条总原则下。

**拆解**：

### 上下文里到底装了什么

按官方列举逐项盘点，很容易发现提示词只是很小一部分：`system instructions` 是常驻的；`tools` 的定义（名称、描述、schema）每轮都在；`tool_result` 会随着循环不断追加；`message history` 越滚越长；通过 `MCP` 拿到的 `external data` 也要落在这里。第 5 章算过 `tools` 与 `tool_result` 的 token 账，第 6 章用 `truncation` 与 `response_format` 压过返回体积，这一章补的是全局视角：这些东西共享同一个 `attention budget`，任何一处膨胀都会挤掉别处的有效信息。所以「模型变笨了」往往不是模型变了，是上下文配置变了。

### 为什么是渐变而不是断崖

`performance gradient rather than a hard cliff` 这句话有个很实际的后果：靠「看有没有报错」发现不了上下文问题。没有溢出异常、没有截断警告，只是召回率一点点下降、工具选择一点点变飘、指令遵守一点点松动。综合报告口径把它和 `10 步` / `59%` 那条算术放在同一层理解：两者都是渐进退化，都只能靠评测集和基线发现。这也解释了为什么第 11 章坚持每次 prompt / 模型 / 工具变更都跑评测门——渐变型退化不会自己报警。

### 提示词分区与例子的正确用法

`XML tagging` 与 `Markdown headers` 的价值不在美观，而在让模型能定位到某类指令；把 `<instructions>`、`## Tool guidance`、`## Output description` 分开之后，改一处不至于牵动全局，也便于做差异评测。`diverse, canonical examples` 则是对「穷举边界情况」的否定：与其写二十条「如果遇到 X 就不要 Y」，不如给三五个覆盖典型行为的规范示例。官方那句「例子是图片」背后的机制是 `few-shot prompting` 提供的是行为示范而非规则解释器。第 2 章把提示词当接口版本化，这一章给的是这个接口的内容规范。

### just-in-time 与第 8、9 章的接口

官方把 `just-in-time` 与 `compaction`、token 高效的工具并列，说它们的指导原则完全一致：找最小的 `high-signal tokens` 集合。这条原则往三个方向长：往长任务走，是第 8 章的 `compaction`、`structured note-taking` 与 `sub-agent architectures`，解决「上下文装不下但状态不能丢」；往外部知识走，是第 9 章的 `RAG` 与 `hybrid retrieval`，解决「事实太多且会变，不能预先塞满」；往工具走，是第 6 章的 `minimal viable set of tools` 与 `25,000 tokens` 上限，解决「定义常驻」。三者是同一份预算的三个出口，分开学容易，合起来管才是上下文工程。

**今天怎么做**：把一次真实运行的完整请求 dump 出来，按 `system instructions` / `tools` / `message history` / `tool_result` / `external data` 五类统计 token 占比；先砍占比最大且最旧的那一类，再看评测分有没有掉。**延伸，非语料原文**：给这份占比表设一个基线数值写进仓库，之后每次改动只对比一个数；上下文问题几乎从不以异常形式出现，有个基线比事后凭感觉判断可靠得多。

<div class="chapend" markdown="1">

**容易误解的地方**：把 `context engineering` 当成 `prompt engineering` 换个名字——后者管指令的写法，前者管整个 token 集合；以为上下文窗口没满就安全——`context rot` 是渐变；把「minimal」理解成「越短越好」——官方特意注明 minimal 不必然等于 short；还有把一长串边界情况塞进提示词当严谨，实际官方明确不推荐这种做法。

**记住这一句**：上下文是 `finite resource with diminishing marginal returns`；每次传给模型什么都要重新策划，目标是最小的高信号 token 集合，而不是塞满窗口。

</div>
