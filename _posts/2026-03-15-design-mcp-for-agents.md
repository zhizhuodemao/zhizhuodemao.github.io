---
title: 为 Agent 设计工具，而不是为人
date: 2026-03-15 00:00:00 +0800
categories: [AI, MCP]
tags: [ai, mcp, agent, 工具设计, claude-code, js逆向]
---

> MCP 工具设计中的克制、约束与 Agent 思维

## 0. 写在前面

这篇文章不是教程，是一组设计原则。

如果你正在构建 MCP server，或者正在考虑构建一个，我想分享一个核心认知：**给 AI Agent 设计工具和给人设计 API 是两件完全不同的事**。传统 API 设计追求完整性和灵活性；Agent 工具设计追求克制和约束。

这些原则不是我凭空发明的。Anthropic 在 [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) 中提出了 ACI（Agent-Computer Interface）的概念；MCP 协议在 [spec](https://modelcontextprotocol.io/docs/concepts/tools) 中定义了工具应该如何被设计。这篇文章是我在实践中对这些原则的理解和延伸。

## 1. 你的用户是 Agent，不是人

MCP 工具的直接使用者不是人类开发者，而是 LLM Agent。这个区别决定了一切。

**人类开发者**可以浏览文档理解 API 全貌，能记住初始化顺序和状态依赖，会忽略不需要的返回字段，在几百个 API 中快速定位需要的那个。

**LLM Agent** 做不到这些。它面对的约束是刚性的：

```
┌──────────────────────────────────┐
│ 系统提示（System Prompt）          │ ← 工具定义在这里
│                                    │
│ 每个工具定义占 ~200-500 tokens      │
│ 工具越多，这里占得越大               │
│ 留给推理的空间就越小                 │
│                                    │
├──────────────────────────────────┤
│ 对话历史 + 工具返回值               │ ← 越大，前面的内容越早被压缩
├──────────────────────────────────┤
│ 当前推理空间                       │ ← 上面占得越多，这里越小
└──────────────────────────────────┘
```

举个具体的例子。假设你做了一个数据库 MCP server，Agent 的任务是"查出过去 7 天的活跃用户数"。

**场景 A**：你提供了 8 个工具——`connect`、`list_tables`、`describe_table`、`query`、`explain_query`、`list_indexes`、`get_stats`、`disconnect`。Agent 可能会这样做：

```
1. connect(host, port, db)         ← 先连接
2. list_tables()                   ← 看有什么表
3. describe_table("users")         ← 看表结构
4. query("SELECT COUNT(*) ...")    ← 执行查询
```

4 步，其中前 3 步是"准备工作"。而且 Agent 可能会困惑：要不要先 `list_indexes` 看看有没有索引？要不要最后 `disconnect`？

**场景 B**：你提供了 2 个工具——`query(sql)` 和 `describe_schema(table?)`。`query` 内部自动管理连接池。Agent 会这样做：

```
1. query("SELECT COUNT(DISTINCT user_id) FROM events WHERE ts > NOW() - INTERVAL '7 days'")
```

1 步。直接得到结果。

Anthropic 在他们的 Agent 设计指南中点明了该怎么想这件事：

> "Put yourself in the model's shoes. Is it obvious how to use this tool, based on the description and parameters, or would you need to think carefully about it?"
>
> — [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents), Anthropic

如果你需要"仔细想"才能用对，Agent 大概率也用不对。

## 2. 克制：少即是多

### 为什么要少

工具越多，问题越多：

1. **工具定义本身就是成本**。每个工具定义占 200-500 tokens 的系统提示空间。工具越多，Agent 还没开始干活，"思考空间"就已经被挤占了大半。

2. **选择越多，出错越多**。Agent 要从一大堆工具中选出正确的那个，工具越多越容易选错。假设你有一个 `get_users` 和一个 `list_users`，Agent 很可能每次都要犹豫该调哪个。选错一次就浪费一整轮调用——不仅浪费了这次的 token，还污染了上下文历史。

3. **上下文会溢出**。每次工具调用的返回值累积在对话历史中。考虑一个真实场景：Agent 在第 2 步拿到了关键线索，但到了第 15 步时，前面的对话已经因为上下文窗口不够被压缩了——Agent 就"忘记"了第 2 步的发现，开始兜圈子重复之前做过的事。工具越少 → 步骤越少 → 这种遗忘就越不容易发生。

Anthropic 的建议很直白：

> "When building applications with LLMs, we recommend finding the simplest solution possible, and only increasing complexity when needed."
>
> — [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents), Anthropic

### 一个标杆：Claude Code 只有 26 个工具

Anthropic 自己的 Agent 产品 Claude Code 是最好的参照。它能完成几乎所有软件工程任务——读写文件、搜索代码、执行命令、管理 Git、做计划、跑测试、调用子 Agent、操作 Jupyter Notebook——总共只用了 **26 个内置工具**：

```
Read, Edit, Write, Glob, Grep, Bash, Agent, LSP,
WebFetch, WebSearch, AskUserQuestion, Skill,
NotebookEdit, EnterPlanMode, ExitPlanMode,
TaskCreate, TaskGet, TaskUpdate, TaskList,
TaskOutput, TaskStop, EnterWorktree, ExitWorktree,
CronCreate, CronDelete, CronList
```

26 个。不是 260 个。

注意这些工具的设计哲学：

- **`Read` 而不是 `cat + head + tail + sed`**。Claude Code 的系统提示里甚至明确写着："To read files use Read instead of cat, head, tail, or sed"。一个 `Read` 工具通过 `offset` 和 `limit` 参数覆盖了所有"读取文件某一部分"的场景。
- **`Grep` 而不是 `grep + rg + ag + find`**。一个工具，支持正则、文件类型过滤、上下文行数、多种输出模式。
- **`Edit` 而不是 `sed + awk + patch`**。通过 `old_string → new_string` 的精确替换，消除了复杂的正则替换语法和行号计算——这正是 Anthropic 说的 "poka-yoke"（防错法）。

最难的不是加工具，而是**克制住不加工具**。当你的底层能力足够丰富时，把每个能力都暴露成工具是最容易的事——但也是对 Agent 最不友好的事。Anthropic 选择了 26 个，这本身就是一个设计声明。

### 怎么做到少

**规则 1：删掉 90% 场景用不到的工具。**

底层 API 可能有几百个方法。但你的目标用户（Agent）在特定场景下，常用的操作就那么几个。把这些操作做成工具，其他的不暴露。

以浏览器自动化为例。Playwright 的 API 有几十个方法：`goto`、`reload`、`goBack`、`goForward`、`waitForLoadState`、`waitForURL`、`waitForNavigation`、`setExtraHTTPHeaders`、`setDefaultTimeout`、`route`、`unroute`……如果全暴露，Agent 每次导航前都要纠结"要不要先 `setExtraHTTPHeaders`？要不要 `waitForLoadState`？"。实际上 Agent 90% 的时候只需要 `navigate(url)` 和 `reload()`，让工具内部自动处理等待策略和 headers。

没有的工具不会被误用。没有 `debugger_enable` 这个工具，Agent 就不会花一步去"启用调试器"；没有 `cat`，它就会用 `Read`。**不存在的选项不会造成错误的选择**。

**规则 2：每个工具都应该直接推进用户的任务。**

如果一次工具调用的目的不是"获取信息"或"执行操作"，而是"配置工具系统本身"——那说明架构有问题。Agent 的每一步都应该离目标更近一步，而不是花在与任务无关的脚手架工作上。

一个简单的检验方法：把 Agent 的每一步调用写下来，划掉所有"配置/初始化/清理"步骤，剩下的才是有效步骤。如果有效步骤只占一半，说明你的工具设计让 Agent 在做无用功。

Anthropic 提出了一个很好的思维框架：

> "Put yourself in the model's shoes."
>
> — [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents), Anthropic

站在 Agent 的角度想：如果你是一个拿到工具列表就要开始干活的执行者，你希望工具是"拿起来就能用"的，还是"先花几步配置环境才能用"的？

**规则 3：工具应该是自包含的（self-contained）。**

Anthropic 在同一篇文章中强调：

> "Poka-yoke your tools. Change the arguments so that it is harder to make mistakes."
>
> — [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents), Anthropic

Poka-yoke（防错法）的核心是**让错误在结构上不可能发生**。如果一个工具在调用时依赖某个前置条件，与其让 Agent 记住"先调 A 再调 B"这个顺序，不如让工具 B 在内部自动处理 A 的逻辑。

回到数据库的例子：与其让 Agent 记住 `connect → query → disconnect` 这个三步流程，不如让 `query(sql)` 内部自动管理连接。这样做有两个好处：减少了工具数量（3 → 1），也消除了 Agent 因为忘记 `connect` 或 `disconnect` 而失败的可能性。

## 3. Harness 模式：在工具侧做约束

### 什么是 Harness

Harness 的意思是"约束装置"。在 Agent 工具设计中，harness 模式指的是：**由工具侧来约束输出的格式和大小，而不是期望 Agent 自己做数据筛选**。

核心洞察：**LLM 的上下文不是免费的，每一个 token 都有成本**。你不能指望 LLM"只取它需要的数据"。如果你一次返回了 20,000 tokens 的完整 HTTP headers，即使工具描述里写了"请只关注关键字段"——那 20,000 tokens 也已经实实在在地消耗了上下文空间，而且 Agent 在后续推理时必须"跳过"这些无用数据去找真正有用的几行，这本身就增加了出错概率。

Anthropic 在实践中用了一个制造业概念——**防错法（Poka-yoke）**：

> 在构建编码 Agent 时，他们发现模型在使用相对路径时会犯错。解决方案不是在 prompt 里写"请使用绝对路径"，而是把工具改成**只接受绝对路径**。修改后，"the model used this method flawlessly"。
>
> — [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents), Anthropic

不是教 Agent 做对的事，而是**让工具不允许做错的事**。

### Harness 的实践方式

**1. 默认返回摘要，不返回全量**

想象一个网络请求监控工具。Agent 的任务是"找到登录接口的请求"。

```
# 坏：返回完整的请求对象
{
  "id": 3,
  "url": "https://api.example.com/v1/auth/login",
  "method": "POST",
  "status": 200,
  "requestHeaders": {
    "Accept": "application/json",
    "Accept-Encoding": "gzip, deflate, br",
    "Accept-Language": "zh-CN,zh;q=0.9,en;q=0.8",
    "Connection": "keep-alive",
    "Content-Type": "application/json",
    "Cookie": "session=abc123; _ga=GA1.2.xxx; _gid=GA1.2.yyy; ...",
    "Host": "api.example.com",
    "Origin": "https://example.com",
    "Referer": "https://example.com/login",
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) ...",
    "X-Request-ID": "req-uuid-here",
    "X-Trace-ID": "trace-uuid-here"
  },
  "responseHeaders": {
    "Content-Type": "application/json",
    "Set-Cookie": "token=xyz789; Path=/; HttpOnly; Secure",
    "X-RateLimit-Remaining": "99",
    "X-RateLimit-Reset": "1710000060",
    ...
  },
  "timing": { "dns": 5, "connect": 12, "ssl": 8, "send": 1, "wait": 45, "receive": 3 },
  "body": "{\"code\":0,\"data\":{\"token\":\"eyJhbGciOiJIUzI1NiJ9...\",\"user\":{...}}}"
}
```

这个返回可能有 800-1,500 tokens——而 Agent 只需要知道"有一个 POST /auth/login 请求，状态码 200"就够了。如果列表有 20 个请求，全量返回就是 16,000-30,000 tokens。

```
# 好：一行摘要，~30 tokens
id=3 | POST /v1/auth/login | 200 | 0.8kb
id=5 | GET  /v1/user/me    | 200 | 1.2kb
id=8 | GET  /v1/feed        | 200 | 15kb
```

20 个请求的摘要只需要 ~600 tokens。Agent 一眼就能判断 `id=3` 是目标。需要看详情？再用 `get_detail(id=3)` 去获取**单条**的完整信息。

这就是**两层漏斗**设计：`list` 返回摘要用于决策，`get` 返回详情用于分析。

**2. 在工具内部做信息裁剪**

不要让 Agent 通过参数来控制返回数据的大小。如果你发现自己加了 `maxSize`、`autoSummarize`、`fieldFilter`、`stripBase64` 这些参数——**你在把约束的责任推给调用者**。

想想看：Agent 第一次调用这个工具时，它怎么知道 `maxSize` 该设多少？设大了浪费上下文，设小了可能截断关键信息。Agent 没有"经验"来做这个判断，它只能猜。

正确的做法是在工具实现中硬编码合理的默认值：

```python
# 工具内部的 harness 逻辑（伪代码）
def format_variables(scope_variables):
    # 跳过 global scope —— 浏览器的 global 有几千个变量，全返回毫无意义
    if scope.type == "global":
        return "  (global scope omitted)"

    # 最多展示 20 个变量
    shown = scope_variables[:20]
    result = [f"  {v.name}: {truncate(v.value, max_len=200)}" for v in shown]

    if len(scope_variables) > 20:
        result.append(f"  ... and {len(scope_variables) - 20} more")

    return "\n".join(result)
```

Agent 不需要思考"要不要截断"，工具已经替它做了最合理的选择。

**3. 返回文本，不返回原始 JSON**

LLM 在训练中见过无数的 stack trace、日志输出、命令行结果。这些是它最"自然"的输入格式。

Anthropic 也建议：

> "Keep the format close to what the model has seen naturally occurring in text on the internet."
>
> — [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents), Anthropic

同样一份调用栈信息，两种返回方式：

```
# 好：格式化文本（~120 tokens）
Call Stack:
  1. encryptSign @ vendor.js:1234:56
  2. interceptRequest @ vendor.js:2345:78
  3. XMLHttpRequest.send @ (native)

Async Parent:
  4. handleClick @ app.js:89:12
  5. onClick @ react-dom.js:3456:23
```

```json
// 坏：原始 JSON（~250 tokens，还更难读）
{
  "callFrames": [
    {"functionName": "encryptSign", "url": "https://cdn.example.com/static/js/vendor.abc123.js", "lineNumber": 1233, "columnNumber": 55, "scriptId": "42"},
    {"functionName": "interceptRequest", "url": "https://cdn.example.com/static/js/vendor.abc123.js", "lineNumber": 2344, "columnNumber": 77, "scriptId": "42"},
    {"functionName": "send", "url": "", "lineNumber": 0, "columnNumber": 0, "scriptId": "0"}
  ],
  "parent": {
    "callFrames": [
      {"functionName": "handleClick", "url": "https://cdn.example.com/static/js/app.def456.js", "lineNumber": 88, "columnNumber": 11, "scriptId": "38"},
      {"functionName": "onClick", "url": "https://cdn.example.com/static/js/react-dom.ghi789.js", "lineNumber": 3455, "columnNumber": 22, "scriptId": "15"}
    ]
  }
}
```

文本版不仅 token 更少（约一半），而且 LLM 解读起来更快——它见过无数类似格式的 stack trace。JSON 版则需要 LLM 在一堆花括号和重复的键名中提取信息，行号还是 0-based（`1233` vs 人类习惯的 `1234`），URL 是完整 CDN 地址而不是简短文件名。

**4. 合并相关信息到一次返回**

如果 Agent 每次在某个场景下都需要同时知道 A、B、C 三样信息，把它们合并到一个工具的返回中。

以调试场景为例。断点命中后，Agent **总是**需要知道三件事：在哪停的（代码位置）、怎么到这的（调用栈）、当前状态是什么（变量值）。

```
# 坏：三次调用，三次 token 往返
get_call_stack()        → 返回调用栈（~200 tokens）
get_variables()         → 返回变量（~300 tokens）
get_current_location()  → 返回代码位置（~100 tokens）

# 总计：3 次调用 + 3 次返回 ≈ 900 tokens（含调用开销）
```

```
# 好：一次调用，所有决策依据
get_paused_state()

→ 返回（~400 tokens）：
  Paused at: vendor.js:1234:56 (encryptSign)

  Call Stack:
    0. encryptSign @ vendor.js:1234:56
    1. interceptRequest @ vendor.js:2345:78

  Local Variables:
    url: "/api/v1/login"
    payload: {"username": "test", "password": "***"}
    signature: "a3f8b2c1..."
```

一次调用，Agent 马上就能判断下一步该做什么——是步进看 `encryptSign` 内部逻辑，还是已经找到了目标函数。

## 4. 任务思维：从 Agent 的目标出发

### API 平铺 vs 任务抽象

底层系统通常提供低层的、原子化的方法。以 Chrome DevTools Protocol（CDP）为例，仅 `Debugger` 域就有十几个方法：

```
Debugger.enable
Debugger.disable
Debugger.setBreakpointByUrl
Debugger.setBreakpoint
Debugger.removeBreakpoint
Debugger.getPossibleBreakpoints
Debugger.continueToLocation
Debugger.pause
Debugger.resume
Debugger.stepOver
Debugger.stepInto
Debugger.stepOut
Debugger.evaluateOnCallFrame
Debugger.setVariableValue
...
```

一种常见的错误是把这些方法一对一地包装成 MCP 工具。这是 **API 平铺思维**：底层有什么就暴露什么。

任务思维问的是另一个问题：**Agent 想做什么？** Agent 不想"调用 Debugger.setBreakpointByUrl"，它想"在加密函数的入口处断下来"。

这两个意图之间的差距，就是工具设计者应该填补的：

```
# API 平铺：Agent 需要知道 CDP 的工作流程
1. enable_debugger()                              ← 必须先启用
2. search_scripts("encryptSign")                  ← 搜索函数位置
3. get_script_source(scriptId, line, line+10)      ← 确认代码
4. set_breakpoint(scriptId, lineNumber, column)    ← 算好行列号设断点
5. resume()                                        ← 恢复执行等待命中

# 任务抽象：Agent 只需要表达意图
1. set_breakpoint_on_text("encryptSign")  ← 内部自动搜索、定位、设置
```

第一种方式中，Agent 需要理解：调试器要先 enable、scriptId 从哪来、行列号怎么算（压缩代码可能只有一行，列号是 308556）、为什么要 resume。**这些都是 CDP 的实现细节，不是 Agent 需要关心的。**

### 工具描述就是 Prompt

Anthropic 揭示了一个关键发现：

> "We actually spent more time optimizing our tools than the overall prompt."
>
> — [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents), Anthropic

工具定义（名字、描述、参数）直接出现在 LLM 的系统提示中。它们**就是 prompt 的一部分**。每一个字都影响 Agent 的行为。

**工具名应该自解释。**

| 工具名 | 问题 |
|--------|------|
| `get_initiator_stack_trace_for_network_req` | 太长，浪费 tokens |
| `get_stack` | 太模糊，什么的 stack？ |
| `get_request_initiator` | 刚好：足够明确，又不冗余 |

**描述写两句话。** 第一句说"做什么"，第二句说"什么时候用"或"为什么有用"。

```
# 好（2 句，~30 tokens）
"Gets the JavaScript call stack that initiated a network request.
 This helps trace which code triggered an API call."

# 坏（6 句，~80 tokens，大部分信息 Agent 用不到）
"Gets the full JavaScript call stack including async parent frames for a
 specific network request identified by its unique request ID. Supports
 both XHR and Fetch requests. Returns formatted stack trace with function
 names, file URLs, line numbers and column numbers. Use the requestId from
 list_network_requests. Note: initiator information is only available for
 requests captured after the debugger was enabled..."
```

多出来的 50 tokens 看起来不多？乘以几十个工具，差距就是几千 tokens 的系统提示空间。而且长描述中的条件说明（"only available after the debugger was enabled"）反而引入了 Agent 需要记住的前置条件——如果你的工具是自包含的，这个条件根本不应该存在。

**参数描述写明上下游。** Agent 需要知道这个参数的值从哪来：

```
requestId: "The request ID (from list_network_requests) to get the initiator for."
```

这一句话让 Agent 知道：先调 `list_network_requests` 拿到 ID，再调这个工具。不需要额外的文档，不需要看 README，工具定义本身就是完整的使用说明。

### 工具间的隐式链路

好的工具集不是一堆独立的函数，而是一条**可预测的分析链路**。工具之间通过返回值中的 ID 自然串联：

```
list_items        →  返回 id
                       ↓
get_detail(id)    →  详情（包含关联的 file_path、line_number）
                       ↓
get_source(path, line) → 源代码
                       ↓
set_breakpoint(path, line) → 断点设置完成
```

Agent 不需要阅读使用手册就能发现这条链路。每个工具的返回值自然引导向下一个工具——就像 Unix 管道：`ls | grep | xargs` 的每一步输出都是下一步的输入。LangGraph 的文档把这种模式叫做 "tool result integration"：

> "Each tool execution produces observations that feed back into the LLM's context, creating informed subsequent decisions."
>
> — [LangGraph Agentic Concepts](https://langchain-ai.github.io/langgraph/concepts/agentic_concepts/), LangChain

反面例子是工具之间没有自然连接：一个工具返回的 `scriptId` 在页面刷新后失效了，Agent 调用下一个工具时传入了失效的 ID，得到一个莫名其妙的错误。**如果 ID 会失效，工具应该接受不会失效的标识符**（比如 URL）作为输入，内部自己做 ID 映射。

## 5. 封装复杂性

### 状态适配放在工具内部

如果一个工具在不同状态下需要不同的执行策略，**让工具自己判断**，不要让 Agent 判断。

以"在页面中执行一段 JavaScript"为例。根据当前状态，实际需要三种不同的执行方式：

```
状态 1: Agent 正在断点暂停处
  → 需要在当前栈帧的作用域中执行，才能访问局部变量

状态 2: Agent 需要访问页面全局变量（如 window.xxx）
  → 需要在页面主世界（main world）中执行

状态 3: 普通的代码执行
  → 在隔离上下文中执行最安全
```

```
# 坏：Agent 需要判断状态，选择 3 个工具之一
evaluate_in_frame(code)    ← 断点上下文
evaluate_in_page(code)     ← 页面主世界
evaluate_isolated(code)    ← 隔离上下文

# 好：一个工具，内部自动适配
evaluate(code)
  ├── 检测到断点暂停？ → evaluateOnCallFrame
  ├── 需要访问 window？ → 注入到页面主世界
  └── 默认 → 隔离上下文执行
```

Agent 不需要知道什么是"栈帧"、什么是"主世界"。它只知道"我要执行这段代码"，工具负责选择最合适的方式。这样工具数量从 3 减少到 1，Agent 的心智负担也从"理解三种执行上下文的区别"减少到零。

### 安全/环境约束封装在内部

如果你的工具运行在有反检测需求、权限限制、或环境差异的场景下，这些处理逻辑应该在**工具实现内部**，而不是暴露给 Agent。

以浏览器自动化的反检测为例。直接调用 CDP 的 `Runtime.enable` 会被很多反爬脚本检测到。一个好的 `evaluate(code)` 工具应该内部自动选择不触发检测的执行路径（比如通过 DOM 注入而不是 CDP 调用），而不是让 Agent 在 `evaluate_safe(code)` 和 `evaluate_fast(code)` 之间做选择。

Agent 不需要知道你是用什么方式绕过反爬检测的，也不需要知道你的执行引擎在不同平台上有不同的实现。它只需要调用一个工具，拿到结果。

### 错误处理返回有用信息

Agent 收到错误后的反应完全取决于错误信息的质量。对比这两种：

```
# 坏：原始异常——Agent 不知道怎么修复
"Error: Protocol error (Debugger.evaluateOnCallFrame): Cannot find context with specified id"

Agent 的反应：困惑。可能会重试同样的调用，然后再次失败。
```

```
# 好：可执行的指引——Agent 知道下一步做什么
"Cannot evaluate: execution is not paused at a breakpoint.
 Use set_breakpoint_on_text() to set a breakpoint first, then trigger the code to pause."

Agent 的反应：调用 set_breakpoint_on_text()，然后重试。
```

好的错误信息包含三个要素：**发生了什么** + **为什么** + **建议怎么做**。Agent 看到建议就能直接行动，而不是在原始错误信息里猜测原因。

## 6. 设计原则清单

### 克制

- 工具数量控制在 **50 个以内**——Claude Code 用 26 个工具覆盖了整个软件工程场景
- 删掉 90% 场景用不到的工具——不存在的选项不会造成错误的选择
- 每个工具都应该直接推进用户任务，不要有"配置工具系统本身"的工具
- 工具应该自包含——调用即生效，内部自动处理前置依赖

### Harness

- **默认返回摘要**——列表工具返回一行摘要（~30 tokens/条），详情工具返回单条完整信息
- 在工具侧做数据裁剪——硬编码合理默认值，不依赖 Agent 传参控制
- 单次返回控制在 **2,000 tokens 以内**——超过这个阈值说明返回了太多无用信息
- 返回格式化文本——LLM 读 stack trace 比读 JSON 快，tokens 也更少
- 主动跳过无用信息——global scope 变量、base64 数据、完整 Cookie 字符串

### 任务导向

- 每个工具对应一个**认知步骤**，不是一个底层 API 方法
- 合并高频组合操作——搜索 + 定位 + 设断点 = 一个工具
- 工具描述**两句话**：做什么 + 什么时候用
- 参数描述写明数据来源——`"The ID (from list_xxx) to ..."`
- 工具间通过返回值自然串联——像 Unix 管道一样，上一步的输出引导下一步的输入

### 封装

- 状态适配放在工具内部——3 种执行策略合并为 1 个工具
- 安全/环境约束对 Agent 透明——反检测、平台差异等在内部处理
- 错误信息包含三要素：**发生了什么 + 为什么 + 建议怎么做**

## 7. 自检问题

在发布你的 MCP server 之前：

1. **完成最常见的 3 个任务，各需要几次工具调用？** 如果超过 5 次，考虑合并。
2. **去掉一半工具后，还能完成 80% 的任务吗？** 如果能，去掉它们。
3. **有没有"不推进用户任务"的调用步骤？** 配置、初始化、状态检查——这些应该在工具内部。
4. **单次工具返回的平均 token 数是多少？** 如果经常超过 2,000 tokens，需要做摘要。
5. **新来的 Agent 不看文档，能正确使用你的工具吗？** 工具名和参数描述应该自解释。
6. **工具返回的 ID 会失效吗？** 如果会，换一个不会失效的标识符。

## 8. 实践案例：js-reverse-mcp 的设计

上面讲了一堆原则，这一节用一个真实项目把它们串起来。

[js-reverse-mcp](https://github.com/nicecaesar/js-reverse-mcp) 是一个面向 JS 逆向分析的 MCP server。它的底层是 Chrome DevTools Protocol（CDP）——一个有几百个方法、横跨十几个域的复杂协议。最终暴露给 Agent 的工具只有 **35 个**。

### 35 个工具的全景

| 类别 | 工具 | 说明 |
|------|------|------|
| **脚本分析** | `list_scripts` | 列出页面中所有 JS 脚本 |
| | `get_script_source` | 获取源码（支持行范围和字符偏移） |
| | `search_in_sources` | 在所有脚本中搜索字符串/正则 |
| **断点** | `set_breakpoint_on_text` | 搜索代码文本并自动设置断点 |
| | `break_on_xhr` | 在 XHR/Fetch 请求处断下 |
| | `remove_breakpoint` | 移除断点 |
| | `remove_all_breakpoints` | 移除所有断点 |
| | `list_breakpoints` | 列出所有断点 |
| **调试控制** | `get_paused_info` | 获取断点状态（调用栈 + 变量 + 位置） |
| | `resume` / `pause` | 恢复/暂停执行 |
| | `step_over` / `step_into` / `step_out` | 单步调试 |
| **函数追踪** | `trace_function` | 追踪任意函数调用（含 webpack 内部函数） |
| | `inject_before_load` | 在页面加载前注入脚本 |
| | `remove_injected_script` | 移除注入的脚本 |
| **网络** | `list_network_requests` | 列出请求（默认 20 条摘要） |
| | `get_network_request` | 获取单个请求详情 |
| | `get_request_initiator` | 获取请求的 JS 调用栈 |
| | `break_on_xhr` / `remove_xhr_breakpoint` | XHR 断点 |
| **WebSocket** | `list_websocket_connections` | 列出连接 |
| | `get_websocket_messages` / `get_websocket_message` | 获取消息 |
| | `analyze_websocket_messages` | 分析消息模式 |
| **页面管理** | `list_pages` / `select_page` / `new_page` / `navigate_page` | 页面操作 |
| | `list_frames` / `select_frame` | Frame 操作 |
| | `take_screenshot` | 截图 |
| **运行时** | `evaluate_script` | 在页面中执行 JS |
| **控制台** | `list_console_messages` / `get_console_message` | 控制台消息 |

35 个。CDP 的 `Debugger` 域有十几个方法，`Network` 域有十几个方法，`Runtime`、`Page`、`DOM`、`DOMDebugger` 各有几十个——加起来几百个方法，削减到 35 个。

### 砍掉了什么

**没有 `enable` / `disable`。** CDP 要求在使用 `Debugger` 域之前先调用 `Debugger.enable`，使用 `Network` 域之前先调用 `Network.enable`。在 js-reverse-mcp 中，这些初始化在选择页面时自动完成。Agent 调用 `set_breakpoint_on_text` 时不需要知道调试器已经在背后启用了。

**没有 `Debugger.setBreakpointByUrl` 和 `Debugger.setBreakpoint`。** 这两个 CDP 方法需要调用者提供精确的行号和列号。对于压缩后的 JS 代码（整个文件可能只有一行，几十万字符），计算列号是个复杂任务。`set_breakpoint_on_text` 把搜索、定位、设置三步合并为一步。

**没有 `Runtime.evaluate`。** CDP 的 `Runtime.evaluate` 有一个致命问题：调用它会隐式触发 `Runtime.enable`，而这个命令是各大反爬系统（Cloudflare、DataDome、字节 bdms 等）重点检测的目标。`evaluate_script` 通过 DOM 注入脚本的方式绕过了这个限制——Agent 不需要知道这些反检测细节。

**没有 `Network.getResponseBody`、`Network.getRequestPostData` 等。** 这些被合并到 `get_network_request` 中，按需返回。

**没有 `Debugger.getScriptSource`。** 被 `get_script_source` 取代，接受 URL（不会失效）而不是 scriptId（页面刷新后失效）。

### 几个工具的设计细节

**`set_breakpoint_on_text`：搜索 + 定位 + 设置，一步到位**

Agent 输入一段代码文本（比如函数名），工具内部做了三件事：

```
1. 调用 Debugger.searchInContent 在所有已加载脚本中搜索文本
2. 获取匹配位置的源码，精确计算列号（处理压缩代码中一行几十万字符的情况）
3. 调用 Debugger.setBreakpointByUrl 设置断点
```

支持 `urlFilter` 缩小搜索范围，支持 `occurrence` 指定第几个匹配，支持 `condition` 条件断点。断点在页面导航后自动恢复——Agent 不需要在每次页面刷新后重新设置。

**`trace_function`：追踪 webpack/rollup 打包的内部函数**

这是一个典型的"看起来简单，实现复杂"的工具。Agent 只需要提供函数名，工具内部用 8 种模式搜索函数定义：

```javascript
// 工具内部搜索的 8 种模式
`function ${name}`        // function encryptSign
`${name}=function`        // encryptSign=function
`${name} = function`      // encryptSign = function
`${name}=(`               // encryptSign=(  (箭头函数)
`${name} = (`             // encryptSign = (
`${name}(`                // encryptSign(   (调用处)
`${name}:function`        // encryptSign:function (对象方法)
`${name}: function`       // encryptSign: function
```

找到之后，不是设置普通断点，而是设置 **logpoint**（条件断点 + console.log）——函数每次被调用时自动记录参数和返回值，但不会暂停执行。这意味着 Agent 可以在不打断页面正常运行的情况下观察函数行为。

**`evaluate_script`：三种执行上下文，一个入口**

Agent 调用 `evaluate_script(code)` 时，工具内部做了三层判断：

```
如果当前在断点暂停状态？
  → 使用 Debugger.evaluateOnCallFrame 在当前栈帧中执行
  → Agent 可以直接访问断点处的局部变量

如果 mainWorld=true？
  → 通过 DOM <script> 注入执行（绕过 Runtime.enable 检测）
  → Agent 可以访问 window.xxx 等页面全局变量

否则：
  → 在隔离的执行上下文中执行
  → 最安全，不会干扰页面状态
```

注意第二种路径的实现方式：创建一个隐藏的 `<div>` 作为 DOM bridge，注入一个 `<script>` 标签在页面主世界中执行代码，然后通过 DOM 属性把结果传回。这个方案完全避免了 CDP `Runtime.enable`，但 Agent 看到的只是一个 `mainWorld: true` 的布尔参数。

**`get_paused_info`：一次返回所有决策依据**

断点命中后的返回值结构：

```
🔴 Execution Paused

Reason: breakpoint
Hit breakpoints: bp-1

📍 Call Stack:
  0. encryptSign @ vendor-dynamic.js:1:308556
     CallFrameId: {"ordinal":0,"injectedScriptId":1}
  1. interceptRequest @ vendor-dynamic.js:1:309012
     CallFrameId: {"ordinal":1,"injectedScriptId":1}

🔍 Scope Variables (top frame):
  [local]:
    url: "/api/sns/web/v1/homefeed"
    payload: {"source_note_id":"6712..."}
    timestamp: 1710000000
    sign: {"X-s":"XYW_eyJz...","X-t":"1710000000"}
    ... and 3 more

  [closure]:
    config: {"baseURL":"https://edith.xiaohongshu.com",...}
    instance: [Object]

💡 Use resume, step_over, step_into, or step_out to continue.
```

几个 harness 细节：
- **跳过 global scope**。浏览器的全局作用域有几千个变量（`window` 的所有属性），全返回会吃掉 10,000+ tokens。只显示 local 和 closure。
- **变量限制 20 个**。超出部分只报数量 `... and 3 more`，不浪费上下文。
- **提供 CallFrameId**。Agent 如果要在特定栈帧中执行代码，这个 ID 直接可用。
- **尾部提示下一步操作**。Agent 不需要记住有哪些调试控制工具，返回值本身就列出了可选操作。

### 返回格式：Markdown 文本，不是 JSON

所有工具的返回值都通过 `response.appendResponseLine()` 构建为格式化文本。以 `get_request_initiator` 为例：

```
Request initiator for https://edith.xiaohongshu.com/api/sns/web/v1/homefeed:

Type: script
URL: vendor-dynamic.js
Line: 1

Call Stack:
  1. sendRequest @ vendor-dynamic.js:1:308556
  2. (anonymous) @ vendor-dynamic.js:1:309012
  3. XMLHttpRequest.send @ (native)

Async Parent Stack:
  1. handleClick @ app.js:89:12
  2. onClick @ react-dom.js:3456:23
```

注意：URL 显示的是简短文件名（`vendor-dynamic.js`），不是完整 CDN 地址（`https://sns-webpkg-mscdn.xhscdn.com/fe-static/vendor-dynamic.abc123.js`）。行号是 1-based（人类习惯），不是 CDP 原始的 0-based。这些都是工具侧的小处理，但累积起来让 Agent 的上下文干净了很多。

### 反检测作为透明约束

js-reverse-mcp 基于 Patchright（Playwright 的反检测分支），内置了多层反检测：

| 层级 | 措施 | Agent 是否感知 |
|------|------|------|
| C++ 层 | 移除 `navigator.webdriver`，避免 CDP 泄露 | 否 |
| 启动参数 | 60+ 隐身参数，移除自动化特征 | 否 |
| 导航策略 | 不通过 CDP `Network.enable` 捕获请求 | 否 |
| 脚本执行 | DOM 注入替代 `Runtime.enable` | 仅一个 `mainWorld` 参数 |
| 请求伪装 | 自动带 Google Referer | 否 |
| 状态保持 | 持久化 user-data-dir，登录态跨会话保留 | 否 |

Agent 用 `navigate_page` 打开一个有反爬检测的网站时，不需要调用任何"反检测配置"工具。一切都在工具内部默认处理好了。这就是 harness：**约束和安全策略封装在工具内部，对 Agent 完全透明**。

## 9. 结语

传统 API 设计的美德——完整性、灵活性、可组合性——在 Agent 工具设计中会变成负担。

一个好的 MCP 工具集更像是一个**高度约束的 DSL**：少量高级原语，每个完成一个完整的任务，默认返回最小必要信息，内部自动处理状态和约束。

记住 Anthropic 的那句话：**站在模型的角度想——用这个工具，显而易见吗？**

如果答案不是"显然"，请简化它。

---

**参考资料**

- Anthropic. [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents). 2024.
- Model Context Protocol. [Tools Specification](https://modelcontextprotocol.io/docs/concepts/tools). 2025.
- LangChain. [LangGraph Agentic Concepts](https://langchain-ai.github.io/langgraph/concepts/agentic_concepts/). 2025.
