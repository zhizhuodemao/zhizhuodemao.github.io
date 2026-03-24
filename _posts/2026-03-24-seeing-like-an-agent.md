---
title: "Seeing like an Agent：大模型工具的设计指南"
date: 2026-03-24 10:00:00 +0800
categories: [AI, Agent]
tags: [agent, mcp, tool-design, claude-code]
---

# Seeing like an Agent：大模型工具的设计指南

> *"One of the hardest parts of building an agent harness is constructing its action space."*
> *构建 Agent 框架最难的部分之一，是设计它的动作空间。*
> *—— Anthropic, Lessons from Building Claude Code: Seeing like an Agent*

> *"Claude Code currently has ~20 tools, and we are constantly asking ourselves if we need all of them. The bar to add a new tool is high, because this gives the model one more option to think about."*
> *Claude Code 目前有约 20 个工具，我们一直在问自己是否每一个都有必要。添加新工具的门槛很高，因为每多一个工具，就多给模型一个需要思考的选项。*

> *"Experiment often, read your outputs, try new things. See like an agent."*
> *多做实验，审视输出，尝试新方法。学会站在 Agent 的视角看问题。*

---

## 根源：大模型的输出是概率性的

要理解为什么给大模型设计工具和给程序设计 API 是两件完全不同的事，需要先回到一个根本事实：**大模型的本质是一个概率函数。**

它的每一次输出都是从概率分布中采样的结果——不是确定性的计算，而是发散的、不完全可预测的生成。同样的输入，不同的采样可能产生不同的行为：选择不同的工具、传入不同的参数、走向不同的分析路径。

这意味着，当你给大模型 50 个工具时，你不是在给它 50 个确定的选项，而是在给它一个巨大的概率解空间。模型需要在这个空间里采样出"下一步该做什么"，选项越多，分布越分散，每一步偏离最优路径的概率就越高。

**工具设计的本质，是约束大模型的解空间。**

少量、正交、边界清晰的工具，把模型的选择空间从无序的高维分布压缩到一个低维、结构化的子空间里。在这个子空间里，模型的每一次采样都更可能落在有效区域内。

这不是限制模型的能力，而是让它的能力更有效地释放。就像给河流修堤坝——不是阻止水流，而是让水流到该去的地方。

Anthropic 在构建 Claude Code 的过程中反复踩坑、反复重构，最终沉淀出一句话：**See like an agent**——站在模型的位置看问题。当你理解了模型是一个概率系统，你就知道工具设计的目标不是"暴露更多能力"，而是"让每一步选择都尽可能确定"。

以下四条原则，都是这一根源的具体推论。

---

## 一、输出克制：给对的数据，给够用的量

> *"To put myself in the mind of the model I like to imagine being given a difficult math problem. What tools would you want in order to solve it?"*
> *为了站在模型的角度思考，我喜欢想象自己面对一道数学难题。你会想要什么工具来解题？*

工具的输出有两个维度需要克制：**量**和**性质**。

**量的克制**——每次只给够做下一个判断的数据。大模型的 context window 是固定的，每多塞一段无用信息，有用信息的权重就下降一点。一个工具返回 10 万行结果，和返回 0 行结果一样没用——前者淹没，后者缺失。

Anthropic 在构建 Claude Code Guide 子代理时就遇到了这个问题：

> *"This worked but we found that Claude would load a lot of results into context to find the right answer when really all you needed was the answer."*
> *这个方法有效，但我们发现 Claude 会把大量结果加载到上下文中来寻找正确答案，而实际上你只需要那个答案本身。*

模型为了找到一个答案，把大量搜索结果全部加载到上下文里。信息量是够了，但大部分是噪声，反而干扰了判断。最终的解决方案是构建一个专门的子代理，带有精确的搜索指令和返回规范——不是给更多数据，而是给更精准的数据。

正确的粒度是：模型看完输出后，能决定"下一步查什么"。工具应该有分页、有上限、有摘要，让模型在多轮对话中逐步深入。

**性质的克制**——提供数据，不提供结论。

> *"By giving Claude a Grep tool, we could let it search for files and build context itself."*
> *通过给 Claude 一个 Grep 工具，我们可以让它自己搜索文件、自己构建上下文。*

