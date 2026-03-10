---
title: unidbg 适合 MCP 吗？
date: 2026-03-10 00:00:00 +0800
categories: [AI, 逆向]
tags: [ai, unidbg, mcp, skill, android逆向]
---

MCP（Model Context Protocol）是当下 AI 工具链的热门话题。当 unidbg 社区引入 MCP 功能后，不少人觉得这是个好方向——让 AI 直接操作调试器，下断点、读内存、单步执行，听起来很美好。

事实上，在 unidbg 官方加入 MCP 之前，笔者就已经尝试过给 unidbg 做 MCP 接入。结果发现问题比想象中多得多，最终放弃了这条路线，转而采用 Claude Code 的 Skill（自定义命令）机制来辅助逆向分析。

本文记录这个过程中的思考和踩坑，探讨为什么 unidbg 这类模拟器场景下，MCP 可能并不是最佳选择。

## 我们最初想做什么

### 理想中的 MCP

设想是这样的：unidbg 启动后暴露一组 MCP 工具，AI 可以：

1. 读写内存和寄存器
2. 下断点、单步执行
3. trace 指令和内存访问
4. 反汇编、搜索内存模式

逆向工程师只需要对话："帮我看看这个函数第三个参数指向什么结构体"，AI 自动下断点、读数据、推理分析。

### unidbg 官方的实现

后来 unidbg 官方确实做了这件事——一个完全自研的 HTTP+SSE 服务器（约 2600 行代码），暴露了 50+ 个调试工具：

- **内存操作**：read_memory、write_memory、search_memory
- **寄存器操作**：get_registers、set_register
- **代码分析**：disassemble、assemble、patch
- **断点管理**：add_breakpoint、add_breakpoint_by_offset
- **执行控制**：continue_execution、step_over、step_into
- **追踪**：trace_code、trace_read、trace_write
- **函数调用**：call_function、call_symbol

实现得很完整，本质上是一个 **AI 驱动的 GDB**。

但当我们把它放到实际的逆向分析工作流中一评估，就发现了一系列结构性问题。

## 第一道坎：补环境——MCP 完全无法介入

### unidbg 补环境是什么

用过 unidbg 的人都知道，让一个 SO 跑起来最痛苦的不是逆向分析本身，而是**补环境**——手动模拟 SO 依赖的所有 Java/Android 环境。

unidbg 不是真机，它只模拟 ARM CPU。当 native 代码通过 JNI 回调 Java 层时，unidbg 会抛出异常告诉你："这个 JNI 方法我不认识，你得自己写 mock。"

### 真实的补环境过程

这是一个典型的循环：

```
运行 → 崩溃: "callObjectMethodV not implemented: okhttp3/Interceptor$Chain->request()"
→ 补上这个 mock → 重新编译运行
→ 崩溃: "callObjectMethodV not implemented: okhttp3/Request->url()"
→ 补上这个 mock → 重新编译运行
→ 崩溃: "callObjectMethodV not implemented: okhttp3/HttpUrl->toString()"
→ ...（重复 20-30 次）
```

一个典型的网络请求签名 SO，最终需要 mock 的 JNI 接口清单可能是这样的：

```java
// 网络库调用链
"okhttp3/Interceptor$Chain->request()"
"okhttp3/Request->url()"
"okhttp3/Request->method()"
"okhttp3/Request->headers()"
"okhttp3/Request->body()"
"okhttp3/Request->newBuilder()"
"okhttp3/Request$Builder->addHeader()"
"okhttp3/Request$Builder->build()"
"okhttp3/HttpUrl->toString()"
"okhttp3/HttpUrl->encodedPath()"
"okhttp3/HttpUrl->encodedQuery()"
"okhttp3/Headers->size()"
"okhttp3/Headers->name()"
"okhttp3/Headers->value()"

// IO 缓冲区
"okio/Buffer-><init>()"
"okio/Buffer->writeString()"
"okio/Buffer->readByteArray()"
"okio/Buffer->clone()"
"okio/Buffer->read()"

// Android 系统 API
"android/content/Context->getSharedPreferences()"
"android/content/SharedPreferences->getString()"

// 文件系统模拟
"/proc/self/status"  → 需要伪造 TracerPid: 0（反调试）
"/proc/self/cmdline" → 需要返回正确的包名

// 总计可达 200-300 行 Java mock 代码
```

