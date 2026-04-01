# Claude Code Prompt Engineering 学习指南

本目录是一组面向 GitHub 阅读场景整理的教学文档，目标不是复述源码，而是从 Claude Code 源码中提炼出可学习、可迁移、可复用的 prompt engineering 方法。

## 适合谁阅读

这组文档适合以下读者：

- 想系统理解 Claude Code prompt 架构的开发者
- 正在设计 Agent、CLI、Copilot 类产品的工程师
- 希望把 prompt 从“文案”提升为“系统设计”的团队

## 学习目标

完成本组文档后，建议至少能够回答以下问题：

- Claude Code 的 system prompt 为什么稳定。
- 工具、Skill、Agent 三层 prompt 为什么要分开设计。
- 为什么 context、memory、cache、compact 必须一起考虑。
- 为什么安全、输出体验、压缩策略也属于 prompt engineering。
- 如果重新实现一套类似框架，应该从哪里开始。

## 阅读顺序

1. `01-system-prompt-architecture.md`
   理解 system prompt 的装配方式、分层方式和优先级规则。
2. `02-tool-skill-agent-patterns.md`
   理解 Tool prompt、Skill prompt、Agent prompt 的职责边界。
3. `03-context-memory-cache.md`
   理解上下文来源、规则文件、memory、prompt cache 与 compact 的关系。
4. `04-safety-compression-and-output.md`
   理解安全策略、压缩机制和用户可见输出为什么要进入 prompt 层设计。
5. `05-build-your-own-framework.md`
   将前四篇的设计原则翻译成一个可以直接落地的框架模板。

## 建议搭配阅读的源码

- `src/constants/prompts.ts`
- `src/constants/systemPromptSections.ts`
- `src/utils/systemPrompt.ts`
- `src/context.ts`
- `src/utils/claudemd.ts`
- `src/skills/loadSkillsDir.ts`
- `src/tools/SkillTool/prompt.ts`
- `src/tools/AgentTool/prompt.ts`
- `src/tools/WebFetchTool/prompt.ts`
- `src/services/compact/prompt.ts`

## 推荐阅读方式

如果时间有限，优先阅读第 1、3、5 篇。

如果目标是做源码研究，建议按顺序完整阅读，再对照源码文件逐段查看。

如果目标是直接搭框架，可以先看第 5 篇，再回到第 1 和第 3 篇补系统设计细节。
