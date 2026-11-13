# 第8章 长任务：压缩、结构化笔记与子代理

**一句话**：当任务跨过 `context window limit`，靠三种手段保住连贯性——`compaction` 压缩对话、`structured note-taking` 把状态写到窗口外、`sub-agent architectures` 用 `clean context windows` 隔离探索；三者补的缺口不同，不能互相替代。

**为什么需要它**：官方对长时程任务的定义是：Agent 要在一串动作上维持连贯性、上下文与目标导向行为，而 token 数超过 LLM 的上下文窗口。问题不止「装不下」，还有 `context pollution`——循环里积累的工具结果与中间消息会把有效信息稀释掉。官方给出的三类对策直接针对这类约束：`compaction`、`structured note-taking` 与 `multi-agent architectures`。这一章是第 7 章总原则的必然延伸：既然目标是最小高信号 token 集合，那么当任务本身比窗口长时，就必须有办法「丢掉一部分、但丢得可恢复」。

**核心结构**：`compaction` 的做法是：对话接近 `context window limit` 时，把内容做摘要，用这份摘要重新开一个上下文窗口；它通常是上下文工程里改善长期连贯性的第一根杠杆，核心是 `high-fidelity manner` 地蒸馏窗口内容，让 Agent 以尽量小的性能损失继续。`Claude Code` 的实现是把 `message history` 交给模型去总结与压缩关键细节，保留架构决策、未解的 bug 与实现细节，丢弃冗余的工具输出或消息。`structured note-taking`（也叫 `agentic memory`）则是 Agent 定期把笔记写到上下文窗口之外的记忆里，之后再拉回来；像 `Claude Code` 建 `to-do list`，或自定义 Agent 维护一个 `NOTES.md`，这个简单模式让 Agent 能跨几十次工具调用跟踪进度、保住关键上下文与依赖；`context resets` 之后，Agent 读自己的笔记继续。`sub-agent architectures` 换了个思路：不让一个 Agent 跨整个项目维持状态，而由专门的子代理用 `clean context windows` 处理聚焦任务；每个子代理可能耗掉数万 token 探索，但只返回 `condensed, distilled summary`（常常 `1,000-2,000 tokens`），于是 `lead agent` 专注于综合与分析，形成清晰的 `separation of concerns`。

**关键设计**：**压缩最难的是「留什么、丢什么」**。官方说 `compaction` 的艺术在取舍：过于激进的压缩会丢掉那些细微但关键、且重要性要到后面才显现的上下文。给工程师的建议是拿复杂的 agent trace 仔细调压缩提示词——先最大化召回率，确保压缩提示抓住 trace 里每一条相关信息，再迭代提升精确率、剔除多余内容。最容易摘的「低垂果实」是清理工具调用与结果：某个工具在消息历史深处被调用过之后，Agent 为什么还要再看一遍原始结果？因此 `tool result clearing` 被官方称为最安全、最轻量的压缩形式。官方给的三条适用判据也要记住：`compaction` 维持需要大量来回对话的任务的会话流；`structured note-taking` 擅长有清晰里程碑的迭代开发；`multi-agent architectures` 处理并行探索确实有收益的复杂研究与分析。

**拆解**：

### 三种手段补的是三种不同的缺口

把三者摊在一张表上看更清楚：`compaction` 解决「同一段对话太长」，代价是可能丢细节，适合来回对话；`structured note-taking` 解决「状态需要在窗口之外活着」，代价是要设计笔记格式与读写时机，适合有里程碑的迭代开发；`sub-agent architectures` 解决「探索本身产生大量低价值中间信息」，代价是编排复杂度与总 token，适合并行研究。综合报告口径的 `阶段 3` 要求同时实现长任务压缩与结构化笔记，并做一次超过上下文窗口的任务、把第 15 步后为什么跑偏解释清楚——这个练习正好逼你分清「跑偏是因为丢了该留的信息」还是「因为笔记根本没被读回来」。

### 压缩提示词为什么必须先召回后精确

官方这条顺序容易被反着做。先写「只保留要点」的压缩提示，看起来干净，但会静默丢掉那些此刻显得无关、二十步之后决定成败的细节——而 `context rot` 是渐变，丢了也不会报错。正确顺序是先在复杂 `trace` 上最大化召回、把相关信息全留住，再逐步剔除冗余提精确率。`tool result clearing` 之所以被称作最安全的轻触式压缩，是因为它丢的是「几乎可以证明不再需要」的那一类内容，而不是靠模型判断重要性。第 11 章的 `状态断言` 与轨迹评测，恰好是检验压缩是否压错了东西的手段。

### 结构化笔记与 agentic memory 的工程形态

`NOTES.md` 与 `to-do list` 是官方给的两个具体例子，重点不在文件格式，而在「写在窗口外、之后再拉回」。这带来三个必须自己回答的问题：谁写（主 Agent 还是单独一步）、什么时候写（每个里程碑还是每 N 轮）、`context resets` 之后谁负责读回来。综合报告口径把 `memory` 分成短期 thread 状态与跨会话长期记忆（《LangGraph》overview 的 `persistence`、`checkpoint` 也落在这层），并提醒「要『记住这个人』→ 记忆层」与「事实在变 → 检索」是两条不同的路。第 9 章会把这个区分做成判断树，避免把记忆、检索、微调混成一团。

### 子代理的收益与代价要一起算

`sub-agent architectures` 的收益是 `clean context windows` 与 `separation of concerns`：详细搜索上下文被隔离在子代理里，`lead agent` 只看 `condensed, distilled summary`。代价是总 token 与编排：综合报告口径提到 orchestrator-worker 在内部评测上比单 Agent 基线高 `90.2%`，但 token 消耗约为单次对话的 `15 倍`；第 3 章也见过同一组数字。所以官方给三条适用判据时特意把 `multi-agent architectures` 限定在「并行探索有收益」的场景，综合报告口径则直说「先让一个 Agent 变可靠，再有理由才加第二个」。第 10 章的 `harness` 视角会再收一次：每加一个组件，都要能回答它补偿的是模型什么缺陷。

**今天怎么做**：挑一个超过窗口的任务，先只加 `tool result clearing`，跑一遍；再补 `compaction`，对比第 15 步之后的行为；最后把笔记写进 `NOTES.md` 并强制一次 `context resets` 后读回。**延伸，非语料原文**：三步各留一份轨迹摘要，只标注「哪一步开始丢关键事实」；这个标注比任何压缩提示词的措辞调整都更能告诉你该改哪里。

<div class="chapend" markdown="1">

**容易误解的地方**：以为 `compaction` 等于「截断旧消息」——它是高保真蒸馏并重建窗口，`truncation` 只是工具响应侧的省 token 手段；把 `agentic memory` 当成 `RAG` 的别名——一个存 Agent 自己的进度，一个取外部事实；以为子代理越多越好——`15 倍` token 与编排复杂度要一起算；还有压缩时先追求简洁，正好颠倒了官方「先召回后精确」的顺序。

**记住这一句**：长任务保连贯靠三件套——`compaction` 压对话、`structured note-taking` 存状态、`sub-agent architectures` 隔探索；先做 `tool result clearing`，再谈更重的机制。

</div>
