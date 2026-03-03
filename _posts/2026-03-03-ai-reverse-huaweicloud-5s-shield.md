---
title: 使用AI分析华为云5s盾（CT2-WAAP）检测逻辑
date: 2026-03-03 00:00:00 +0800
categories: [AI, 逆向]
tags: [ai, js逆向, waf, 安全研究, mcp]
---

> 本文仅供安全研究和学习交流，请勿用于非法用途。

## 一、背景

### 1.1 华为云5s盾是什么
访问某些政府/企业网站时，会弹出"请稍候…正在验证您是否是真人"的5秒等待页面，这就是华为云的 WAF 产品 **CT2-WAAP**。

服务端返回 **412 Precondition Failed**，页面加载两个JS：一个动态配置脚本（每次不同），一个约565KB的静态bundle。JS自动完成验证后设置cookies，页面reload拿到200。

### 1.2 分析目标
- 搞清楚5s盾的**检测逻辑**——它到底检测了什么？
- 理解**风险评估体系**——打分规则是怎样的？
- 搞清楚**服务端能看到什么**——6个cookie里哪些信息真正被校验？

### 1.3 工具
- **Chrome DevTools MCP**：自己写的 MCP Server，让 AI 直接操作 DevTools（断点、Hook、网络请求分析）
- **Claude Code**：AI 驱动分析、写解混淆脚本、写Python实现

---

## 二、Bundle解混淆

bundle 是标准的 OB 混淆器产物，6层混淆叠加：

| 层 | 手法 |
|----|------|
| 字符串数组 | 所有字符串提取到大数组，索引函数访问 |
| 嵌套包装 | 3-4层函数转发链 |
| 操作符代理 | `a+b` → `obj['key'](a,b)` |
| 常数混淆 | `107` → `0x117b+0x25*-0x85+0x229` |
| 控制流平坦化 | while+switch状态机 |
| 死代码 | 永真/永假虚假分支 |

用 Babel AST 写了6步pipeline自动还原，这部分比较常规就不展开了。重点聊还原后发现的检测逻辑。

---

## 三、风险评估体系

解混淆后发现 bundle 里有一个完整的**风险评估器**，这是最有意思的部分。

### 3.1 评分规则

| 检测项 | 加分 | 说明 |
|--------|------|------|
| Chrome版本未通过能力抽检（桌面） | **+60** | UA声明的版本与实际能力不匹配 |
| Chrome版本未通过能力抽检（移动） | +30 | 移动端降权 |
| 声明版本严重低于能力下界（差≥10） | **+80** | 版本差异太大 |
| 声明版本略低于能力下界（差<10） | +40 | 轻微不一致 |
| WebGL软件渲染（桌面） | +30 | 无头浏览器特征 |
| WebGL软件渲染（移动） | +15 | 降权 |
| 非Chrome伪装Chrome | +40 | UA伪造 |

**降权系数**：高可信移动端 ×0.8，中可信 ×0.9

**阈值**：0~40 low，40~75 medium，75+ high

### 3.2 Chrome能力抽检

不只看UA字符串，还检测浏览器的**实际API能力**：

```
capabilityFloor: 能力检测推断的最低Chrome版本（如120）
declaredVersion: UA声明的Chrome版本（如131）
```

如果你UA写 Chrome/131，但浏览器实际只支持 Chrome 100 级别的API，就会触发版本不一致。**伪造UA容易，伪造浏览器能力很难**——这个思路是对的。

### 3.3 WebGL渲染检测

检测 WebGL 是硬件渲染还是软件渲染。Headless Chrome 通常是软件模拟的 WebGL，桌面端直接+30分。

### 3.4 移动端降权

几乎所有检测项对移动端都有降权，推测是因为移动端浏览器环境多样性大，误判风险高。

---

## 四、Cookie设计与服务端校验

bundle 生成6个 `CT_*` cookie，但服务端能读到的信息差别很大：

### 4.1 Cookie一览

| Cookie | 内容 | 加密 | 服务端能读 |
|--------|------|------|-----------|
| CT_g9aa1ec2 | UUID + 校验和 | 明文 | 校验格式 |
| CT_f7ba0eb8 | MD5(指纹JSON) + 校验后缀 | Hash | 只有hash |
| CT_rqu7ab01 | 时间戳 + bundleMD5 | AES | 能解密 |
| CT_6eadf26c | 鼠标/键盘事件计数 | AES | 能解密 |
| CT_e6tzab00 | 时间戳 + UUID | AES | 能解密 |
| CT_f7ba0eb9 | **完整风险评估结果** | AES | 能解密 |

### 4.2 CT_f7ba0eb9：核心cookie

这是服务端拿到的最详细的数据，解密后是一个JSON：

