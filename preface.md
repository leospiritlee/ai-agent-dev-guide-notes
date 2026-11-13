## 序

这本册子的起点不是「怎么搭一个 Agent」，而是一次没能上线的运行。跑通一次工具调用只要几十行代码，让第两百次运行仍然可信，要补的东西却横跨提示词、工具契约、上下文预算、状态恢复、评测、权限和灰度。中间那段距离，就是这本指南要填的。

先说清语料从哪来。对本地资料盘做只读扫描，书籍类文件 6461 份（排除 node_modules、coverage、src 等代码目录），严格关键词命中 4 份：《人工智能简史》《深度学习：智能时代的核心驱动力量》《深度学习》《深度学习的数学》，都是通用 AI 科普或数学教材，没有一本 AI Agent 开发专著，与本书主题不对口，故不纳入语料。所以这本不是任何一本书的精读，而是综合整理：底本来自 2026-11-12 的网络多源检索，合并成约 5.6 万字，分两级存放。A 级是一手官方文档原句，用 `WebFetch` 抓取后逐字保留，包括 `Anthropic` 的 `Building effective agents`、`Effective context engineering for AI agents`、`Writing effective tools for agents`，`Claude Platform Docs` 的工具使用页，`Model Context Protocol` 的定义页与架构页，`OpenAI Agents SDK` 首页与运行页，`LangGraph` overview，以及 `OWASP` 的 `2025 Top 10 Risk & Mitigations for LLMs and Gen AI Apps` 类目名。B 级是一份多源综合研究报告，由检索代理汇总，里面出现的比例、预测、基准数字与学习阶段划分都属于二手转述。正文中凡引用 B 级数字，一律写明「综合报告口径」，不把它伪装成官方文档结论。

两级口径不调和。`MCP` 官方文档那页的规格版本标着 `2026-07-28`，研究报告里说这一版把协议改成了 `stateless`；`A2A` 的 v1.0 时间、`Agent Skills` 开放标准的时间都来自 B 级生态综述。这些时间点是各自来源的陈述，本书照录并标明出处级别，不替它们对齐。同理，`Anthropic` 那份文档发表于 2024 年 12 月，却至今仍是 `agentic systems` 分层的行业基准；研究报告是 2026 年 11 月的口径，两者相差近两年，本书不把后者写进前者的引文里。

贯穿全书的一条判据是：**先问这个任务该不该让模型自己决定流程，再问怎么把它做可靠**。`Building effective agents` 的建议非常直白：找最简单的方案，只在确有必要时增加复杂度，甚至可能根本不建 `agentic systems`；多数应用用一次带 `retrieval` 与上下文示例的 LLM 调用就够了。这不是保守，而是算术：综合报告口径里那个 `10 步` 流水线每步可靠率 `95%`、整体只剩约 `59%` 的例子，说的就是复杂度会自己吃掉可靠性。第 1 章立这条边界，第 2—3 章打底座与固定路径，第 4—6 章把循环和工具讲到协议层，第 7—9 章处理上下文、长任务与知识接入，第 10 章讲协议分层与运行时，第 11—12 章把评测、成本、安全、灰度与上手路径收成一条能照着走的线。

## 一条运行的旅程

```text
┌─────────────────────────────────────────────────────┐
│ 一条运行的旅程：从一次工具调用到一次可信上线        │
│ 一 定边界   简单性 > 工作流 > Agent > 该不该上      │
│ 二 打底座   原生 API > 工具契约 > 结构化输出 > 预算 │
│ 三 跑循环   工具调用 > 观察 > 反馈 > 终止条件       │
│ 四 管上下文 token 预算 > 压缩 > 笔记 > 子代理       │
│ 五 接外部   MCP > Skill > A2A > 框架 > 运行时       │
│ 六 敢上线   评测 > 可观测 > 成本 > 安全 > 灰度      │
└─────────────────────────────────────────────────────┘
```

六段是依赖，不是并列的知识点。没定边界就上框架，后面每个组件都在替一个本不存在的假设买单；没写熟一次原生 `function calling` 往返，就看不懂 `tool_use` 与 `tool_result` 之间到底谁在承担失败；没把上下文当预算，长任务会在第 15 步之后悄悄跑偏，而你会去怪模型变笨；没建评测集就谈灰度，`canary` 只是把未知流量放大了一点。顺序反过来的典型症状是：演示那天一次通过，上线后每天都不通过——综合报告口径里，`pass^k` 与 `pass@1` 的差别说的就是这件事。

