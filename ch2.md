# 第2章 最小骨架：增强型 LLM 与原生 API

**一句话**：最短的可用起点不是框架，而是一次直连 API 的 `augmented LLM` 调用，配齐 `structured output`、`streaming`、`prompt caching` 与一份写坏过的工具 schema；这一层跑不通，上面任何编排都只是在放大不确定性。

**为什么需要它**：官方明确建议开发者从直接使用 LLM API 开始，理由有两层：很多模式几行代码就能实现，以及不理解框架底层是常见的错误来源。综合报告口径把 `阶段 0`（第 1 周）定义成「基线校准」——不走框架，直接调原生 API，覆盖 `Messages / Chat Completions`、`tool use`、`streaming`、`structured output`、`prompt caching`，并刻意把工具 schema 写坏，看模型怎么胡来。这个「刻意写坏」是整章的关键：不亲手见过一个含糊参数名怎样诱发幻觉参数，后面读 `ACI`、`input_schema`、`strict: true` 时就只会当成配置项，而不是一道防幻觉防线。产出物也很朴素——一个 30 分钟内能写完的单步任务（分类、抽取、转换）。

**核心结构**：最小骨架有四段。第一段是输入契约：`structured output` 优先用约束解码或 JSON schema，而不是靠提示词祈祷格式；综合报告口径把「类型化工具契约」列为 L1 工程底座，用 `Pydantic` / Zod 定义工具 I/O schema，把它当作防幻觉参数的第一道防线，工具返回值必须结构化、可校验。第二段是调用与流：`streaming` 让长输出不必等整段生成，`prompt caching` 把稳定前缀的成本压下来。第三段是增强：`retrieval`、`tools`、`memory` 三类按需加入，官方特别强调要「针对具体用例裁剪这些能力」，并确保它们给 LLM 提供的是简单且文档良好的接口。第四段是失败出口：合法失败要有明确返回，而不是让模型编一个看起来合理的结果。

**关键设计**：**把提示词当接口，而不是当咒语**。综合报告口径说得很具体：像代码一样版本化、测试 prompt，用代码而非 UI 管理；密钥走运行时注入；锁依赖保证可复现；目录按 entry point / domain logic / agent graph / tool wrappers / RAG module / eval module 分层。`OpenAI Agents SDK` 首页给了一个轻量入口：`pip install openai-agents` 与 `from agents import Agent, Runner`，它的 `Function tools` 能把任意 Python 函数变成工具，带 `automatic schema generation` 与 `Pydantic-powered validation`。但同一页也写清了取舍：SDK 默认用 `Responses API` 包装模型调用，当你想自己掌控 loop、工具分发与状态处理，或工作流很短、主要就是返回模型响应时，应该直接用 `Responses API`；想让运行时管理轮次、工具执行、`guardrails`、`handoffs`、`sessions` 时才用 SDK。这条分界，正好是本章与第 10 章的接口。

**拆解**：

### 结构化输出为什么是底座而不是优化

模型输出错误在单步任务里表现为解析失败，在 Agent 里会变成错误的工具参数、错误的分支、乃至一次写操作。综合报告口径把结构化输出和函数调用放在 L2，明确「优先约束解码 / JSON schema，而非靠提示词祈祷格式」。`OpenAI Agents SDK` 的 `output_type` 与 `Pydantic-powered validation` 就是这一层的落地；`Claude Platform Docs` 的 `strict: true` 保证工具调用严格匹配 schema。实践上要连着做三件事：定义 schema、给每个字段写无歧义的名字、让校验失败返回可行动的错误信息——第 6 章会看到官方把「错误信息也要被提示词工程化」写成了明确建议。

### 刻意写坏一次工具 schema

`阶段 0` 那个练习值得单独拆开。把参数从 `user_id` 改回 `user`，把描述从「按城市取当前天气」改成含糊的「取数据」，观察模型传进来的东西怎么漂；再加上一个与其他工具功能重叠的定义，看选择怎么摇摆。官方在工具设计文里给的原因很直白：当工具功能重叠或目的含糊时，Agent 会困惑于该用哪个；参数应当无歧义命名，`user_id` 优于 `user`。亲手制造过这三类失败，后面 `namespacing`、`minimal viable set of tools`、`high signal information` 这些词才不是口号。这也是为什么第 5 章要从一次逐字的协议往返讲起。

### 四类增强各补什么、先加哪个

`augmented LLM` 的三类增强不是并列清单，而是三种不同的缺口：`retrieval` 补模型不知道的事实，`tools` 补模型做不到的动作，`memory` 补跨轮次留不住的状态。官方给的顺序是先优化单次调用、加 `retrieval` 与上下文示例，再考虑复杂度。综合报告口径把它写成模型层面的默认杠杆顺序：Prompt/上下文 → `RAG` → `微调` → `蒸馏`，并给了口诀——事实在变用检索，行为不对先改提示与上下文再考虑 `微调`，要「记住这个人」用记忆层。注意 `微调` 不用来灌知识：事实在权重里会过期，且无法引用来源。第 9 章会把这条顺序展开成判断树。

### 从单步到多步之前，先有评测集

单步任务最好测，也最适合建立第一条基线。综合报告口径把「先写评测集，再写 Agent」列为 2026 年最高杠杆的顺序，并给出起步规模 `30–50 条`（含边缘案例与拒答案例），后续扩到 `50–150 条` 乃至 `100–150 条`。即使这一章只做分类，也该把「什么样的输入算对、什么样的输入应该拒答」写成可复跑的用例。第 11 章的 `pass^k`、`状态断言`、`CI gate` 都建立在这份早期资产上；没有它，第 3 章起的每一次结构调整都无法证明是改进。

**今天怎么做**：本周只做一个单步任务，用原生 SDK 写，不用框架；把输出绑到 schema 上，加 `streaming` 与 `prompt caching`，再故意把工具参数名改含糊跑一遍，记录模型传错的参数。**延伸，非语料原文**：同时写 30 条输入与期望输出存进版本库，哪怕暂时只是手工核对；这份文件后面会变成第 11 章评测门的种子，重写它的成本远高于现在多花的一小时。

<div class="chapend" markdown="1">

**容易误解的地方**：把 `augmented LLM` 当成「还没有 Agent 的半成品」——官方说多数应用优化这一层就够；把 `structured output` 当成格式美化——它是防幻觉参数的第一道防线；以为 `prompt caching` 只是省钱——综合报告口径里它和长上下文一起改变了「要不要搭 `RAG`」的判断；还有一上手就 `pip install openai-agents` 然后不再看底层，恰好踩中官方点名的「对底层做了错误假设」。

**记住这一句**：最小骨架 = 原生 API + `structured output` + 一份写坏过的工具 schema + `30–50 条` 可复跑用例；先把一次调用做对、做可测，再谈让它自己决定下一步。

</div>
