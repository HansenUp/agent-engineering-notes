# agent-engineering-notes

hansenup 的 Agent 工程笔记，从企业技术负责人的视角，记录 AI Agent 落地时的选型、架构和注意事项。

以案头调研和学习分享为主。每篇都会写明信息截止日期、来源，以及哪些内容没有核实。欢迎在 Issues 里指正。

## 文章

| 日期 | 文章 | 简介 |
|---|---|---|
| 2026-10 | [从「存在之学」到「可执行本体」：Ontology 的概念、演进与企业 AI 落地](articles/2026-10-ontology-enterprise-ai.md) | 按「概念 → 演进 → 企业应用」梳理 Ontology 从哲学、知识工程、语义网、知识图谱到可执行本体的脉络；企业应用部分讨论业务语义执行层、Action 与权限治理、FDE 交付方法和是否引入 Ontology 的判断。 |
| 2026-10 | [四家官方 Agent SDK 怎么选：Google ADK、Claude Agent SDK、OpenAI Agents SDK、DeepSeek Harness](articles/2026-10-official-agent-sdk-comparison.md) | 用"谁控制循环、谁保存状态、谁能拦截、谁负责恢复"四个问题，加一个"绑定"问题，对比四家官方 Agent SDK；附场景决策指南和企业落地注意事项。信息截至 2026-10-29。 |
| 2026-09 | [Agent Framework（智能体框架）详细调研对比报告](articles/2026-09-agent-framework-research.md) | 对比 LangChain、LangGraph、AgentScope 2.x、Spring AI Alibaba、Claude Agent SDK、DeepSeek Harness 的架构层次、能力矩阵、许可与运维、PoC 评分方法和企业组合建议。核验于 2026-09-15/16，2026-10-28 复核版本。[交互版](https://hansenup.github.io/enterprise-agent-guide/research.html) |

## 说明

- 内容是个人学习分享，不代表任何组织的观点。
- 版本和功能变化很快，请以各项目官方文档为准。