```json
{
  "level": "low",
  "riskScore": 0,
  "reasons": [],
  "detail": {
    "UACH": null,
    "mobile": {"isMobile": false, "confidence": "low"},
    "chrome": {"isChrome": true},
    "capabilityFloor": 120,
    "declaredVersion": 131,
    "webgl": {"level": "normal"}
  },
  "ips": []
}
```

加密方式：`salt(4字符随机) + AES-CBC(风险JSON, salt+密钥片段, salt+密钥片段)`

服务端用配置中的密钥片段 + cookie前4字符作为salt还原AES密钥，解密得到这个JSON。

### 4.3 指纹采集了但不上传

CT_f7ba0eb8 在本地采集了很多指纹：

```
userAgent, language, pixelRatio, deviceMemory, colorDepth,
hardwareConcurrency, resolution, canvas(MD5), WebGL(vendor/renderer),
fonts, audio(sampleRate), plugins...
```

但这些**只算了个MD5就丢了**，服务端拿到的只是一个32位hash + 4位校验后缀。服务端没法知道你的canvas长什么样、WebGL renderer是什么。

这个hash更多是用来做**设备一致性追踪**（同一浏览器多次访问hash一致），而非指纹校验。

---

## 五、设计分析

### 5.1 做得好的地方
- **Chrome能力抽检**：不只看UA，还检测实际API支持，比纯UA校验强
- **WebGL渲染检测**：有效识别无头浏览器
- **移动端降权**：降低误判率，比一刀切好
- **多维度交叉**：版本一致性 + WebGL + 事件计数 + 时间校验

### 5.2 架构短板

**客户端自我评估**：风险分数由JS计算后发给服务端。等于让考生自己改卷子——你告诉它 `riskScore: 0`，它就信了。对比 Cloudflare 由服务端根据原始数据独立评判，这个差距很大。

**密钥全在前端**：AES密钥通过配置脚本明文下发，知道规则就能自己算。

**指纹只传hash**：服务端无法独立验证指纹值是否合理。

**无PoW**：没有需要真实计算量的挑战，不像 Cloudflare Turnstile 有 Proof of Work。

**waf.log是摆设**：一开始以为 `POST /ctct/nwaf/waf.log` 是验证必要步骤，分析后发现它只在异常时上报错误，正常流程根本不调用。

---

## 六、MCP工作流

这次分析全程用了自己写的 **Chrome DevTools MCP Server**，配合 Claude Code。简单聊聊这个工作流的体验。

### 6.1 请求分析

```
list_network_requests → 找到412响应和JS加载
get_network_request(reqid) → 查看请求/响应详情
```

直接在对话里看网络请求，不用来回切 DevTools 面板。

### 6.2 源码搜索 + 断点调试

```
search_in_sources("CT_f7ba0eb9") → 定位cookie生成代码
set_breakpoint_on_text("evaluateRisk") → 断在风险评估函数
get_paused_info → 查看调用栈和作用域变量
evaluate_on_callframe("JSON.stringify(result)") → 查看运行时数据
```

对混淆代码特别有用——静态看容易被绕晕，断点打上去运行时数据一清二楚。AI 能直接在断点处执行表达式查看变量，比手动调试效率高很多。

### 6.3 Hook监控

```
hook_function("XMLHttpRequest.prototype.setRequestHeader")
trace_function("_0x307fe7")  // AES加密函数
```

不用手动注入代码，MCP 直接 hook 目标函数，监控调用参数和返回值。

### 6.4 解混淆 + 算法还原

AI 根据分析结果自动写 Babel AST 解混淆脚本，6步从565KB混淆代码还原出可读代码。然后读懂算法逻辑，直接写Python实现。中间的AES加密、cookie格式这些细节不用人工翻译。

### 6.5 整体效率

从抓包到纯Python跑通200，大概几个小时。大部分时间花在理解业务逻辑上，工具和代码层面 AI 基本兜住了。MCP 的核心价值是**让 AI 拥有了 DevTools 的眼睛和手**，不再局限于静态分析。

---

## 七、总结

华为云5s盾的防护思路没问题——能力抽检、WebGL检测、事件计数都是有效的检测维度。但"**客户端自我评估，服务端直接信任**"这个架构决定了它的上限。

对普通爬虫和 curl 足够了，但对逆向分析几乎是透明的。

| 对比项 | 华为云5s盾 | Cloudflare Turnstile |
|--------|-----------|---------------------|
| 风险评估 | 客户端计算，结论上传 | 服务端独立评判 |
| 挑战机制 | 无 | PoW（Proof of Work） |
| 指纹校验 | 只传hash | 原始数据上传 |
| 密钥管理 | 前端明文下发 | 服务端持有 |
| 动态对抗 | 混淆不变 | 定期更新挑战逻辑 |