Claude Code 的搜索设计经历了一次关键转变：从 RAG 自动喂上下文，到给 Grep 工具让模型自己搜。区别不在于技术实现，在于谁来做判断。RAG 替模型做了"什么信息是相关的"这个判断，但这个判断可能是错的。而 Grep 只是一个搜索工具，模型自己决定搜什么、怎么搜、搜到的结果是否有用。

> *"This is a pattern we've seen as Claude gets smarter, it becomes increasingly good at building its context if it's given the right tools."*
> *我们观察到一个规律：随着 Claude 变得更聪明，只要给它合适的工具，它构建自身上下文的能力就越来越强。*

工具的职责是快速取回数据。"这个数据意味着什么"——这个判断留给使用者。

自动化的"检测"和"识别"看似省事，实际上有两个问题：第一，它可能是错的，但调用方已经跳过了验证环节；第二，它把中间过程黑盒化了，调用方无法在中途调整方向。给模型原子级的操作——搜索、读取、过滤——让它自己组合出分析策略。

---

## 二、一步完成：无状态，无前置，无清理

> *"While Claude could just ask questions in plain text, we found answering those questions felt like they took an unnecessary amount of time. How could we lower this friction?"*
> *虽然 Claude 可以用纯文本提问，但我们发现回答这些问题花了不必要的时间。如何降低这种摩擦？*

Anthropic 在设计 AskUserQuestion 时反复强调降低摩擦。同样的道理适用于工具调用：每多一步有状态的操作，就多一层摩擦。

Claude Code 早期的 TodoWrite 工具就是一个有状态设计的教训。模型需要在开头写入 Todo 列表，执行过程中逐项勾选，还要时刻维护列表的一致性。为了防止模型忘记列表内容，团队甚至每 5 轮插入一次系统提醒。

> *"Being sent reminders of the todo list made Claude think that it had to stick to the list instead of modifying it."*
> *不断收到待办列表的提醒，反而让 Claude 觉得自己必须严格遵守列表，而不是根据实际情况去修改它。*

状态管理反而束缚了模型——它把维护列表当成了目标，而不是完成任务。后来替换为 Task Tool，让子代理之间直接通信、共享更新，模型不再需要独自维护一个全局状态。

有状态的工具要求调用方记住上下文——先打开、再操作、再关闭。对程序来说这不是问题，对大模型来说每多记一个状态，就多一个遗忘和出错的机会。模型可能忘记关闭 session，可能把两个 session 的 ID 搞混，可能在第三步才发现第一步的参数传错了。

理想的工具是无状态的：一条命令传入所有必要参数，返回一个完整结果。不需要前置操作，不需要后续清理。如果必须有状态（比如缓存），让工具自己管理，对调用方透明。

---

## 三、少而正交：每个工具做一件事，做完整

> *"Claude Code currently has ~20 tools, and we are constantly asking ourselves if we need all of them. The bar to add a new tool is high, because this gives the model one more option to think about."*
> *Claude Code 目前有约 20 个工具，我们一直在问自己是否每一个都有必要。添加新工具的门槛很高，因为每多一个工具，就多给模型一个需要思考的选项。*

这是 Anthropic 的原话，值得反复读。Claude Code 是一个功能极其丰富的产品，但它只有约 20 个工具，而且团队在持续质疑是否都需要。

一个简单的检验标准：**如果两个工具能完成同一件事，说明有一个是多余的。如果一件事需要三个工具顺序执行才能完成，说明这件事应该是一个工具。**

前者意味着工具之间语义重叠，模型不知道该选哪个。后者意味着工具的粒度切错了——把一个完整的操作人为拆成了流水线，强制模型维护中间状态和调用顺序。Anthropic 在 AskUserQuestion 的第一次尝试中就犯了类似的错误：把提问功能塞进 ExitPlanTool，一个工具同时做两件事，模型困惑了。反过来也一样——如果把一件事拆成三个工具，模型同样会困惑。

> *"Paper would be the minimum, but you'd be limited by manual calculations. A calculator would be better, but you would need to know how to operate the more advanced options. The fastest and most powerful option would be a computer, but you would have to know how to use it to write and execute code."*
> *纸笔是最低配置，但你会受限于手工计算。计算器更好，但你需要会用高级功能。最快最强的是电脑，但你得会写代码。*

> *"This is a useful framework for designing your agent. You want to give it tools that are shaped to its own abilities."*
> *这是设计 Agent 的一个实用框架：给它与自身能力匹配的工具。*