每个 mock 还不是简单返回 null 就行。比如 `SharedPreferences->getString()` 需要根据 key 返回特定的值（密钥、配置等），`Headers->name()/value()` 需要维护一个完整的请求头列表并按索引返回。有些 mock 之间还有状态依赖——`Buffer->writeString()` 写入的数据，后面 `Buffer->readByteArray()` 要能读出来。

### 为什么 MCP 处理不了补环境

MCP 的工具集全部是**运行时调试工具**——read_memory、set_register、add_breakpoint 等等。但补环境需要的是：

**1. 修改 Java 源码**

MCP 没有任何工具能编辑 `.java` 文件。当 SO 崩溃在一个未实现的 JNI 方法上时，你需要在继承 `AbstractJni` 的测试类里加一个 `case` 分支。这是纯粹的代码编辑操作，不是调试操作。

**2. 触发编译**

改完代码后需要 `mvn compile` 或 IDE 增量编译。MCP 不接入构建系统。

**3. 读取编译/运行报错**

unidbg 的报错信息打印在 Java 的 stdout/stderr 上，不是在调试器控制台里。MCP 看不到这些输出。

**4. 理解 JNI 语义**

补环境不仅是"返回一个值"，还需要理解 SO 期望的语义。比如：

```java
// SO 调用 chain.proceed(request) 期望拿到 Response
// 如果返回 null，后续代码就 NPE 了
// 必须返回一个合法的 Response mock
case "okhttp3/Interceptor$Chain->proceed(Lokhttp3/Request;)Lokhttp3/Response;":
    return responseClass.newObject(null);
```

这需要理解 OkHttp 的 API 设计，不是简单的内存读写能解决的。

**5. 反复重启**

补环境的循环是"改一个 mock → 重启整个模拟器 → 看下一个报错"。每次都是完整的进程生命周期。MCP 连接的是一个运行中的调试会话，一旦进程结束，MCP 会话也就断了。

### 这个阶段真正需要的是什么

需要的是一个能**读报错→改代码→触发编译→重新运行**的循环。这恰好是 AI 代码编辑工具（Claude Code、Cursor 等）的核心能力，而不是 MCP 调试工具的能力。

## 第二道坎：算法还原中的试错

### hook 点经常选错

SO 跑通后进入逆向分析阶段。核心工作是"下断点捕获数据→验证算法假设"。

但 hook 点经常选错，这是常态：

1. 先 hook 地址 A → 发现第一次调用时寄存器指向的数据是空的
2. 改成只看第二次调用 → 发现数据格式不对
3. 换一个地址 B → 发现是解密后的数据，已经过了密钥设置阶段
4. 回头换地址 C → 终于拿到正确的数据

最终写出来的代码可能是这样的：

```java
debugger.addBreakPoint(module.base + 0xABCDE, new BreakPointCallback() {
    int callCount = 0;
    @Override
    public boolean onHit(Emulator<?> emulator, long address) {
        if (callCount == 0) {  // 只在第一次调用时捕获
            RegisterContext ctx = emulator.getContext();
            long x2 = ctx.getLongArg(2);
            byte[] keyCtx = emulator.getBackend().mem_read(x2, 0xF0);
            // 解析轮密钥...
        }
        callCount++;
        return true;
    }
});
```

### MCP 的致命问题：不可回溯

```
AI: add_breakpoint(地址A) → continue → 断点命中
AI: read_memory(x2, 240) → "数据是空的"
AI: "可能 hook 早了，continue 等第二次命中"
AI: continue → 程序直接跑完了，没有第二次
AI: "需要重新开始... 但我没有重启模拟器的能力"
→ 卡住，需要人工干预
```

