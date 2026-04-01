# 05. 将 Claude Code 的 Prompt 工程翻译成可落地框架

## 学习目标

阅读本篇后，建议能够完成以下工作：

- 为自己的 Agent 系统设计一套基础 prompt 分层。
- 知道哪些能力应该优先实现，哪些能力可以后置。
- 能将 Claude Code 的设计原则转化为自己的目录结构、builder 代码和模板。

## 1. 先建立一个现实目标

如果要复刻 Claude Code 的完整能力，成本会很高；但如果目标是提炼其 prompt engineering 精华，完全可以先实现一个“足够稳”的最小框架。

建议的最小框架包含六层：

1. 固定 system prompt
2. 模式层 prompt
3. 项目规则文件层
4. 环境上下文层
5. 工具 prompt 层
6. compact prompt 层

只要这六层组织得当，系统通常就已经优于大多数“单 prompt + 若干工具”的实现。

## 2. 推荐的目录组织

下面是一种接近 Claude Code 思路的组织方式：

```text
prompts/
  system/
    base.md
    modes/
      interactive.md
      autonomous.md
      coordinator.md
    sections/
      safety.md
      output.md
      tool-policy.md
  tools/
    read.md
    edit.md
    shell.md
    search.md
  skills/
    commit.md
    review.md
    debug.md
  compact/
    summary.md
runtime/
  prompt-builder.ts
  context-builder.ts
  memory-loader.ts
  tool-registry.ts
```

这不是唯一正确方案，但它体现了几个关键原则：

- 主规则与局部规则分离
- prompt 内容与运行时逻辑分离
- system、tool、skill、compact 分开维护

## 3. 一个可直接复用的 Prompt Builder 结构

下面是一段简化后的示意代码：

```ts
type PromptSection = {
  name: string
  compute: () => Promise<string | null> | string | null
  cacheBreak?: boolean
}

async function buildPrompt(ctx: RuntimeContext) {
  const staticSections = [
    introSection(ctx),
    safetySection(ctx),
    toolPolicySection(ctx),
    outputSection(ctx),
  ]

  const dynamicSections = await resolveSections([
    modeSection(ctx),
    memorySection(ctx),
    languageSection(ctx),
    envSection(ctx),
    sessionGuidanceSection(ctx),
  ])

  return [
    ...staticSections.filter(Boolean),
    '__PROMPT_DYNAMIC_BOUNDARY__',
    ...dynamicSections.filter(Boolean),
  ]
}
```

这段代码的重点不在语法，而在思路：

- 用 section 而不是大字符串
- 把稳定内容和变化内容隔离
- 允许后续引入缓存和诊断能力

## 4. system prompt 至少应包含哪些层

### 4.1 身份层

明确系统的角色、主要职责和服务边界。

### 4.2 任务原则层

明确系统如何做任务，例如：

- 先理解再行动
- 不超范围修改
- 失败先诊断
- 结果如实汇报

### 4.3 风险动作层

明确：

- 哪些动作默认可执行
- 哪些动作需要确认
- 被拒绝后如何处理
- 哪些破坏性动作不能当作捷径

### 4.4 工具总规则层

明确：

- 优先用专用工具
- 什么时候可以并行
- shell 应作为补充而不是默认方案

### 4.5 输出规范层

明确：

- 什么时候给进展更新
- 简单问题如何回答
- 复杂问题如何结构化
- 文件引用格式是什么

### 4.6 动态上下文层

注入：

- 模式信息
- memory
- 当前环境
- 当前任务
- 特定会话指导

## 5. 项目规则文件建议单独设计

Claude Code 给出的一个重要启发是：长期规则不要直接硬编码在 prompt builder 里。

可以设计类似如下的规则体系：

- `AGENT.md`：项目公开规则
- `AGENT.local.md`：本地私有规则
- `.agent/rules/*.md`：按主题拆开的规则

### 5.1 适合放进规则文件的内容

- 代码风格要求
- 目录职责
- 测试要求
- 协作规范
- 提交规范
- 项目的特殊禁忌

