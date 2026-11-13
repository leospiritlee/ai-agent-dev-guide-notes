# 第6章 工具设计：ACI 是首要失败源

**一句话**：工具不是 API 的一层包装，而是给模型用的界面；`ACI` 的设计质量决定 Agent 的上限，`namespacing`、`high signal information`、`token efficiency` 与 `minimal viable set of tools` 是四条最省力的改进杠杆。

**为什么需要它**：官方在 `Building effective agents` 的 ACI 附录里给了一个尺度：想想人在 `HCI` 上投入多少精力，就准备在 `agent`-computer interfaces 上投入同样多。工具设计文把原因说透了：LLM Agent 的「上下文」有限，而计算机内存便宜且充足；一个常见错误，是工具仅仅把已有的软件功能或 API 端点包了一层，不论它是否适合 Agent 使用。`Effective context engineering for AI agents` 补了另一种典型失败：`bloated tool sets` 覆盖太多功能，或制造出「该用哪个工具」的歧义决策点——如果一位人类工程师都不能明确说出某个情境该用哪个工具，就别指望 AI Agent 能做得更好。工具这一层之所以是首要失败源，是因为它同时是模型的动作集合、上下文的常驻负担、以及权限边界的落点。

**核心结构**：官方总结出五条有效工具的做法：选对要实现（以及不实现）的工具；用 `namespacing` 划清功能边界；把有意义的上下文返回给 Agent；为 `token efficiency` 优化工具响应；对 `tool descriptions` 与规格做提示词工程。具体形状上，不要实现 `list_users`、`list_events`、`create_event` 三个工具，而应考虑一个能找空闲时间并直接排期的 `schedule_event`；每个工具要有清晰而独特的用途，功能重叠或目的含糊会让 Agent 困惑。`namespacing` 可以按服务（`asana_search`、`jira_search`）或按资源（`asana_projects_search`、`asana_users_search`）加共同前缀，`MCP` 客户端有时默认就这么做。返回值只给 `high signal information`，优先上下文相关性而非灵活性，避开 `uuid`、`256px_image_url`、`mime_type` 这类低层技术标识；可用 `response_format` 枚举让 Agent 选择 `“concise”` 或 `“detailed”`。对可能吃掉大量上下文的响应，建议组合 `pagination`、范围选择、过滤、`truncation` 并设合理默认值。

**关键设计**：**工具应该 `self-contained`、`robust to error`，且意图极其明确**。这是 `Effective context engineering for AI agents` 的原话，还加了一句类比：与设计良好的代码库里的函数一样。参数命名要无歧义，官方例子是把 `user` 改成 `user_id`；工具调用报错（例如输入校验失败）时，应当把错误信息做提示词工程，清晰传达具体可行动的改进，而不是抛不透明的错误码或 traceback。`25,000 tokens` 是官方给出的 `Claude Code` 工具响应默认上限。最根本的一条是「更多工具不总是带来更好结果」——过多或重叠的工具会把 Agent 从高效策略上带偏；反过来，`curating a minimal viable set of tools` 还能让长交互里的上下文维护与剪枝更可靠。第 5 章算过 token 账，这一章给的是同一笔账的设计解法。

**拆解**：

### 为什么「包一层 API」几乎总是错的

`list_users` + `list_events` + `create_event` 映射到 `schedule_event`，不是合并接口这么简单。三个底层工具要求模型自己编排「查人 → 查空闲 → 算交集 → 创建」，任何一步的参数或时间格式出错都会在下一轮被放大；一个面向工作流的工具把这段确定性逻辑收回代码里，模型只需表达意图。官方的建议是「先构建少数几个针对高影响工作流、经过深思熟虑的工具，匹配你的 `evaluation tasks`，再据此扩展」。这条与第 3 章 `prompt chaining` 的精神一致：能被代码确定的步骤，不要交给概率组件。

### 工具描述的写法与错误信息的写法

官方给的标尺是把工具描述想象成「你会怎么向团队新同事描述这个工具」；因为这些文本会被加载进 Agent 的上下文，它们会共同把 Agent 引向有效的工具调用行为。参数命名同理：`user_id` 明确到不需要猜。错误信息是同一件事的另一半——校验失败时返回「location 需要 City and state 格式，例如 San Francisco, CA」比返回 stack trace 有用得多。综合报告口径把这条写成五条最短建议之一：在工具层和权限层花钱，不要在提示词层省钱；工具描述与错误信息是最被低估的质量杠杆。

### 用评测驱动工具迭代，而不是凭直觉加工具

官方把工具开发定义成 `evaluation-driven` 的迭代过程：先生成大量扎根真实用途的 `evaluation tasks`，每条评测提示都要配一个 `verifiable response or outcome`；除顶层准确率外，还建议采集单次工具调用与任务的运行时长、工具调用总次数、token 总消耗、工具错误数；并用 `held-out test sets` 确保没有对「训练」评测过拟合。`search_logs`、`get_customer_context` 这类工具名在官方示例里出现，就是「面向真实工作流」的具体样子。第 11 章的评测集与 `CI gate` 在这里第一次显出必要性：工具形状改了，得有东西能证明是变好而不是变了。

### 工具数量与上下文预算是同一个问题

`bloated tool sets` 的害处有两层：功能歧义让模型选错，以及所有工具定义常驻每轮请求、持续消耗 `attention budget`。所以 `minimal viable set of tools` 不只是可维护性建议，还是上下文工程的一部分。综合报告口径给 `阶段 2` 的具体动作是「工具数刻意压到 `3–5 个`」。与之配套的是返回侧：`pagination`、`truncation`、`response_format`、`25,000 tokens` 上限都在控制 `tool_result` 回灌的体积；第 8 章的 `tool result clearing` 则处理更久之后的历史。把两端合起来看，工具设计其实是在管一条双向的 token 管道。

**今天怎么做**：把现有工具清单摊开，逐个问「它对应一个真实工作流吗，还是只对应一个 API 端点」；合并功能重叠项，给参数改名，把返回值里的低层标识删掉，再加 `pagination` 与 `truncation` 的默认值。**延伸，非语料原文**：为每轮改动作一组 10 条真实任务做前后对照，只统计「选对工具的比例」和「总 token 消耗」两个数；先别动提示词，这样你能确认改善来自工具形状而不是别处。

<div class="chapend" markdown="1">

**容易误解的地方**：以为工具越细粒度越灵活——`list_users` 那组例子说明细粒度会把确定性编排推给模型；把 `namespacing` 当成命名美观——它降低的是选择歧义；以为 `truncation` 会丢信息而拒绝使用——不截断时撑爆的是 `attention budget`，代价更大；还有把 `held-out test sets` 当成学术讲究，实际上它是防止「照着评测集把工具写死」的唯一办法。

**记住这一句**：工具是给模型用的界面，`ACI` 值得和 `HCI` 同等的投入；少而明确、面向工作流、返回 `high signal information`，比多而全的 API 包装可靠得多。

</div>