或者更常见的：

```
AI: 断点命中 → read_memory → 拿到一堆数据
AI: "我觉得应该看地址 B 而不是这里"
AI: remove_breakpoint → add_breakpoint(地址B) → continue
→ 但执行已经过了地址 B，这次跑不到了
→ 又卡住了
```

unidbg 模拟器每次运行都是从头开始：初始化 → 加载 SO → JNI_OnLoad → 调用目标函数 → 结束。整个生命周期可能只有几秒。**错过一个断点，就得重头再来。但 MCP 没有"重启模拟器"的能力。**

而代码是**声明式**的——`callCount == 0` 写一次，改个数字重跑 3 秒搞定。

### 代码方式的迭代成本

```
改一行 callCount == 1 → 重新编译运行 → 3 秒拿到结果
改一个地址 0xABCDE → 0xABCF0 → 3 秒拿到结果
多加一行 ctx.getLongArg(3) → 3 秒拿到结果
```

每次迭代成本极低，而且之前所有的 mock 环境、初始化逻辑都在代码里，不会丢失。

## 第三道坎：算法验证——MCP 没有计算能力

### 逆向分析的本质是"验证假设"

逆向不是"看数据"，而是"提出假设→验证→修正"的循环。

比如你怀疑 SO 里有一段魔改 AES，验证方式是：

```java
// 假设：SO 使用自定义 Rcon 做 key expansion
// 验证：本地实现 key expansion → 对比 SO 的轮密钥

static int[] keyExpansion(byte[] key) {
    int[] w = new int[44];
    for (int i = 0; i < 4; i++)
        w[i] = getU32(key, i * 4);
    for (int i = 4; i < 44; i++) {
        int temp = w[i - 1];
        if (i % 4 == 0)
            temp = subWord(rotWord(temp)) ^ CUSTOM_RCON[i / 4 - 1];
        w[i] = w[i - 4] ^ temp;
    }
    return w;
}
```

然后还有 InvMixColumns、AES-CBC 解密、PKCS#7 验证... 总计 200 行纯算法代码。

### MCP 的困境

AI 通过 `read_memory` 拿到轮密钥的 hex dump 后，需要在"脑中"做 GF(2^8) 乘法、矩阵运算、S-box 查表... 这不现实。最终 AI 还是会说："我帮你写个 Python 脚本验证吧"——**又绕回了代码生成的路子**。

MCP 工具集里没有"执行任意计算"的工具。它能读数据，但不能处理数据。

## 第四道坎：输出爆炸

### trace 的数据量

unidbg 的 trace 输出量极大。一个包含自定义虚拟机的 SO，一次函数调用可能产生**数十万条指令**。

MCP 的 `trace_code` 实现将每条指令都作为一个 JSON 事件推入无界队列：

```java
activeTraceCode = emulator.traceCode(begin, end, (emu, address, insn) -> {
    JSONObject event = new JSONObject(true);
    event.put("event", "trace_code");
    event.put("address", "0x" + Long.toHexString(address));
    event.put("mnemonic", insn.getMnemonic());
    event.put("operands", insn.getOpStr());
    event.put("regs_read", formatRegValues(insn, regsAccess.getRegsRead()));
    server.queueEvent(event);  // 每条指令一个事件，无上限
});
```

然后 `poll_events` 一次性全部拼成字符串返回：

```java
private JSONObject pollEvents(JSONObject args) {
    List<JSONObject> events = server.pollEvents(timeoutMs);
    StringBuilder sb = new StringBuilder();
    for (JSONObject event : events)
        sb.append(event.toJSONString()).append('\n');
    return textResult(sb.toString());
}
```

10 万条指令 × 每条 ~300 字节 = **30MB JSON**。两个问题：

1. **内存爆炸**：`LinkedBlockingQueue` 无界，大范围 trace 直接 OOM
2. **上下文爆炸**：AI 的上下文窗口根本装不下这么多数据