## 十二章速查表

| 章 | 主题 | 一句话 |
|----|------|--------|
| 01 | 全局坐标：先决定要不要 Agent | `agentic systems`、`workflows`、`agents` 三层分开，`simplicity` 优先 |
| 02 | 最小骨架：增强型 LLM 与原生 API | `augmented LLM`、`structured output`、`streaming`、`prompt caching` 是最短起点 |
| 03 | 工作流：五种可组合的固定路径 | `prompt chaining`、`routing`、`parallelization`、`orchestrator-workers`、`evaluator-optimizer` |
| 04 | Agent 循环：工具调用、反馈与终止条件 | `ground truth`、`stopping conditions`、`max_turns`、`final output` 构成闭环 |
| 05 | 工具调用协议：一次结构化往返 | `input_schema`、`tool_use`、`tool_result`、`stop_reason` 逐字对齐 |
| 06 | 工具设计：ACI 是首要失败源 | `namespacing`、`high signal information`、`token efficiency` 决定调用质量 |
| 07 | 上下文工程：token 是预算 | `context rot`、`attention budget`、`right altitude`、`just-in-time` |
| 08 | 长任务：压缩、结构化笔记与子代理 | `compaction`、`agentic memory`、`sub-agent architectures` 各补一段缺口 |
| 09 | 知识与检索：什么时候才需要 RAG | `200K`、`hybrid retrieval`、`rerank`、`租户隔离` 是判断词而非口号 |
| 10 | 协议与框架分层：MCP、Skill、A2A 与运行时 | `JSON-RPC 2.0`、`tools/list`、`progressive disclosure`、`LangGraph` 在不同层 |
| 11 | 评测、可观测与成本 | `pass^k`、`OpenTelemetry`、`cost per successful task`、`CI gate` |
| 12 | 安全、准入与上手路径 | `least privilege`、`kill switch`、`shadow mode`、`canary`、`ADR` |

## 遇到这些问题，去哪章找

- **「我们这个需求到底要不要上 Agent」** → 第 01 章（`agentic systems`、`predefined code paths`、`open-ended problems`）
- **「第一次动手该写什么」** → 第 02 章（`augmented LLM`、`structured output`、`pip install openai-agents`）
- **「流程能定死，还要不要循环」** → 第 03 章（`workflows`、`gate`、`programmatic checks`）
- **「跑起来就不停，怎么收住」** → 第 04 章（`stopping conditions`、`max_turns`、`MaxTurnsExceeded`）
- **「模型调工具到底传了什么」** → 第 05 章（`input_schema`、`tool_use`、`tool_result`、`tool_use_id`）
- **「工具一堆，模型老是选错」** → 第 06 章（`bloated tool sets`、`namespacing`、`minimal viable set of tools`）
- **「工具返回太长，把上下文撑爆」** → 第 06、07 章（`truncation`、`25,000 tokens`、`high-signal tokens`）
- **「提示词越写越长，效果反而变差」** → 第 07 章（`context rot`、`Goldilocks zone`、`few-shot prompting`）
- **「跑到第十五步就开始胡来」** → 第 07、08 章（`attention budget`、`compaction`、`context pollution`）
- **「重启之后什么都忘了」** → 第 08 章（`structured note-taking`、`NOTES.md`、`context resets`）
- **「到底要不要搭 RAG」** → 第 09 章（`200K`、`prompt caching`、`Agentic RAG`）
- **「知识变了、行为不对、要记住这个人」** → 第 09 章（`RAG`、`微调`、`记忆层` 相关表述见该章）
- **「MCP 和 Skill 和 A2A 有什么区别」** → 第 10 章（`MCP`、`Agent Skills`、`A2A`、`AGENTS.md`）
- **「该选哪个框架，还是干脆不用」** → 第 10 章（`No framework`、`LangGraph`、`OpenAI Agents SDK`、`harness`）
- **「怎么知道改了提示词有没有变坏」** → 第 11 章（`pass^k`、`状态断言`、`CI gate`、`离线评测`）
- **「日志一大堆，还是不知道哪步错了」** → 第 11 章（`OpenTelemetry`、`gen_ai.*`、`trace`、`span`）
- **「账单涨了，但说不清贵在哪」** → 第 11 章（`cost per successful task`、`5–30 倍`）
- **「检索文档里藏了指令怎么办」** → 第 12 章（`Prompt Injection`、`read-write separation`、`human approval`）
- **「出事了怎么立刻停」** → 第 12 章（`kill switch`、`dry-run`、`audit log`、`egress allowlist`）
- **「零基础按什么顺序练、怎么判断真会了」** → 第 12 章（`阶段 0`、`阶段 6`、`ADR`）与结语、判断清单

