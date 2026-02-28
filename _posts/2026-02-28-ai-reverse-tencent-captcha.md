---
title: 使用AI还原腾讯点选验证码算法
date: 2026-02-28 18:00:00 +0800
categories: [AI, 逆向]
tags: [ai, js逆向, vm, 安全研究, 验证码]
---

> 本文仅供安全研究和学习交流，请勿用于非法用途。

## 一、背景与目标

### 1.1 腾讯防水墙（TCaptcha）简介
- 腾讯自研验证码服务，广泛用于 QQ、微信生态及第三方接入
- 本文目标：小黑盒登录页的文字点选验证码（"请依次点击：X X X"）

### 1.2 为什么难
- 核心加密逻辑运行在 **CHAOS VM** 中——一个自定义栈式字节码虚拟机
- tdc.js 每次加载动态变化：模块 ID 随机分配、XTEA key 变化、opcode 映射变化
- 存在大量反调试/诱饵机制：14 个 key 构建函数中只有 4 个是真实的

### 1.3 最终目标
- 纯 Python 还原 `TDC.getData(true)` 生成的 `collect` 加密字段
- 端到端跑通：prehandle → 动态解析 tdc.js → OCR 识别 → 构造 collect → 提交 verify

---

## 二、请求流程分析

### 2.1 验证码触发与交互流程
- 用户登录 → 服务端返回 `show_captcha` → 弹出验证码 iframe
- 验证码运行在跨域 iframe `turing.captcha.gtimg.com` 中

### 2.2 五阶段请求流程
1. **prehandle**：获取 sess、pow_cfg、验证码图片信息
2. **tdc.js 加载**：通过 blob URL 加载到 iframe，注册 `window.TDC`
3. **用户交互**：点击汉字，TDC 持续采集行为轨迹
4. **提交验证**：组装 collect/eks/ans/pow 等参数，POST verify
5. **响应处理**：成功返回 ticket，失败 sess 滚动更新

### 2.3 参数分类
- 需要逆向：`collect`（行为数据加密）、`eks`（环境信息）
- 直接透传：`sess`、`ans`、`pow_answer`
- 简单计算：`tlg`（collect 长度）、`pow_calc_time`

### 2.4 无状态架构
- eks 是 XTEA key 的加密信封，服务端解密 eks 还原 key，无需存储 key-session 映射
- key 生命周期：页面加载时生成，整个会话内不变

---

## 三、CHAOS VM 逆向

### 3.1 VM 架构概览
- 栈式字节码虚拟机，主循环 `for(;!E;) E = U[k[N++]]()`
- 操作码表 U[]（约 58 个有效 handler）、指令流 k[]、程序计数器 N
- 字节码通过 Base64 编码 + 字符串表穿插存储

### 3.2 操作码自动识别
- 从 tdc.js 源码解析 handler 数组，通过代码特征匹配 58 个操作码
- 每次加载 tdc.js 操作码索引会变，但 handler 代码不变

### 3.3 反汇编器实现
- 字节码解码：Base64 → charCode + 字符串表合并 → 指令数组
- 递归下降反汇编：识别函数边界、标注跳转目标、按函数拆分输出
- 产出：214 个函数、24000+ 条指令的完整反汇编

### 3.4 AI 在 VM 逆向中的角色
- 精读反汇编代码，将字节码指令序列还原为伪代码
- 识别关键函数：模块注册、collect 构建、加密入口
- 快速迭代：每次理解一层抽象，逐步深入

---

## 四、XTEA 加密还原

### 4.1 算法定位
- 在指令数组中搜索 delta 常量 `2654435769`（0x9E3779B9）
- 定位 XTEA 核心函数（32 轮加密循环）

### 4.2 JS Number 语义陷阱
- `<<` `>>>` `^` `&` 内部截断 32 位（JS 规范 ToInt32/ToUint32）
- `+` `-` 不截断（JS Number 是 float64）
- sum 不截断，真实累加到 `32 × delta = 84941944608`
- Python 实现必须在位运算函数内截断，加减法不截断