工具的数量和形态应该匹配模型的能力。大模型擅长文本模式匹配、多步推理、在多轮对话中逐步缩小范围。所以应该给它搜索、过滤、读取这些原子操作，让它自己组合出复杂的分析流程。大模型不擅长理解黑盒——一个"一键完成"的工具，模型不知道内部发生了什么，无法判断结果是否可信，也无法在中间环节调整方向。

模型的推理能力是有限的。它花在选择工具上的 token，就是没花在解决问题上的 token。

---

## 四、观察迭代：让模型的行为告诉你答案

> *"Most importantly, Claude seemed to like calling this tool and we found its outputs worked well."*
> *最重要的是，Claude 似乎很喜欢调用这个工具，而且输出效果很好。*

> *"Even the best designed tool doesn't work if Claude doesn't understand how to call it."*
> *即使是设计最好的工具，如果 Claude 不知道怎么调用它，也毫无用处。*

传统软件开发的验证方式是：写测试、跑 benchmark、做 code review。工具设计完成就上线，用户不用就是用户的问题。

大模型时代，验证方式变了。工具好不好，不是开发者说了算，是**模型的行为**说了算。

Anthropic 在评价 AskUserQuestion 时，用的标准不是"架构是否优雅"，而是"Claude seemed to like calling this tool"。模型喜不喜欢用，用的结果好不好——这是后验反馈，比任何先验设计都重要。

Anthropic 在设计 AskUserQuestion 时试了三个方案：

1. 把问题参数塞进 ExitPlanTool——模型困惑了，因为一个工具同时做两件事
2. 让模型输出特殊格式的 markdown——模型经常格式出错
3. 做一个独立的 AskUserQuestion tool——模型"seemed to like calling this tool"

最终生效的方案不是最优雅的，而是最符合模型认知模式的。工具设计的终极标准不是"架构是否合理"，而是"模型是否自然地知道什么时候该调、怎么调"。

具体来说，上线一个工具后应该关注：

**模型是否主动调用它？** 如果一个工具很少被使用，不是模型笨，是工具有问题。可能是名字不够明确——模型看到 `build_dep_tree_from_slice` 不知道这是干什么的，但看到 `taint` 就知道是反向追踪。可能是功能和其他工具重叠——模型不确定该用哪个，干脆都不用。也可能是使用前提太复杂——需要先调另一个工具拿到 ID，模型觉得麻烦就绕过了。

**模型调用后的结果是否有效？** 如果模型每次调完一个工具都要再调两三个补充工具才能完成任务，说明第一个工具的输出粒度不对。如果模型拿到返回值后"困惑"了（开始重复调用、换不同参数试、或者干脆放弃），说明返回格式有问题。

**模型是否以你预期的方式使用它？** 如果模型把一个查询工具当成了验证工具，或者反过来，说明工具的语义边界不清晰。工具的名字、描述、参数名，对模型来说就是它理解这个工具的全部信息。

这些反馈无法在设计阶段获得。只能上线、观察、迭代。**先验设计给你一个起点，后验反馈告诉你方向。**

---

## 结语：今天的补丁就是明天的累赘

> *"As model capabilities increase, the tools that your models once needed might now be constraining them. It's important to constantly revisit previous assumptions on what tools are needed."*
> *随着模型能力的提升，曾经需要的工具可能正在限制它们。不断回顾"需要哪些工具"这一假设，至关重要。*

大模型的能力迭代速度远快于传统软件。今天模型需要你拆分成三步的操作，下个版本可能一步就能完成。今天需要的输出格式约束，下个版本的模型可能天然就能遵守。如果工具设计一成不变，那些为旧模型做的适配会逐渐堆积成技术债——不是代码层面的债，而是认知层面的债：**工具在限制模型本可以发挥的能力。**

每当模型能力更新时，应该重新审视当前的工具集：哪些工具还被需要？哪些工具的限制可以放宽？哪些为旧模型打的补丁该拆掉了？

唯一不变的是那句话：**See like an agent。**

站在模型的位置，用它的眼睛看你设计的工具。你会如何使用这些工具？你需要记住多少状态？你能处理多大的输出？你面对 25 个选项时会不会犹豫？

如果你自己都觉得累，模型一定也觉得累。