## 二十个自测问题

1. 说清 `workflows` 与 `agents` 的分界：区别到底在「有没有工具」还是「谁决定控制流」？
2. 为什么官方文档说「最简单的方案」经常意味着不建 `agentic systems`？代价换来了什么？
3. `augmented LLM` 的三类增强分别是什么？为什么这一步往往就够了？
4. `prompt chaining` 里的 `gate` 做什么？为什么它能把 `latency for higher accuracy` 变成可控交易？
5. `parallelization` 的 `sectioning` 与 `voting` 分别适合什么任务？与 `orchestrator-workers` 差在哪？
6. 一次 agent loop 里，`final output` 的判定规则是什么？`handoff` 发生时循环怎么走？
7. `max_turns` 触顶会抛什么？为什么 `MaxTurnsExceeded` 属于安全设施而不是错误处理？
8. `client tools` 与 `server tools` 的根本差别是什么？`stop_reason` 在往返里起什么作用？
9. `strict: true` 解决的是哪一类失败？`tool_use_id` 为什么必须原样带回？
10. 为什么「把现有 API 包一层」是最常见的工具设计错误？该改成什么形状？
11. `namespacing` 怎么帮模型选对工具？`user_id` 比 `user` 好在哪？
12. `context rot` 是什么？为什么它造成的是 `performance gradient rather than a hard cliff`？
13. `right altitude` 的两个失败模式分别是什么？`XML tagging` 与 `Markdown headers` 解决什么？
14. `compaction`、`structured note-taking`、`sub-agent architectures` 各自适合哪类长任务？
15. 为什么 `tool result clearing` 被称作最安全的轻量压缩？什么情况下压缩会压掉关键信息？
16. 什么条件下「全量塞进上下文」比搭 `RAG` 更简单更准？`rerank` 为什么性价比高？
17. `Tools`、`Resources`、`Prompts` 三类 `primitives` 各管什么？`tools/list` 与 `tools/call` 怎么用？
18. `Stdio transport` 与 `Streamable HTTP transport` 的取舍是什么？`OAuth` 落在哪一层？
19. `pass^k` 与 `pass@1` 的差别，为什么决定了「演示成功」和「生产成功」不是一回事？
20. 一票否决的红旗有哪几条？`shadow mode` → `canary` → 放量的顺序为什么不能跳过？

## 一个提醒

这本册子只做一件事：把「做一个 Agent」从一句口号，还原成一组可以逐层追问的工程判断。凡是资料底本里没有的机制、没有的版本特性、没有的性能数字，本书都**明确不写**：不写某厂商未公开的内部实现，不写没有出处的基准分数，不把 B 级研究报告的调研比例写成 A 级官方文档的结论，不把 2026 年的生态动态倒灌进 2024 年那篇文档的原句里。资料互相矛盾或年代不齐时，按各自来源标注，不选边、不调和。

书里凡带「延伸，非语料原文」标记的段落，是编排者给动手开发者的练习设计与学习建议，不是检索所得资料的表述；这类句子一律不加反引号，不伪装成术语。反过来，正文中出现的每一个行内代码词，都能在资料底本里逐字找到——这是本书区别于「凭印象科普」的底线。

还有一条容易被忽略的边界：这是开发指南，不是产品手册。`OpenAI Agents SDK`、`LangGraph`、`MCP` 的具体参数名、方法签名、规格版本会随版本变，请回到各官方文档核对；本书负责的是看懂这些接口背后「为什么这样设计」，以及换一个框架、换一代模型时哪些判断仍然成立。综合报告口径里那句提醒值得抄在这里：`harness` 里每个组件都编码了一个关于「模型做不到什么」的假设，模型变强后这些假设会过期，要逐个拆掉验证。