### 4.3 Key 提取：真实 vs 诱饵
- 主加密函数有 14 个子函数，其中只有 4 个写入真实 key buffer
- 判别方法：通过 XTEA 函数的 scope 追踪 key buffer 在父函数中的索引
- 每个 key 函数的打包公式：`key[i] = pack(string, offset)` = `Σ (charCodeAt(s[j]) - offset) << (8*j)`
- 字符串和偏移值编码在字节码中，每次 tdc.js 不同

### 4.4 运行时偏移混淆
- XTEA 函数的条件分支中，特定 key 索引会加上额外偏移值
- 偏移值作为 `PUSH_IMM` 直接嵌入字节码，自动提取

### 4.5 加密包装流程
```
明文字符串 → 每 4 字符小端序 pack 为 int32 → XTEA 加密 → int32 小端序 unpack → Base64 编码
```

### 4.6 互联网调研
- 搜索中英文社区（看雪、CSDN、知乎、GitHub），无人公开过从字节码自动静态提取 key 的方法
- 现有方案：插桩手动分析、断点抓取、补环境执行

---

## 五、collect 动态构造

### 5.1 collect 整体结构
```
collect = base64(xtea(chunk1_padded)) + base64(xtea(traj_padded)) + base64(xtea(chunk2_padded)) + base64(xtea(sd_str))
```
- chunk1/traj/chunk2 空格 pad 到 24 字节对齐，sd 不 pad

### 5.2 cd 数组与 37 个模块
- 37 个采集器模块（36 种类型），采集浏览器指纹、环境信息、行为数据
- 31 个有标准 `.get` 导出，6 个无 `.get` 但仍产生 cd token
- guid_token 是 RAW 模式（逗号分隔产生多个 token）

### 5.3 模块指纹识别：从失败到成功
- **失败方案（v1-v3）**：静态分析 scope slot → 类型识别准确，但位置映射全部错误
- **根因**：LOAD_DYN、共享引用赋值、运行时 scope 变化等动态行为无法纯静态推断
- **成功方案**：字符串集合 + 结构特征指纹
  - 每个模块包含稳定的 JS API 字符串（如 `"userAgent"`、`"getTimezoneOffset"`）
  - 无区分性字符串的通用模块用 `.get` 函数指令数 + entry 函数指令数区分
  - 3 个 tdc.js 版本、37 个模块全部正确识别

### 5.4 模块顺序还原
- 从编排函数字节码提取 require_seq（37 个模块的调用顺序）
- 结合指纹识别确定每个位置的模块类型，生成正确的 cd JSON

### 5.5 func_4310 构建逻辑
- 遍历模块数组，调用 `.get()` 获取 `[value, secondary, type_flag]`
- type_flag=2 触发轨迹分界（加密 chunk1，插入轨迹密文）
- type_flag=1 表示 RAW 输出（不加引号）
- 数字 falsy 时输出 0，字符串加引号

---

## 六、全流程打通

### 6.1 轨迹生成
- 贝塞尔曲线模拟鼠标移动轨迹
- 差分编码：第一个点绝对值，后续点相对偏移
- 固定结尾标记 `[1, 1, 12]`
- 坐标系：相对 `tCaptchaDyContent` 容器（360×360），使用 `event.offsetX/Y`

### 6.2 OCR 文字识别
- HSV 颜色空间分割黄色文字区域
- PaddleOCR v5 汉字识别
- 当前瓶颈：形近字误识别（如 边/芭/笆）

### 6.3 指纹随机化
- 6 款 Mac 设备画像（屏幕分辨率/WebGL 渲染器/CPU 核心数/Canvas 指纹）
- Chrome 130-145 版本 + macOS 版本随机组合
- 生成一致的 UA/sec-ch-ua/环境变量

### 6.4 PoW 求解
- MD5 碰撞：已知 prefix + 目标 MD5 前缀，暴力枚举 nonce

### 6.5 端到端编排
```
prehandle → 下载 tdc.js → 动态解析(key/offsets/eks/module_types)
→ 下载验证码图片 → OCR 识别 → 生成轨迹 → 构建 collect → PoW → verify
```

---

## 七、工具介绍：JS Reverse MCP

