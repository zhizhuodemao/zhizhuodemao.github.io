---
title: AI 能抓包了？Android 全链路逆向分析之 - 自研抓包 MCP
date: 2025-01-31 18:00:00 +0800
categories: [AI, 逆向]
tags: [ai, android逆向, mcp, 抓包, 安全研究]
---

## 一、前言

做过 Web 端和 App 端逆向的人都知道，Web 端的逆向分析工具可以说非常统一 —— Chrome 自带的 DevTools 就够了，既能抓包，又能调试，还能替换资源、进行 Trace。虽然谷歌未必是为了我们逆向分析才做得这么好，但毫无疑问，Web 端的逆向门槛确实低了很多。

对于 Web 端的逆向分析，之前我已经开发了基于 chrome-devtools-mcp 的 [js-reverse-mcp](https://github.com/zhizhuodemao/js-reverse-mcp)，基本上可以完成从抓包到打断点、到执行 JS 的全链路逆向分析，已经相当完善了。

然而 App 端就完全不同了。

App 端的逆向分析工具**碎片化严重**：从抓包到静态分析到动态调试，从 Java 层到 So 层，每个环节都有自己的工具，而且往往多个工具功能交叉。比如抓包，Fiddler、Charles、Reqable、r0capture、eCapture 其实都能完成，各有各的优势。

这也意味着，全链路 AI 进行移动端逆向分析，很难有一个统一的解决方案。

目前社区在抓包之后的分析环节，已经有了各自的方案：Java 层有 jadx-mcp，So 层有 ida-pro-mcp。但在**定位入口**这个环节，更多还是凭借抓包工具加上各自的经验。

**这其实是全链路逆向分析的最后一块拼图。**

补上这块拼图，AI 自己独立分析一个 App 才真正成为可能。这也是我研发抓包 MCP 的初衷。

---

## 二、技术选型

基于上述背景，我是如何考虑技术选型的呢？

首先排除几个方案：
- **Fiddler、Charles、Reqable**：不开源，也没有自己的 MCP，无法使用
- **r0capture**：依赖 Frida，容易产生额外特征，且不同设备对 Frida 的兼容性不同
- **eCapture**：很好，只需要 Root 就可以抓包，但问题在于只支持内核比较新的手机

除此之外，eCapture 还有一个**核心问题**：

### LLM 只能吃文本 (JSON)，吃不下二进制流

| 对比维度 | eCapture | Mitmproxy |
|---------|----------|-----------|
| **数据格式** | 碎片化的 TCP/TLS 字节流，需要 TShark 重组解析 | 原生 Python 对象，`flow.request.json()` 直接拿结构化数据 |
| **请求-响应关联** | 很难把请求 A 和响应 B 精准对应（尤其是并发时）| 天生知道哪个响应对应哪个请求 |
| **对 AI 友好度** | 低，需要大量预处理 | 高，零损耗直接喂给 LLM |

简单说：
- **eCapture 是"窃听器"**：抓到的是原始字节流，请求和响应是分离的
- **Mitmproxy 是"代理"**：天然就是完整的 Request-Response 对

基于以上原因，最终选择 **Mitmproxy** 作为抓包的基础设施。

---

## 三、核心设计

选好了 Mitmproxy 作为基础设施，接下来就是核心问题：**如何让大模型高效地分析抓到的流量？**

这里有三个关键问题需要解决：
1. 请求数据如何组织，才能让大模型理解？
2. 请求量太大怎么办，如何筛选？
3. 抓包进程和 MCP 进程如何共享数据？

下面逐一分析。

### 3.1 请求对象设计

#### 问题：请求量大，上下文爆炸

做过 App 逆向的都知道，随便打开一个 App 操作几分钟，轻松产生几百个请求。

算笔账：
- 一个普通请求（含请求头、响应头、响应体）：**10KB ~ 100KB**
- 操作 App 5 分钟产生 200 个请求
- 全量数据：200 × 50KB = **10MB**

而 Claude 的上下文窗口是有限的，即使是 200K token 的模型，也撑不住这么大的数据量。更别说 token 是要花钱的。

**所以，不能把所有请求的完整数据一股脑喂给大模型。**

#### 解决：参考谷歌 CDP 的设计

怎么解决？这里可以参考谷歌的思路。

通过分析 Chrome DevTools Protocol 的设计可以发现，谷歌的方案非常巧妙：

1. **列表只展示元数据**：请求列表只包含 ID、URL、状态码、大小等核心信息
2. **详情按需获取**：通过请求 ID 获取完整的请求头、请求体、响应体

这样，大模型先看"目录"，定位到感兴趣的请求后，再获取"正文"。

#### 方案：TrafficRecord 对象

按照这个思路，我们把每个请求封装成一个对象：

```python
@dataclass
class TrafficRecord:
    # 基础标识
    id: str                    # "req-1"
    timestamp: float           # Unix 时间戳

    # 请求信息
    method: str                # GET, POST, PUT...
    url: str                   # 完整 URL
    domain: str                # 域名

    # 响应信息
    status: int                # 状态码 200, 404...
    resource_type: str         # XHR, Document, Image...

    # 统计
    size: int                  # 响应大小（字节）
    time_ms: float             # 耗时（毫秒）

    # 详细数据（按需获取）
    request_headers: dict      # 请求头
    request_body: bytes        # 请求体
    response_headers: dict     # 响应头
    response_body: bytes       # 响应体

    # 其他
    timing: dict               # 时间详情
    error: str | None          # 错误信息
```

大模型调用 `traffic_list` 时，只返回元数据（ID、URL、状态码等），每条记录只有几百字节。定位到目标请求后，再通过 `traffic_get_detail` 获取完整信息。

**效果：200 个请求的元数据列表，压缩到几十 KB，大模型轻松处理。**

---

### 3.2 筛选策略设计

#### 问题：用户不知道要找哪个请求

解决了数据组织的问题，还有一个更现实的问题：**用户根本不知道要找哪个请求。**

想想看，用户来问的往往是这样的：
- "帮我找酷安 App 的搜索接口"
- "分析一下这个 App 的登录请求"
- "看看这个 App 是怎么加密的"

用户知道的是**业务需求**，不知道的是**具体哪个 URL**。如果用户知道是哪个 URL，还用得着这个 MCP 吗？

但问题是，一个 App 可能有：
- 50 个不同的域名（主站、CDN、统计、广告...）
- 200 个请求
- 其中真正相关的可能就 3~5 个

**大模型不可能把 200 个请求全看一遍，那样既慢又贵。**

#### 解决：像人类一样分层筛选

那人类是怎么做的？

有经验的逆向工程师拿到抓包数据后，通常是这样操作的：
1. **先按域名过滤**：只看 `api.xxx.com`，过滤掉 CDN、统计、广告
2. **再按类型过滤**：只看 XHR 请求，过滤掉图片、CSS、JS
3. **然后搜索关键词**：在 URL 或响应体里搜 `search`、`login`
4. **最后看详情**：找到目标请求，查看完整参数

这就是**分层筛选**的思路：先粗筛缩小范围，再精筛定位目标。

#### 方案：三层筛选策略

参考谷歌 CDP 的设计范式，加上我对 App 逆向的经验，最终设计出这套方案：

```
全量流量 → 粗筛（元数据）→ 精筛（内容搜索）→ 目标请求
```

#### 第一层：粗筛（基于元数据）

通过 `traffic_list` 工具，按元数据快速过滤：

| 筛选条件 | 说明 | 示例 |
|---------|------|------|
| `filter_domain` | 域名（支持通配符）| `*api*.coolapk.com` |
| `filter_type` | 资源类型 | `XHR`（API 通常是 XHR）|
| `filter_status` | 状态码 | `2xx`, `4xx`, `500-599` |
| `filter_url` | URL 关键词 | `search`, `login` |

**逆向经验：**
- App 的 API 请求 90% 是 `XHR` 类型
- 域名通常包含 `api`、`gateway`、`service`
- 搜索接口 URL 常含 `search`、`query`、`find`

#### 第二层：精筛（基于内容搜索）

通过 `traffic_search` 工具，搜索请求/响应内容：

```python
traffic_search(
    keyword="搜索词",            # 用户输入的搜索内容
    search_in=["response_body"], # 搜索范围
    method="POST",               # 限定方法
    domain="%coolapk%"           # 限定域名
)
```

搜索范围支持：
- `url` - URL 中的关键词
- `request_headers` - 请求头（如 Token、签名）
- `request_body` - 请求参数
- `response_headers` - 响应头
- `response_body` - 响应内容

#### 第三层：大响应分片读取

响应体太大怎么办？分片读取，避免上下文溢出：

```python
traffic_read_body(
    request_id="req-5",
    field="response_body",
    offset=0,        # 起始位置
    length=4000      # 每次读 4KB
)
```

#### 实际流程示例

**用户：** "帮我找酷安 App 的搜索接口"

**AI 执行：**

```
1. traffic_list(filter_domain="%coolapk%", filter_type="XHR")
   → 缩小到 50 条 API 请求

2. traffic_search(keyword="search", search_in=["url"])
   → 找到 3 条 URL 含 search 的请求

3. traffic_search(keyword="用户搜索的关键词", search_in=["request_body"])
   → 精确定位到目标请求

4. traffic_get_detail("req-23")
   → 获取请求头、参数等元数据
```

**设计原则：先缩小范围，再精确定位，避免全量读取。**

---

### 3.3 跨进程数据共享

#### 问题：为什么是两个进程？

你可能会问：为什么抓包和 MCP 要分成两个进程？合成一个不行吗？

技术上当然可以，但从用户体验考虑，分开更好：

**代理进程（用户手动启动）：**
- 用户在终端运行 `uv run android-proxy-start`
- 能看到实时输出，确认代理启动成功
- 能看到手机是否连上了（有请求进来）
- 出问题了能看到错误日志

**MCP 进程（Claude 自动调用）：**
- 用户在 Claude Desktop 里提问
- Claude 自动调用 MCP 查询流量
- 用户无感知，只看到分析结果

这样分离的好处：
- **可调试**：代理挂了，用户能立刻在终端看到
- **可验证**：用户能确认手机确实连上了代理
- **解耦**：代理重启不影响 MCP，MCP 重启不影响代理

#### 问题：两个进程如何共享数据？

但分成两个进程后，就有个问题：**代理抓到的数据，MCP 怎么读取？**

几个方案：
- 用 Redis？难道要求每个用户都装 Redis？
- 用内存共享？两个独立进程无法直接共享内存
- 用文件？JSON 文件频繁读写性能差，而且没法做复杂查询

**所以我们的方案是：SQLite。**

#### 方案对比

| 方案 | 优点 | 缺点 |
|-----|------|------|
| Redis | 高性能、支持订阅 | 需要额外安装，用户门槛高 |
| 内存共享 | 快 | 两个独立进程无法直接共享 |
| 文件(JSON) | 简单 | 频繁读写性能差，无法结构化查询 |
| **SQLite** | **零依赖、结构化查询** | 写入需要锁 |

#### 架构图

```
┌─────────────────┐                    ┌─────────────────┐
│  代理进程        │                    │  MCP 进程        │
│  (mitmdump)     │                    │  (Claude 调用)   │
│                 │      SQLite        │                 │
│  捕获流量 ──────────→  写入  ←──────────  查询/搜索     │
│                 │  /tmp/traffic.db   │                 │
└─────────────────┘                    └─────────────────┘
```

#### 为什么选 SQLite？

1. **零依赖** - Python 内置 `sqlite3`，无需安装任何东西
2. **文件即数据库** - 跨进程天然支持，读同一个文件即可
3. **结构化查询** - SQL 支持复杂筛选（域名、状态码、内容搜索）
4. **并发友好** - 支持多读单写，MCP 查询不阻塞代理写入

**为什么不用 Redis？**

> 让用户装 Redis 只为了一个抓包工具？杀鸡用牛刀。SQLite 零配置，开箱即用。

---

## 四、总结与展望

基于以上设计，我们完成了基于 Mitmproxy 的 Android 抓包 MCP：

- **请求对象设计**：参考 CDP，元数据 + ID 定位，避免上下文爆炸
- **分层筛选策略**：粗筛 + 精筛 + 分片读取，精准定位目标请求
- **跨进程数据共享**：SQLite 零依赖，开箱即用

在这个基础上，构建一个专属于 Android 逆向的 AI Agent 不再是难题。

结合 jadx-mcp（Java 层）、ida-pro-mcp（So 层），全链路 AI 逆向分析的拼图已经完整。

**未来，逆向分析可能真的只需要大模型就够了。**

---

**项目地址：** [https://github.com/zhizhuodemao/android_proxy_mcp](https://github.com/zhizhuodemao/android_proxy_mcp)