### 代码方案天然解决这个问题

```java
// Java 端：trace 写文件，不占 AI 上下文
traceHook = emulator.traceCode(module.base, module.base + module.size);
traceHook.setRedirect(new PrintStream(new File("trace_output.log")));
```

```python
# Python 端：按需过滤，只看关心的指令
for line in open("trace_output.log"):
    if "str" in line and target_address in line:
        print(line.strip())
```

写文件 + 离线过滤，天然支持任意大小的 trace 数据。

## 我们最终选择的方案：Skill

### 什么是 Skill

Claude Code 的 Skill 是一种自定义命令机制。在项目目录下创建 `.claude/skills/<name>/SKILL.md`，定义一个 prompt 模板。使用时输入 `/skill-name` 触发 AI 按模板生成代码。

核心区别：**MCP 让 AI 操作调试器，Skill 让 AI 写代码。**

### 针对 unidbg 设计的 Skill

**`/hook [地址] [描述]`** — 断点捕获代码生成

```yaml
---
name: hook
description: 生成 unidbg BreakPointCallback 代码
allowed-tools: Read, Write, Edit
argument-hint: [地址 要捕获的数据描述]
---

阅读项目中已有的 hook 脚本作为参考模板。
在用户指定的地址生成 BreakPointCallback：
- 读取相关寄存器（根据 ARM64 调用约定推断参数）
- 读取寄存器指向的内存（根据描述推断数据结构和大小）
- 打印格式化输出（hex dump + 人类可读解释）
- 支持 callCount 过滤（如"只看第 N 次调用"）

目标: $ARGUMENTS
```

**`/trace [范围] [关注点]`** — Trace 捕获 + 分析脚本

```yaml
---
name: trace
description: 生成 unidbg trace 捕获脚本和配套的 Python 分析脚本
allowed-tools: Read, Write, Edit
argument-hint: [函数地址范围 关注的数据]
---

生成两个文件：
1. Java trace 脚本：使用 traceCode/traceRead/traceWrite，输出到文件
2. Python 分析脚本：解析 trace 日志，提取用户关注的模式

目标: $ARGUMENTS
```

**`/verify [算法假设]`** — 算法验证脚本

```yaml
---
name: verify
description: 根据逆向分析假设，生成 Python 验证脚本
allowed-tools: Read, Write, Edit
argument-hint: [算法描述或假设]
---

根据用户描述的算法假设，生成 Python 验证脚本：
- 读取 hook 产出的数据文件（如有）
- 实现算法的 Python 版本
- 与 SO 输出做逐字节对比
- 输出 PASS/FAIL 和差异详情

假设: $ARGUMENTS
```

**`/jni [报错信息]`** — JNI Mock 自动补全

```yaml
---
name: jni
description: 根据 unidbg 运行报错，自动补全缺失的 JNI mock
allowed-tools: Read, Write, Edit
argument-hint: [报错信息或 signature]
---

阅读当前项目的主测试类（继承 AbstractJni 的类），
根据报错的 JNI signature，在对应的 override 方法中补上 case。

根据 signature 推断合理的返回值：
- 返回 String → new StringObject(vm, "合理默认值")
- 返回 Object → dvmClass.newObject(null)
- 返回 int → 合理的默认整数
- 返回 void → 直接 return

报错: $ARGUMENTS
```

### Skill 为什么能覆盖 unidbg 的全流程

| 阶段 | 需求 | MCP | Skill |
|------|------|-----|-------|
| 补环境 | 改 Java 代码、编译、看报错 | 完全无法介入 | `/jni` 自动补全 mock |
| 下断点 | 在指定地址捕获数据 | 能做，但试错成本高 | `/hook` 生成 callback 代码 |
| trace 分析 | 大量指令追踪 + 过滤 | 输出爆炸 | `/trace` 写文件 + Python 过滤 |
| 算法验证 | 实现算法并对比 | 无计算能力 | `/verify` 生成验证脚本 |
| 迭代修改 | 改参数重跑 | 需要重启整个流程 | 改一行代码 3 秒重跑 |