### 7.1 为什么需要这个工具
- 传统 JS 逆向依赖手动操作 DevTools：打断点、看调用栈、抓变量，效率低且难以自动化
- AI 编码助手（Claude Code、Cursor、Copilot）擅长分析代码，但无法直接操控浏览器调试器
- **JS Reverse MCP** 是连接 AI 与浏览器调试能力的桥梁，让 AI 能像人一样操作 DevTools

### 7.2 核心能力
- **脚本分析**：列出页面所有 JS 脚本、搜索代码、获取源码（支持压缩代码的字符偏移定位）
- **断点调试**：设置/移除断点、条件断点、XHR 断点，暂停后检查调用栈和作用域变量
- **函数追踪**：Hook 任意函数（包括 webpack 打包的模块内部函数），监控调用参数和返回值
- **单步调试**：step over / step into / step out，在断点处求值任意表达式
- **网络分析**：查看请求详情、获取请求发起的调用栈（定位加密参数的生成入口）
- **运行时检查**：深度检查对象结构、获取存储数据、监控 DOM 事件、页面截图

### 7.3 在本项目中的实际应用
- **请求分析**：`list_network_requests` + `get_network_request` 快速定位加密参数所在的请求
- **入口定位**：`break_on_xhr` 设置 XHR 断点 → 触发请求 → `get_paused_info` 获取调用栈，直接找到加密函数
- **Key 验证**：`set_breakpoint` 在 XTEA 函数打条件断点 → `evaluate_on_callframe` 提取运行时 key 和 scope 变量
- **iframe 切换**：`list_frames` + `select_frame` 切换到跨域 iframe 上下文，直接调用 `TDC.getData(true)`
- **Hook 拦截**：`hook_function` 拦截 `encodeURIComponent`，抓取 collect 加密后、URL 编码前的数据
- **源码搜索**：`search_in_sources` 在所有加载的脚本中搜索关键字（如 `"x-sign"`、`"encrypt"`）

### 7.4 安装与配置

**GitHub**: [https://github.com/anthropics/js-reverse-mcp](https://github.com/anthropics/js-reverse-mcp)

```bash
# 安装
git clone https://github.com/anthropics/js-reverse-mcp.git
cd js-reverse-mcp && npm install && npm run build

# Claude Code 配置
claude mcp add js-reverse node /路径/js-reverse-mcp/build/src/index.js

# 连接已运行的 Chrome（推荐，可保留登录状态和 cookie）
claude mcp add js-reverse node /路径/js-reverse-mcp/build/src/index.js -- --browser-url=http://127.0.0.1:9222
```

### 7.5 典型工作流
```
AI: list_network_requests → 找到目标请求
AI: get_request_initiator → 定位加密函数调用栈
AI: search_in_sources("encrypt") → 搜索加密相关代码
AI: set_breakpoint_on_text("encryptData(") → 在加密函数设断点
用户: 触发操作
AI: get_paused_info → 查看参数、调用栈、作用域变量
AI: evaluate_on_callframe("key") → 提取运行时密钥
AI: step_into / step_over → 单步跟踪加密逻辑
AI: resume → 继续执行
```

---

## 八、总结与思考

### 8.1 AI 的实际贡献
- 字节码精读与伪代码还原：24000+ 条指令的反汇编中快速定位关键逻辑
- 模式识别：从 14 个 key 子函数中识别真实/诱饵的判别规则
- 方案迭代速度：模块分析器 v1→v2→v3→指纹方案，快速试错
- 跨文件关联：追踪 scope chain、函数调用链、模块注册映射

### 8.2 AI 做不好的部分
- JS 位运算语义：需要人工提醒 `+` 不截断但 `<<` 截断的细微差异
- VM 动态行为推断：LOAD_DYN、共享引用等运行时语义超出静态分析能力
- 创造性突破：从"分析每个模块的值"到"用指纹识别模块类型"的思路转换

### 8.3 安全对抗趋势
- VM 保护的有效性与局限性
- 动态化（key/opcode/模块 ID 随机化）增加了逆向成本
- 但模块代码语义不变，给了指纹识别的机会
