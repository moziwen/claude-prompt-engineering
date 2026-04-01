# 02. Tool, Skill, Agent 三层 Prompt 模式

## 学习目标

阅读本篇后，建议能够理解以下问题：

- Tool prompt 与 system prompt 的职责区别是什么。
- Skill prompt 为什么本质上是可复用 prompt 模板。
- Agent prompt 为什么必须区分 fresh agent 与 fork agent。
- Claude Code 为什么能把多层 prompt 组合得比较稳定。

## 1. 总体框架

Claude Code 的 prompt 体系不是只有 system prompt 一层。从源码可以清楚看到至少还有三类重要结构：

- Tool prompt
- Skill prompt
- Agent prompt

这三类 prompt 处理的是不同粒度的问题：

- Tool prompt 负责局部动作规范。
- Skill prompt 负责高频任务模板。
- Agent prompt 负责任务委派和上下文继承。

## 2. Tool prompt 的职责

Claude Code 中的工具说明不是面向人类开发者的 API 文档，而是面向模型的操作规程。

典型文件包括：

- `src/tools/FileReadTool/prompt.ts`
- `src/tools/BashTool/prompt.ts`
- `src/tools/WebSearchTool/prompt.ts`
- `src/tools/WebFetchTool/prompt.ts`

### 2.1 FileReadTool 的设计思路

`Read` 工具不仅声明“能读取文件”，还明确告诉模型：

- 路径必须使用绝对路径。
- 对大文件应优先局部读取。
- 目录读取不应使用 Read，而应切换到 Bash。
- 图片、PDF、截图都可以通过 Read 处理。

这种设计减少了模型在文件读取场景下的误用概率。它传递的不是能力列表，而是正确用法。

### 2.2 BashTool 的设计思路

`Bash` prompt 的重点不在 shell 语法，而在风险约束和操作边界。例如：

- 不要用 Bash 代替专用工具。
- Git 操作需要遵守安全规则。
- sandbox 和后台任务的用法有明确边界。

这使得 Bash 不再是“万能逃生口”，而是受系统规则约束的最后手段。

### 2.3 WebSearchTool 的设计思路

`WebSearch` prompt 体现了两个常见而重要的策略：

- 对输出格式做强约束，例如必须给 Sources。
- 对查询方式做强约束，例如涉及近期信息时必须带当前年份。

这说明 Claude Code 会把检索策略直接前置到工具 prompt 中，而不是等待模型在推理中自行发现这些规则。

### 2.4 WebFetchTool 的设计思路

`WebFetch` 更进一步，它实际上做了二级 prompt 设计：

- 第一层负责抓取网页内容。
- 第二层把网页内容与一个更小、更快模型的 prompt 组合起来，用于提炼结果。

在这个二级 prompt 中，还会加入：

- 引用长度限制
- 仅基于当前内容回答的要求
- 避免过度复述的要求
- 特定内容类型的输出限制

这是一种非常值得借鉴的模式：

**主任务使用一个 prompt，子任务使用更窄、更具体的 prompt。**

## 3. Tool prompt 的核心价值

如果抽象出 Claude Code 的设计原则，Tool prompt 的价值主要体现在三点：

1. 约束模型动作方式，而不仅是描述工具功能。
2. 把常见错误用法提前写进 prompt，减少误用。
3. 把局部最佳实践沉淀为工具级策略，而不是依赖主 prompt 统一承担。

## 4. Skill prompt 的职责

`src/skills/loadSkillsDir.ts` 展示了一套非常成熟的 prompt 模块化方案。

一个 skill 并不只是一个命令别名，它通常包含：

- 描述信息
- 使用时机
- 允许工具列表
- 参数提示
- model / effort 设定
- path 匹配规则
- hooks 或 shell 行为

### 4.1 为什么 skill 本质上是 prompt 模板

Claude Code 运行一个 skill 时，会：

1. 读取 skill markdown 正文。
2. 替换参数占位符。
3. 注入 session 变量。
4. 在允许范围内执行 prompt 内嵌 shell。
5. 把最终文本作为当前任务 prompt 注入对话。

因此，skill 的实质不是“命令”，而是“可参数化、可路由、可复用的 prompt 模块”。

### 4.2 SkillTool prompt 为什么关键

`src/tools/SkillTool/prompt.ts` 中有一条非常关键的规则：

- 如果当前任务匹配 skill，必须先调用 Skill tool，再继续生成其他回答。

这意味着 Claude Code 不只是提供 skill 能力，还把“如何发现并使用这些能力”本身做成了 prompt 规则。

换句话说，它把 prompt routing 也模块化了。

## 5. Agent prompt 的职责

`src/tools/AgentTool/prompt.ts` 是 Claude Code 最值得研究的 prompt 文件之一。它最有价值的地方在于：

- 它区分了 fresh agent 与 fork agent。
- 它明确规定了各自应采用的 prompt 写法。
- 它甚至直接教模型如何给另一个 agent 写 prompt。

### 5.1 fresh agent

fresh agent 没有继承上下文，因此 prompt 需要承担 briefing 功能，通常应该包含：

- 当前任务目标
- 背景和原因
- 已确认的信息
- 相关文件或线索
- 输出要求

### 5.2 fork agent

fork agent 继承父上下文，因此 prompt 应更接近 directive，重点是：

- 当前只处理哪一部分
- 哪些范围不属于本 agent
- 其他 agent 正在处理什么
- 返回结果的形式是什么

Claude Code 在这一点上非常明确，这正是它的多 Agent 协作相对稳定的重要原因。

## 6. 为什么这三层必须分开

Claude Code 将 Tool、Skill、Agent 三层 prompt 分开，是因为三者处理的问题不同：

### 6.1 Tool prompt 解决动作准确性

它帮助模型“怎么正确使用局部能力”。

### 6.2 Skill prompt 解决高频任务复用

它帮助模型“在遇到某类任务时，不必每次从零组织工作流”。

### 6.3 Agent prompt 解决任务分发质量

它帮助模型“在多 Agent 场景中，正确地说明任务、边界和上下文”。

如果把这三层都塞进 system prompt，系统会很快失去清晰性与可维护性。

## 7. 复用建议

如果希望在自己的系统中借鉴这一模式，建议按以下顺序实施：

1. 先给关键工具写 prompt。
2. 再把高频工作流做成 skill。
3. 最后给多 Agent 系统设计专门的 agent prompt 规范。

### 7.1 最值得优先写 prompt 的工具

- 文件读取
- 文件编辑
- shell 执行
- 网络搜索
- 网页抓取
- agent spawn

### 7.2 最适合做成 skill 的任务

- 提交代码
- 审查 PR
- 生成发布说明
- 故障排查
- 验证流程

### 7.3 多 Agent 系统中最重要的一条规则

一定要区分：

- 新 agent 需要完整 briefing
- fork agent 需要简洁 directive

## 8. 本篇结论

Claude Code 的强点不在于“它有 Tool、Skill、Agent”，而在于：

- 它为三者分别设计了不同粒度的 prompt。
- 它让这些 prompt 各自承担不同职责。
- 它通过路由和委派规则把三层连接起来。

## 源码定位

- `src/tools/FileReadTool/prompt.ts`
- `src/tools/BashTool/prompt.ts`
- `src/tools/WebSearchTool/prompt.ts`
- `src/tools/WebFetchTool/prompt.ts`
- `src/skills/loadSkillsDir.ts`
- `src/tools/SkillTool/prompt.ts`
- `src/tools/AgentTool/prompt.ts`