### 效果对比

以"捕获 AES 轮密钥"为例：

**MCP 方式（交互 8 轮，最终还是要写代码）：**

```
AI: add_breakpoint_by_offset("target.so", "0xABCDE")       → 设置断点
用户触发执行（需要人工操作）
AI: poll_events                                              → 断点命中
AI: get_register("x2")                                      → 拿到地址
AI: read_memory(address, 240)                                → 拿到 hex dump
AI: "我看到了轮密钥数据，需要做 key expansion 验证..."
AI: "但我算不了 GF(2^8) 乘法..."
AI: "我帮你写个 Python 脚本吧"                               → 绕回代码生成
```

**Skill 方式（1 轮）：**

```
用户: /hook 0xABCDE "AES 函数入口，X2 指向 key context (0xF0 字节)，解析为 11 轮密钥"
AI: 直接生成完整 Java 文件（断点 callback + 数据解析 + 格式化打印）
用户: 运行 → 拿到结果 → 继续下一步
```

## 代码 vs 操作：根本差异

### 产出物的生命周期

MCP 的产出是**一次性操作**——"在某地址读 X2 寄存器"执行完就消失了。

代码的产出是**永久资产**——一个验证脚本不仅是工具，更是算法还原的完整文档。一个典型项目可能积累近 10 个 Java 调试脚本和 30+ 个 Python 验证脚本，每个都能独立重跑，合在一起就是整个算法还原过程的知识沉淀。

### 声明式 vs 命令式

代码是**声明式**的——"我要在第 1 次调用时读 X2 指向的 0xF0 字节"写一次就行，反复执行结果一致。

MCP 是**命令式**的——每次都要手动走一遍"下断点→continue→poll→读数据"流程。走错一步就废了，而且没有重来的机会。

### unidbg 的特殊性

和 GDB/Frida/IDA 不同，unidbg 是模拟器，每次运行都是从头开始。MCP 适合的是**长生命周期进程**调试——目标进程持续存在，断点错了可以改，状态随时可查。unidbg 的"短生命周期、反复重跑"特性，天然适合代码驱动而不是交互驱动。

## MCP 真正适合的场景

公平起见，MCP 并非一无是处。它适合：

1. **纯探索阶段**——你还完全不知道一个函数在干什么，让 AI 自由下断点、看数据、推理
2. **一次性查看**——"帮我看看 X0 指向的结构体是什么"，不需要保存结果
3. **长生命周期调试器**——GDB attach 到运行中的进程、Frida 注入到 App，这些场景下 MCP 的交互模型才成立
4. **教学场景**——向初学者演示调试流程，交互式比看代码更直观

## 结论

| 维度 | MCP | Skill（代码生成） |
|------|-----|-------------------|
| 补环境 | 完全无法介入 | `/jni` 自动补全 |
| 产出物 | 一次性调试操作 | 可重跑的代码文件 |
| 试错成本 | 错过断点需重走全程 | 改一行重跑 3 秒 |
| 数据持久化 | 随对话消失 | 永久保存在代码中 |
| 计算能力 | 无法做算法验证 | 脚本可做任意计算 |
| 输出量 | 上下文窗口爆炸 | 写文件 + 离线分析 |
| 知识沉淀 | 无 | 每个脚本都是文档 |

unidbg 逆向的核心循环是"写代码→跑→看结果→改代码"，从最初的补环境到最终的算法验证，每一步都是代码驱动。AI 最大的价值不是替你操作调试器，而是替你写代码。

**对于 unidbg 这种"短生命周期、需要补环境、反复重跑、需要算法验证"的场景，让 AI 生成代码（Skill）比让 AI 操作调试器（MCP）更合适。**

把 AI 当作一个极其高效的代码生成器，远比把它当作一个远程调试操作员更有价值。