### 5.2 不适合放进规则文件的内容

- 临时任务说明
- 高频变化的运行时状态
- 很短生命周期的局部上下文

这些更适合通过当前任务 prompt 或环境上下文注入。

## 6. 工具 prompt 模板建议

Claude Code 的经验表明，关键工具最好都带 prompt。可以直接使用下面的模板：

```md
这个工具用于做什么。

何时使用：
- ...
- ...

何时不要使用：
- ...
- ...

正确使用方式：
- ...
- ...

常见错误：
- ...
- ...
```

最值得优先写 prompt 的工具通常是：

- 文件读取
- 文件修改
- shell 执行
- 网络搜索
- 网页抓取
- agent spawn

## 7. Skill 系统的最小设计

如果系统里有明显的高频任务，建议尽早做 Skill 层。

一个最小 skill 可以包含：

```yaml
name: commit
when_to_use: 当用户要求提交代码时
allowed-tools:
  - Bash
  - Read
argument-hint: "可选的附加说明"
model: inherit
```

技能正文使用 markdown 保存，运行时再做参数展开和 prompt 注入。

Skill 的目标是：

- 复用高频工作流
- 降低模型从零组织流程的成本
- 把领域经验沉淀为可调用模块

## 8. 多 Agent 场景的最小设计

如果要支持多 Agent，务必将 prompt 区分为两类：

### 8.1 新 agent 的 briefing 模板

```text
任务目标是什么。
为什么要做这件事。
已经确认过什么。
哪些文件最相关。
需要你输出什么。
不要做什么。
```

### 8.2 fork agent 的 directive 模板

```text
继续当前上下文，专注处理 X。
范围仅包括 A 和 B，不要处理 C。
另一条 agent 正在处理 D。
完成后只返回结论和关键证据。
```

这两种 prompt 的差别非常重要，不建议混用。

## 9. compact prompt 的最小设计

一个可用的 compact prompt 至少应要求模型保留以下内容：

1. 用户当前意图
2. 技术背景
3. 相关文件与代码段
4. 已完成的修改
5. 错误与修复
6. 当前进度
7. 下一步建议

compact prompt 应明确说明：

- 目标是继续工作，不是生成美观摘要
- 后续对话会依赖这个总结继续执行

## 10. 一份可直接起步的总模板

下面是一份简化版总模板：

```text
# Identity
你是一个面向软件工程任务的 AI 代理。

# Core Rules
先理解再行动。
不要超范围修改。
失败先诊断，不要机械重试。
如未验证，不要声称已经验证。

# Risky Actions
本地可逆动作可直接做。
破坏性、共享状态、不可逆动作先确认。
用户拒绝后不要原样重试。

# Tool Policy
优先用专用工具。
无依赖动作可并行。
只有在没有更合适工具时才用 shell。

# Output
先说明你要做什么。
在关键节点给短更新。
最终只汇报用户需要知道的结果。

__PROMPT_DYNAMIC_BOUNDARY__

# Mode
当前模式说明。

# Project Rules
这里注入项目规则文件。

# Memory
这里注入长期记忆。

# Environment
cwd, git, platform, date, tools。

# Task
当前用户请求。
```

这份模板足够作为 Claude Code 风格系统的起点。

## 11. 推荐实施顺序

为了减少复杂度，建议按如下顺序实现：

1. 主 prompt 分层
2. 规则文件注入
3. 关键工具 prompt
4. Skill 系统
5. 多 Agent prompt
6. compact prompt

这个顺序的优点是每一步都能独立产生收益。

## 12. 本篇结论

将 Claude Code 的 prompt engineering 转化为自己的框架时，最值得复用的是结构，而不是措辞。优先复用以下原则：

- 分层
- 模块化
- 运行时装配
- 外置规则文件
- 工具级约束
- Agent prompt 分型
- 长会话下的 compact 设计

## 对应回读建议

读完本篇后，建议回到以下文档复查：

- `01-system-prompt-architecture.md`
- `03-context-memory-cache.md`
