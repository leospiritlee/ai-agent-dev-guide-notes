# 第5章 工具调用协议：一次结构化往返

**一句话**：`function calling` 不是「模型执行了函数」，而是模型返回一个 `structured call`、由你的代码执行、再把 `tool_result` 送回去的一次往返；`input_schema`、`tool_use`、`tool_use_id`、`stop_reason` 四个字段把这次往返钉死。

**为什么需要它**：`Claude Platform Docs` 的定义写得极准：工具使用（也叫 `function calling`）让 Claude 调用你定义或 `Anthropic` 提供的函数；模型根据用户请求与工具描述决定何时调用，然后返回一个 `structured call`，由你的应用执行（`client tools`）或由 `Anthropic` 执行（`server tools`）。「谁执行」这件事最容易被初学时误读，而它决定了参数校验、权限、重试、审计全部落在哪一侧。官方还点明了额外 token 的三个来源：API 请求里的 `tools` 参数（工具名、描述、schema）、请求与响应里的 `tool_use` 内容块、请求里的 `tool_result` 内容块——工具定义是常驻上下文的一部分，这正是第 6、7 章要算的账。

**核心结构**：一次往返有四步。第一步，传入带 `input_schema` 的工具定义，官方给的逐字示例是名为 `get_weather` 的工具，描述为 "Get the current weather for a given location."，`properties` 里只有一个 `location` 字符串字段，描述为 "City and state, e.g. San Francisco, CA"，`required` 为 `["location"]`。第二步，模型返回 `tool_use` 块，并用 `stop_reason` 标出 `tool_use`；默认 `tool_choice` 是 `{"type": "auto"}`，模型每轮自行决定调用工具还是直接回答。第三步，你的代码执行操作，把结果以 `tool_result` 内容块送回，并用 `tool_use_id` 绑定原调用（官方示例逐字写着 `{"type": "tool_result", "tool_use_id": tool_use.id, "content": weather}`）。第四步，模型带着结果继续生成。`strict: true` 加在自定义工具定义上，用来保证工具调用始终严格匹配 schema。

**关键设计**：**`client tools` 与 `server tools` 的分界是「代码在哪跑」**。官方说得很清楚：`client tools`（包括用户自定义工具、以及带 `Anthropic` 定义 schema 的工具如 `bash`、`text_editor`）跑在你的应用里，模型响应带 `stop_reason: "tool_use"` 和一个或多个 `tool_use` 块，你的代码执行后送回 `tool_result`；`server tools`（如 `web_search`、`web_fetch`、`code_execution`、`tool_search`）跑在 `Anthropic` 的基础设施上。这条分界把安全责任也划开了：客户端工具的参数白名单、最小权限凭证、沙箱、审计全在你这侧，第 12 章的 `least privilege`、`egress allowlist`、`audit log` 都从这里长出来。另外 `tool_choice`、`disable_parallel_tool_use`、`parallel tool use`、`tool runner` 这些项决定一次响应里能否并行发起多个调用——并行能省延迟，但也让「哪些动作必须串行、哪些动作不可逆」变成必须显式设计的约束。

**拆解**：

### 逐字读一遍工具定义

官方示例里那三行值得逐字看：`"name": "get_weather"` 是模型唯一能引用的动作名；`description` 承担着「告诉模型什么时候该用它」的全部责任；`input_schema` 里的 `location` 字段描述给了一个具体写法示例。第 6 章会把这三处拆成 `tool descriptions`、`namespacing` 与参数命名三条设计纪律，但根子在这里：模型看到的只有这些文本，没有你的函数体、没有你的类型注解、没有你的注释。`strict: true` 保证的是结构合规，不是语义正确——参数名含糊、描述含糊时，结构依然完美，值依然离谱。这也是第 2 章要「刻意写坏一次 schema」的原因。

### tool_use_id 与 stop_reason 各自防什么

`tool_use_id` 把结果绑回某一次具体调用，在 `parallel tool use` 场景下尤其关键：多个 `tool_use` 块同时返回，若结果不带 id，模型无法把哪个结果对应到哪个动作。`stop_reason` 则把「这一轮是来要工具执行的，不是来给最终答案的」变成代码可判的状态，避免你的应用把一段工具请求误当成回复展示给用户。两者共同把一次往返从「聊天」变成「协议」。综合报告口径把这类约束归到 L1 的类型化工具契约：工具返回值必须结构化、可校验。

### 一次工具调用的 token 账

官方列的三处来源里，最容易被低估的是第一处——`tools` 参数里的工具名、描述与 schema 在每轮请求里都要带上。于是「多给模型几个工具备选」不是零成本的：每个工具都在持续占用 `attention budget`，且功能重叠时还会制造歧义。第 6 章的 `minimal viable set of tools` 与第 7 章的 `context rot` 是同一笔账的两个视角；官方在 `Claude Code` 里把工具响应默认限制在 `25,000 tokens`，是第三个视角。算账的方式很朴素：把工具定义、每轮 `tool_use`、`tool_result` 三部分都算进每成功任务成本，而不是只数模型输出。

### 从一次往返到一次可靠往返

协议本身不保证可靠，还要在应用侧补四件事：把 `tool_result` 做成结构化且可校验的返回值；对执行失败给出可行动的错误信息而不是 traceback；对不可逆动作在调用前插入 `human approval`；对每次调用留下 `audit log`。官方在工具设计文里明确建议「prompt-engineer your error responses」，让错误说明具体的改进方向。综合报告口径则把「在代码里校验工具参数」列为架构性防御之一：白名单收件人域、金额上限这类检查，不能指望模型自觉遵守。第 6 章讲工具形状，第 12 章讲权限边界，中间的接口就是这次往返。

**今天怎么做**：照官方示例手写一个 `get_weather` 工具，完整跑通定义 → `tool_use` → 本地执行 → `tool_result` → 继续生成；然后把 `strict: true` 去掉、把 `location` 改成含糊参数名，对比两轮行为差异。**延伸，非语料原文**：把这一轮的四个字段原样打印出来存成一个样例文件，后面接 `OpenTelemetry` 的 `execute_tool` span 时，它就是最省事的第一条 fixture。

<div class="chapend" markdown="1">

**容易误解的地方**：以为 `function calling` 会让模型执行函数——模型只返回 `structured call`；把 `server tools` 当成部署在自己服务的工具——它跑在厂商基础设施上；以为 `strict: true` 能保证参数语义正确——它保证的是严格匹配 schema；还有忽略 `tool_use_id` 在 `parallel tool use` 下的作用，等到并发调用一多就出现结果与动作错配。

**记住这一句**：`input_schema` 定形状、`tool_use` 提请求、`tool_result` 回真相、`tool_use_id` 保对应；执行权始终在你的代码里，所以参数校验与权限边界也始终在你的代码里。

</div>
