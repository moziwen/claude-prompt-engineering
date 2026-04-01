# 01. System Prompt 架构

## 学习目标

阅读本篇后，建议能够理解以下问题：

- Claude Code 的 system prompt 如何组装。
- 为什么它要区分静态前缀和动态尾部。
- section 机制如何帮助 prompt 维护和缓存。
- prompt 优先级为什么必须显式设计。

## 1. 总体观察

Claude Code 的 system prompt 不是一个固定模板，而是一条运行时装配链。主入口位于 `src/constants/prompts.ts` 的 `getSystemPrompt()`，它会根据当前模式、可用工具、会话状态、memory、语言偏好和 output style 组装出最终的 prompt 数组。

这意味着 Claude Code 并没有把 prompt 视为单一文本文件，而是视为一组具有职责分工的片段。

## 2. System prompt 的基本装配流程

从源码结构看，主流程可以概括为：

1. 收集当前运行模式信息。
2. 读取 output style、tool 状态、skill 命令列表。
3. 构造基础 system prompt 段。
4. 计算动态 section。
5. 合并静态段和动态段。
6. 在更高一层根据 agent、coordinator、自定义 prompt 规则进一步改写。

其中最重要的两个文件是：

- `src/constants/prompts.ts`
- `src/utils/systemPrompt.ts`

前者负责默认 prompt 的主体装配，后者负责更高层级的覆盖与继承规则。

## 3. 为什么它要区分静态段和动态段

`src/constants/prompts.ts` 中定义了一个边界常量：

- `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`

这个边界的主要目的不是排版，而是缓存稳定性。

### 3.1 静态段通常承载什么

静态段适合承载以下内容：

- 身份与角色定位
- 通用任务原则
- 风险动作原则
- 工具使用总原则
- 基础输出规范

这些内容变化频率低，能够在不同会话之间保持一致，因此适合放在 prompt 前缀部分。

### 3.2 动态段通常承载什么

动态段适合承载以下内容：

- 当前工具状态引出的 session guidance
- 当前 language setting
- output style
- MCP 指令
- scratchpad 信息
- memory 注入
- token budget 等模式化控制项

这些内容变化频率高，如果提前放入前缀，会显著增加 prompt cache 失效率。

### 3.3 这带来的核心收益

这种拆分让 Claude Code 具备三个明显优势：

- prompt 前缀更稳定
- 缓存命中率更高
- 动态变化的影响范围更可控

在产品场景中，这种收益往往比“多写几句更强的 prompt”更重要。

## 4. Section 机制为什么重要

`src/constants/systemPromptSections.ts` 提供了一个非常关键的抽象：

- `systemPromptSection(name, compute)`
- `DANGEROUS_uncachedSystemPromptSection(name, compute, reason)`
- `resolveSystemPromptSections()`

这套机制把“动态 prompt”从字符串拼接提升成了可注册、可缓存、可追踪的 section 系统。

### 4.1 这种设计解决了什么问题

它解决的是 prompt 在工程维护中的三个常见难题：

1. 难维护：不知道一段 prompt 是从哪里来的。
2. 难调试：不知道为什么这轮 prompt 和上一轮不一样。
3. 难扩展：每次加一条规则都要改主模板。

通过 section 机制，每个动态片段都有自己的名字、计算逻辑和缓存属性，因此更容易演化。

### 4.2 `DANGEROUS_uncachedSystemPromptSection` 的意义

这个 API 命名非常直接，它实际上在提醒开发者：

- 这一段内容会频繁变化。
- 这一段可能导致 cache break。
- 只有确实必要时才应该使用。

这种设计值得借鉴，因为它让“性能代价”在 API 层就变得可见。

## 5. Prompt 优先级是如何定义的

`src/utils/systemPrompt.ts` 的 `buildEffectiveSystemPrompt()` 明确规定了 prompt 的优先级顺序：

1. override system prompt
2. coordinator system prompt
3. agent system prompt
4. custom system prompt
5. default system prompt
6. append system prompt

这种显式优先级非常关键，因为复杂系统中的 prompt 冲突通常来自“谁覆盖谁”不明确。

### 5.1 这一设计最值得借鉴的点

Claude Code 把 prompt 操作区分为两类：

- 替换类操作
- 追加类操作

这避免了常见的几类问题：

- agent prompt 与默认 prompt 语义冲突
- 自定义 prompt 意外覆盖安全规则
- 后续追加内容破坏前文语义

如果一个系统存在多种 prompt 来源，优先级链必须从一开始就明确下来。

## 6. 运行模式会直接改变 prompt 形态

Claude Code 并不尝试用单一 prompt 兼容所有模式。根据源码，可以观察到至少有这些变体：

- 默认交互模式
- simple 模式
- proactive 模式
- coordinator 模式
- 自定义 agent 模式

其中 proactive 模式尤其具有代表性。它会额外注入一整段自治工作规则，包括：

- 如何处理 tick
- 无事可做时要 sleep
- 第一个 tick 不要擅自开工
- 用户活跃时优先响应用户

这一设计说明 Claude Code 的 prompt 架构遵循一个重要原则：

**运行模式不同，行为规则应显式改变。**

## 7. 输出规范属于 system prompt 的组成部分

在 `src/constants/prompts.ts` 中，除了任务和工具相关段落，还可以看到：

- `Tone and style`
- `Output efficiency`

这说明 Claude Code 把用户可见输出视为核心产品行为，而不是事后修饰。

这两类规则主要在约束：

- 何时给进展更新
- 如何保持简洁
- 如何引用文件和代码位置
- 如何避免工具调用前后的歧义表达

这类约束对终端类产品尤其重要，因为用户通常看不到完整的内部执行过程，只能通过模型输出理解系统状态。

## 8. `getUsingYourToolsSection()` 体现了调度思维

Claude Code 在 system prompt 中不仅定义了身份与规则，还直接引导模型做工具调度，例如：

- 优先使用专用工具，不要滥用 Bash
- 可以并行调用无依赖工具
- 搜索文件和搜索内容应使用不同工具
- 有任务工具时应及时维护任务状态

这意味着它把“工具路由”也前置到了 system prompt 中。对多工具 Agent 来说，这种调度提示往往能显著降低错误工具选择率。

## 9. 可以直接借鉴的系统 prompt 结构

如果要复用 Claude Code 的思路，可以先把主 prompt 拆成以下 6 层：

1. 身份层
2. 全局任务原则层
3. 风险动作层
4. 工具使用总原则层
5. 输出规范层
6. 动态上下文层

这样的分层既方便维护，也更容易接入后续的 Skill、Agent、memory 和 compact 机制。

## 10. 本篇结论

Claude Code 在 system prompt 上最值得学习的不是具体措辞，而是以下四点：

- prompt 必须结构化，而不是写成单一文本
- 静态内容与动态内容必须分开
- 动态 section 应具备独立缓存与可追踪能力
- prompt 的继承和优先级必须显式设计

## 源码定位

- `src/constants/prompts.ts`
- `src/constants/systemPromptSections.ts`
- `src/utils/systemPrompt.ts`
