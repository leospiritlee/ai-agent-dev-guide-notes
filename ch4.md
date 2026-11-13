# 第4章 Agent 循环：工具调用、反馈与终止条件

**一句话**：`agents` 的运行体通常只是「LLM 在 `environmental feedback in a loop` 里使用工具」，真正需要设计的不是循环本身，而是每一步怎么拿到 `ground truth`、什么时候算 `final output`、以及 `stopping conditions` 怎样把控制权收回来。

**为什么需要它**：官方对生产中的 `agents` 有一段很节制的描述：随着模型在理解复杂输入、推理与规划、可靠使用工具、从错误中恢复这几项能力上成熟，Agent 开始出现；任务一旦明确，Agent 就独立规划与运行，必要时回到人这里补充信息或判断。关键在执行期——每一步都必须从环境拿到 `ground truth`（例如工具调用结果或代码执行结果）来评估自己的进展，可以在检查点或遇到阻碍时暂停等人。任务通常在完成时终止，但也常加入 `stopping conditions`（例如最大迭代次数）以维持控制。这段话说清了一件事：循环不是「让模型一直说下去」，而是「每轮用真实结果校正一次」。缺少真实反馈的循环，只是把同一段推测重复了若干遍。

**核心结构**：`OpenAI Agents SDK` 运行页把 agent loop 写成了可逐条对照的伪代码。第一步，用当前输入调用当前 agent 的 LLM；第二步，LLM 产出输出，然后分三种情况：runner 把输出判为 `final output`，循环结束并返回结果；LLM 请求 `handoff`，更新当前 agent 与输入并重跑循环；LLM 产出 `tool calls`，执行这些调用、把结果追加进上下文并重跑循环。第三步，如果超过传入的 `max_turns`，抛出 `MaxTurnsExceeded`（传 `max_turns=None` 可关闭轮次上限）。判定 `final output` 的规则也很硬：产出符合期望类型的文本输出，且没有 `tool calls`。运行入口有三种：`Runner.run()` 异步返回 `RunResult`，`Runner.run_sync()` 是同步方法、底层仍跑 `.run()`，`Runner.run_streamed()` 异步返回 `RunResultStreaming`；全局设置走 `run_config`。

**关键设计**：**终止条件与循环同等重要，且必须硬编码**。官方说任务常在完成时终止，但「包含 `stopping conditions`（如最大迭代次数）也很常见」；SDK 直接把 `max_turns` 做成异常而非提示词里的建议。综合报告口径把它扩成四个硬上限：最大步数、最大工具调用、最大 token/费用、最大墙钟时间，并强调触顶时升级给人而不是自动重试——重试只会把已经失控的成本再乘一遍。这一层还连着 `guardrails`：SDK 文档里 `InputGuardrailTripwireTriggered`、`OutputGuardrailTripwireTriggered`、`ModelBehaviorError` 以及 `error_handlers`、`output_type`、`RunConfig` 都属于同一套运行时边界。换句话说，一个能上线的循环至少有三道收口：类型化的 `final output`、可编程验证的 `ground truth`、以及不依赖模型自觉的硬上限。

**拆解**：

### 一次循环里哪些东西真的变了

把 SDK 的三分支摊开看，每轮变化的只有三样：当前 agent（因 `handoff` 而变）、输入消息列表（因追加 `tool calls` 结果而变）、以及轮次计数。模型权重没变，提示词模板通常也没变。这解释了为什么工具返回值会在长任务里一路累积成 `context pollution`，也解释了为什么 `tool result clearing` 是安全的轻量 `compaction`——深处消息历史里的原始工具结果，Agent 并不需要再看一遍。第 7、8 章的整套上下文管理，本质上都是在回答「这三样东西里，哪部分可以丢、哪部分必须留」。

### 为什么 ground truth 不能由模型自己生成

官方特意用了 `ground truth` 这个词，并举例「工具调用结果或代码执行」。它的意思是：判断进展的依据必须来自环境，而不是模型对自己上一步的复述。综合报告口径把这条推成了评测原则——任务完成以确定性 `状态断言` 为准，`rubric` 排在精确匹配之后，`LLM-as-judge` 是放大器不是标尺。运行时也一样：让模型自己宣布「已完成」，等于把 `final output` 的判定权交回给它；而 SDK 的规则是「产出期望类型的文本且没有 `tool calls`」，这是代码可判的。第 11 章会看到同一原则在评测集与 `CI gate` 里的翻版。

### 暂停、交还与 human-in-the-loop

官方说 Agent 可以在检查点或遇到阻碍时暂停等人，SDK 把 `Human in the loop` 列为内建机制，`LangGraph` 把它写成 `human-in-the-loop` 与 `checkpoint`、`persistence`。这三处说的是同一件事：中断—审批—恢复需要状态能被持久化，而不只是把消息列表存在内存里。综合报告口径把它放进 `阶段 3`（第 10–14 周），要求实现中断-审批-恢复、失败重放、长任务压缩与结构化笔记，并做一次超过上下文窗口的任务，把第 15 步后为什么跑偏解释清楚。注意顺序：先有 `checkpoint`，才谈得上「暂停之后继续」；否则所谓 human-in-the-loop 只是每次从头再跑一遍。

### 从工作流跨到循环时，最小改动是什么

第 3 章末尾提过，通常第一步是把某个 `gate` 从「失败即退出」改成「失败则把错误回传给模型重试」。这一改就引入了 `environmental feedback in a loop`，但也立刻带来三个新问题：错误回传要占多少上下文（第 7 章）、重试几次必须停（本章 `max_turns`）、以及怎么判断这次重试确实比上次更接近目标（第 11 章 `状态断言`）。所以跨层不是换个框架，而是同时接上预算、上限与可观测。综合报告口径给 `阶段 2` 的产出物很实在：一个可回放、可解释的 Agent，加一份「前 20 次失败的原因分类表」——这张表就是把这三个问题落到纸面上的最低成本做法。

**今天怎么做**：手写一次 agent loop，不用框架，跑通工具调用 → 观察 → 重试 → 终止条件；把 `max_turns` 调成 3 故意触发 `MaxTurnsExceeded`，确认异常路径有日志、有升级出口、不会静默重试。**延伸，非语料原文**：给每轮打印一行「当前 agent / 新增消息数 / 累计 token / 距上限还剩几轮」，这一行日志后面直接长成第 11 章的 `trace`；先让它出现在终端里，比事后补埋点容易得多。

<div class="chapend" markdown="1">

**容易误解的地方**：以为 `final output` 是模型说了算——SDK 的判定是「期望类型的文本且无 `tool calls`」；把 `max_turns` 当成调优参数——它是安全设施，综合报告口径要求触顶升级而非重试；以为加 `handoff` 就等于多 Agent——它只是更新当前 agent 与输入后重跑同一循环；还有把 `ground truth` 读成「标准答案数据集」，官方在这里指的是每一步从环境拿到的真实结果。

**记住这一句**：循环 = LLM + `tool calls` + `ground truth` + `stopping conditions`；`final output` 由代码判定，`max_turns` 由代码兜底，模型只负责决定下一步做什么。

</div>
