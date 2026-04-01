# 03. 上下文, Memory, Cache 的协同设计

## 学习目标

阅读本篇后，建议能够理解以下问题：

- Claude Code 的上下文主要来自哪些层。
- `CLAUDE.md` 体系为什么本质上是项目级 prompt 系统。
- memory、transcript 和 compact 分别承担什么职责。
- prompt cache 为什么属于 prompt engineering 的核心问题。

## 1. Claude Code 的上下文不是单一来源

在 Claude Code 中，上下文至少来自三个主要部分：

- system prompt 本体
- userContext
- systemContext

这种拆分并非形式化处理，而是为了让不同类型的信息进入不同的 prompt 层。

## 2. userContext 与 systemContext 的区别

`src/context.ts` 中的两个函数非常关键：

- `getUserContext()`
- `getSystemContext()`

### 2.1 `getUserContext()` 承载什么

从源码可以看到，它主要承载：

- `CLAUDE.md`
- `CLAUDE.local.md`
- `.claude/rules/*.md`
- 当前日期

这些内容本质上更接近长期规则与用户偏好，因此适合作为 user context 注入。

### 2.2 `getSystemContext()` 承载什么

它主要承载：

- git status 快照
- 当前分支信息
- recent commits
- 特殊缓存干预信息

这些内容属于系统观测到的事实，更适合作为 environment-like context 注入。

### 2.3 为什么要这样拆分

因为这两类信息在以下方面不同：

- 来源不同
- 生命周期不同
- 变化频率不同
- 对缓存稳定性的影响不同

如果把它们统一拼进一个大文本中，后续做裁剪、缓存和调试都会变得困难。

## 3. `CLAUDE.md` 体系为什么值得重点学习

`src/utils/claudemd.ts` 是 Claude Code 中最值得借鉴的 prompt 外置机制之一。

它让项目规则不必硬编码在 prompt 构造器里，而是通过文件系统组织进来。

### 3.1 加载顺序体现了优先级设计

源码中定义了明确的加载顺序，大致包括：

1. managed memory
2. user memory
3. project memory
4. local memory

这意味着不同来源的规则具备不同优先级，系统可以更清晰地决定“谁覆盖谁”。

### 3.2 它不仅支持文件读取，还支持规则系统

这个机制并不只是“读取一个 markdown 文件”，它还支持：

- 目录向上遍历查找
- `.claude/rules/*.md`
- `@include` 引用
- frontmatter 路径匹配
- 文本扩展名白名单
- 循环引用避免
- 超长内容过滤与截断

换句话说，Claude Code 实际上为项目规则实现了一个小型规则加载系统。

### 3.3 为什么这种做法好

因为它把 prompt 中最容易长期变化的部分，从代码层迁移到了文件层。这样做的好处是：

- 团队可维护性更高
- 项目本身可以携带 AI 使用规范
- prompt 逻辑和项目规则解耦

## 4. memory 与 transcript 不是同一种东西

这是 Claude Code 很值得学习的一点。

### 4.1 transcript 负责什么

transcript 的职责是记录历史会话与执行过程。它更接近运行日志和会话持久化。

### 4.2 memory 负责什么

memory 的职责是影响未来行为。它更接近长期偏好、稳定约束和可复用背景。

### 4.3 为什么必须分开

如果把所有历史内容都当成 memory，会出现两个问题：

- 上下文噪音迅速增加
- 真正重要的长期规则反而不稳定

Claude Code 的思路是：

- 把需要长期影响模型行为的内容文件化、结构化。
- 把历史执行过程交给 transcript 和 compact 体系处理。

## 5. prompt cache 为什么是 prompt engineering 问题

Claude Code 在 prompt cache 上投入了明显的工程心智。它关注的不是抽象的“缓存”概念，而是：

- 哪些 prompt 片段会变
- 哪些变化会导致 cache break
- 哪些变化必须移到动态边界后面
- 哪些状态值得被监测

这说明它把 prompt 看成一种长期运行的系统输入，而不是一次性文本。

### 5.1 这在产品中为什么重要

只要系统具备以下特征，prompt cache 就会变得重要：

- 多轮对话
- 多工具
- 多 agent
- 长上下文
- 多种运行模式

Claude Code 的一个精华就在于，它从一开始就承认 prompt 是成本中心和稳定性中心。

## 6. `promptCacheBreakDetection` 体现了监测思维

`src/services/api/promptCacheBreakDetection.ts` 追踪了大量会影响 prompt cache 的状态，例如：

- system prompt hash
- tools hash
- per-tool schema hash
- cache control hash
- model
- beta headers
- effort
- extra body params

这说明 Claude Code 并不是凭经验判断 cache 为什么失效，而是在显式跟踪 prompt 形态变化。

对教学而言，这一点非常重要，因为它告诉我们：

**当 prompt 成为系统核心输入时，它就应该像配置和接口一样被观测。**

## 7. compact 是上下文续航机制，不只是摘要

`src/services/compact/prompt.ts` 中的 compact prompt 设计非常成熟。它并不是简单要求模型“总结一下历史”，而是要求它输出结构化、可继续工作的上下文摘要。

通常包括：

- 用户意图
- 技术背景
- 文件和代码段
- 错误与修复
- 当前工作
- 下一步

### 7.1 为什么要这样设计

因为会话压缩最容易丢失的是工作状态，而不是表面信息。一个看似不错的摘要，如果不能告诉系统“接下来该做什么”，在工程上就没有真正完成 compact 的目标。

### 7.2 `<analysis>` 与 `<summary>` 的意义

Claude Code 要求模型先写 `<analysis>`，再写 `<summary>`。这背后的设计意图是：

- 允许模型先整理信息
- 最终只保留结构化总结
- 提高压缩结果质量，同时避免把草稿噪音带入后续上下文

## 8. 可以直接借鉴的结构

如果要在自己的系统中借鉴 Claude Code 的上下文设计，建议至少区分以下五层：

1. 稳定的 system prompt 规则
2. 会话级动态信息
3. 项目规则文件
4. 长期 memory
5. 可压缩的历史对话与摘要

这一结构能够显著提升系统的可维护性与可预测性。

## 9. 本篇结论

Claude Code 在上下文管理上的精华不在于“给模型更多信息”，而在于：

- 将不同来源的信息分层处理
- 用文件系统承载长期规则
- 将 memory 与 transcript 区分开
- 把 cache 和 compact 纳入 prompt engineering 统一设计

## 源码定位

- `src/context.ts`
- `src/utils/claudemd.ts`
- `src/services/api/promptCacheBreakDetection.ts`
- `src/services/compact/prompt.ts`
