# Agent Framework（智能体框架）详细调研对比报告

> 作者：hansenup（GitHub [@HansenUp](https://github.com/HansenUp)）
>
> 原文（交互版，可检索、可离线阅读）：<https://hansenup.github.io/enterprise-agent-guide/research.html>。本文是该页面的 Markdown 版本，内容以原文为准。

原有对象核验：2026-09-15；Claude Agent SDK 与 DeepSeek Harness 增补核验：2026-09-16。重点对象：LangChain、LangGraph、AgentScope Python 2.x、Spring AI Alibaba；新增 Claude Agent SDK、DeepSeek Harness；补充 AgentScope Java 与其他候选。面向企业 AI 能力平台架构师及高级开发人员。

先看选型结论和版本边界，再对照能力矩阵；准备落地时重点阅读 PoC 与企业组合。此报告未进行框架性能实测，未将开发版文档视为稳定版保证。

本文依据官方文档、仓库与发布记录进行案头调研。没有执行各候选同条件性能测试；“优先评估”“适合”等属于基于架构边界的建议，不代表已验证的性能排名。动态文档、开发分支和稳定发行版分别对待，文档支持不等于目标版本已经具备全部能力。

## 01 选型结论

**Python 侧应分别评估 LangChain + LangGraph 与 AgentScope 2.x；Java 侧应评估 Spring AI Alibaba，并把 AgentScope Java 纳入并列候选。希望复用 Claude 执行能力时评估 Claude Agent SDK；希望定制插件化运行环境时评估 DeepSeek Harness，但应明确其开发者预览边界。无需把各候选全部叠入同一条执行链。**

| 企业需要 | 优先评估 | 决策理由与前提 |
|---|---|---|
| Python，标准工具型 Agent，快速构建 | LangChain 高层 Agent API | 先使用成熟抽象，只有流程控制不足时再下沉图层 |
| Python，复杂分支、审批、状态可见与可恢复 | LangGraph，模型与工具可复用 LangChain | 需要明确的执行图和状态设计；必须配置持久存储与业务幂等 |
| Python，希望连同服务化、工作区、权限交互一起验证 | AgentScope 2.x | 集成面较宽，但要按稳定版本验收；不要继续沿旧 Runtime 架构新建 |
| Java / Spring 业务系统，显式流程与 AI 混合 | Spring AI Alibaba Agent Framework + Graph | 降低跨语言集成成本；验证 Spring Boot / Spring AI / Alibaba BOM 兼容 |
| Java，强调工具循环、工作区和分布式 Agent | AgentScope Java | 不是 Python 版能力的自动映射，应独立验证 Java SDK 和发行版 |
| Python / TypeScript，复用现成工具执行与上下文管理 | Claude Agent SDK | 适合嵌入 Claude 执行循环；企业仍负责部署、授权与业务状态 |
| 定制工具、模型、运行模式和插件组合 | DeepSeek Harness | 适合受控试点及 Harness 研发；固定预览版本、插件和恢复语义 |
| 跨部门、跨系统长事务 | 任一主 Agent 框架 + 企业持久工作流 | Agent 编排与业务事务分层；明确只有一个组件负责外部动作重试 |

依据：LangChain 与 LangGraph 的关系见[LangChain 概览](https://docs.langchain.com/oss/python/langchain/overview)、[LangGraph 概览](https://docs.langchain.com/oss/python/langgraph/overview)；另外两个生态见[AgentScope 仓库](https://github.com/agentscope-ai/agentscope)、[Spring AI Alibaba 架构](https://github.com/alibaba/spring-ai-alibaba)。上表的优先级是本报告判断。

## 02 必须先校准的版本与产品边界

| 对象 | 核验结果 | 对建设方案的影响 |
|---|---|---|
| LangChain | 当前推荐高层入口包括 `create_agent`，不能只按早期 Chains / AgentExecutor 印象理解 | 新项目从当前 API 与 Middleware 开始，旧项目单独盘点兼容包 |
| LangGraph | 状态编排与运行库；可以独立于 LangChain 使用 | 不能把它与 LangChain 当两套完全对立的全栈框架 |
| AgentScope Python | 官方发布页显示 v2.0.8；部分 latest 文档跳到 2.0.9dev | 稳定版优先验证；开发文档中的细项不得直接写进验收承诺 |
| AgentScope Runtime | 原仓库公告称能力已整合入 AgentScope 2.0，建议迁移，计划归档 | 新建方案不再默认另装旧 Runtime；存量先做迁移清单 |
| Spring AI Alibaba | 发布页将 v1.1.2.2 标为 Latest，v2.0.0-M1.1 标为 Pre-release | 稳定线与里程碑线分开；不能因为版本号大就作为生产默认 |
| Spring AI Alibaba 2.0 里程碑 | 发布说明涉及 Spring AI 2.0.0-M1、Spring Boot 4.0.0 升级 | 同时涉及底座升级，迁移不是单换一个依赖 |
| AgentScope Java | 独立仓库与模块；当前 README 展示 2.0.1 与 JDK 17+ | 不把 Python 示例、Java 示例和 Spring 集成版本混用 |
| Claude Agent SDK | Python / TypeScript SDK，复用 Claude Code 的运行能力；不是普通模型 Client SDK 或托管 Agent 产品 | 锁定 SDK、运行二进制与模型；不混用旧包名及两种语言参数 |
| DeepSeek Harness | 官方独立 dsh 项目，Developer Preview；仓库提示可能破坏兼容 | 固定版本或提交与插件组合；不以主分支文档作为稳定承诺 |

新增来源：[Claude Agent SDK 概览](https://code.claude.com/docs/en/agent-sdk/overview)、[DeepSeek Harness 仓库](https://github.com/deepseek-ai/deepseek-harness)。

核验来源：[AgentScope 发布记录](https://github.com/agentscope-ai/agentscope/releases)、[Runtime 迁移公告](https://github.com/agentscope-ai/agentscope-runtime)、[Spring AI Alibaba 发布记录](https://github.com/alibaba/spring-ai-alibaba/releases)、[AgentScope Java](https://github.com/agentscope-ai/agentscope-java)。版本是本次读取结果，不是未来部署锁定建议。

> **2026-10-28 发布前复核（GitHub Releases）：** AgentScope Python 已发布至 v2.0.11（2026-10-27）；AgentScope Java 最新稳定版为 v2.0.4，另有 v2.1.0-BETA1 预发布；Spring AI Alibaba 仍为 v1.1.2.2 Latest、v2.0.0-M1.1 Pre-release；DeepSeek Harness 仍只有预发布版本（dsh-v0.2.x alpha）。上表保留 2026-09 调研时的读取结果，选型时以对应仓库的最新发布记录为准。

Spring AI Alibaba 的概览示例与发布页版本并不完全相同。应以选定版本的 BOM、示例、源码和依赖树一起验证，避免复制不同年代的教程。其稳定发布说明还提到 AgentScope Java 适配；该事实不代表旧适配器自动兼容 AgentScope Java 2.x。

## 03 各候选分别位于什么架构层

### LangChain：Application Framework（应用开发框架）

将模型、消息、工具与常见 Agent 行为组合成较高层接口。适合把大部分标准模式交给框架，减少重复开发。当前 Agent 基于 LangGraph 构建，因此“LangChain 不支持状态、LangGraph 才能做 Agent”并不准确。[官方概览](https://docs.langchain.com/oss/python/langchain/overview)

### LangGraph：Orchestration Runtime（编排运行时）

核心关注 State（状态）、Node（节点）、Edge（边）、条件控制和执行持久化。它为应用开发者提供更直接的控制，但要求开发者明确建模状态、并发合并和失败路径。Graph 不等于只能无环 DAG，Agent 循环是重要使用方式。[官方概览](https://docs.langchain.com/oss/python/langgraph/overview)

### AgentScope：Agent SDK + Agent Service（智能体开发与服务化）

2.x 不应只当作早期多 Agent 消息框架。当前生态将 Agent 基础组件与服务层组合，提供工作区、事件、权限交互和服务化入口。集成范围更宽意味着可复用更多，也意味着需要评审更多默认行为与配套依赖。[官方架构入口](https://github.com/agentscope-ai/agentscope)

### Spring AI Alibaba：Agent Framework + Graph + 配套管理模块

Spring AI 提供模型和工具等基础抽象；Spring AI Alibaba 增加 Agent Framework 与 Graph，并提供 Admin / Studio 等配套模块。不能把 Spring AI、Alibaba Graph 和管理平台视为同一个依赖包，也不能认为它只是一层通义模型适配。[Spring AI 参考](https://docs.spring.io/spring-ai/reference/)、[Alibaba 架构说明](https://github.com/alibaba/spring-ai-alibaba)

### Claude Agent SDK：可嵌入的 Agent 运行能力

通过 Python / TypeScript 消费执行消息，复用工具循环、上下文、会话与扩展机制。它减少自建执行器的工作，但不替企业实现任务控制面或业务事务。[官方概览](https://code.claude.com/docs/en/agent-sdk/overview)

### DeepSeek Harness：可组合的 Harness（运行支撑层）

Cordis 管理插件组合和生命周期，能力由模型、工具、存储、循环等插件提供。当前架构包含 Profile、Bundle 和不同事件域；不是纯 Python 编排库，也不局限于 Web UI。[架构文档](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md)

## 04 核心能力矩阵

标记：**框架**＝核心 API 或其运行库提供机制；**配套**＝另一个模块、服务或适配器；**企业**＝仍需企业业务系统实现；**验证**＝按具体版本确认语义，未在本报告实测。各单元是能力边界，不是性能评分。

| 能力 | LangChain | LangGraph | AgentScope Python 2.x | Spring AI Alibaba | Claude Agent SDK | DeepSeek Harness |
|---|---|---|---|---| --- | --- |
| Function Calling（函数调用） | 框架：模型与工具抽象 | 通过节点和工具集成 | 框架：Toolkit / 模型适配 | 框架：Spring AI 工具 + Agent | 内置工具及自定义工具 / MCP | 工具注册与模型适配插件 |
| ReAct（推理与行动） | 高层 Agent 循环 | 自定义循环或嵌入 Agent | Agent 推理行动循环 | `ReactAgent`；类名不是前端 React | 复用 SDK 内部执行循环 | 默认循环及可组合运行模式 |
| 显式流程 | 可下沉 LangGraph | 核心优势：状态与控制图 | Pipeline / Team 等机制，验证版本 | Graph + 顺序/并行/路由/循环模式 | 外层应用组织；不等于显式状态图 | 插件、Profile 与工作流能力；验证预览版 |
| Checkpoint（检查点） | 依赖底层图与存储配置 | 框架机制 + 持久存储 | 状态/会话持久化；验证中断与崩溃恢复范围 | Graph 持久机制；验证存储和恢复边界 | 会话恢复 / 分叉；不等同业务事务检查点 | 会话事件与恢复机制；验证持久化和版本 |
| HITL（人在回路） | Middleware / 底层中断 | `interrupt` 与恢复 | Permission / 用户确认 | Hooks / Interceptors / 人机协作 | 权限规则、Hook 与人工输入回调 | 审批与策略插件；验证实际配置 |
| Multi-Agent（多智能体） | 模式组合与图集成 | 子图与自定义协调 | Agent Team、任务委派 | 预置 Agent 模式与 Graph | Subagents 任务委派 | 子 Agent / 相关插件与运行模式 |
| MCP（模型上下文协议） | 配套适配器 | 通过工具层接入 | Toolkit / 工作区相关支持 | Spring AI MCP 与相关集成 | SDK 连接外部 MCP 工具 | 按目标版本工具插件和配置验证 |
| A2A（智能体间协议） | 单独核验适配方案 | 不因有图而自动兼容 | v2.0.8 发布说明包含 A2AAgent | 官方文档描述 A2A / Nacos 集成 | 本报告未单独核验 A2A 适配 | 本报告未单独核验 A2A 适配 |
| Context Engineering（上下文工程） | Middleware 等机制 | 状态及节点自行组合 | 压缩、外置结果与中间件 | Hooks / Interceptors 相关策略 | 运行循环中的上下文管理 / 扩展 | 上下文装配、会话记录与插件扩展 |
| RAG（检索增强生成） | 连接与组件生态 | 用节点组织检索流程 | 服务层提供检索相关能力 | Spring AI 基础 + Agent/Graph 编排 | 通过工具接入企业知识检索 | 通过工具 / 插件接入知识检索 |
| Sandbox（沙箱） | 外接执行环境 | 外接执行环境 | 工作区/沙箱适配，按后端验证 | 仓库含沙箱模块，按版本与后端验证 | 部署环境与工具权限分别配置 | 沙箱插件 / 后端；按配置验证 |
| 企业授权与审批 | 企业实现 | 企业实现 | 内置权限机制仍需企业身份与规则 | Hook / Spring 安全体系仍需业务授权 | 企业实现；回调不是全部调用必经点 | 企业实现；插件化不等于强隔离 |
| 外部写入恰好一次 | 企业实现 | 企业实现 | 企业实现 | 企业实现 | 企业实现 | 企业实现 |
| 观测与评测平台 | LangSmith 或自选方案 | LangSmith 或自选方案 | 内置/配套路径，核验留存与导出 | Admin / 现有监控与评测集成 | 消息、Hook、用量与观测集成；评测另建 | Trajectory / 事件与观测插件；评测另建 |

矩阵依据：[LangChain Middleware](https://docs.langchain.com/oss/python/langchain/middleware/overview)、[MCP 适配](https://docs.langchain.com/oss/python/langchain/mcp)、[LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)、[AgentScope 服务](https://docs.agentscope.io/versions/2.0.9dev/en/deploy/agent-service)、[Spring AI Alibaba Hooks](https://java2ai.com/docs/frameworks/agent-framework/tutorials/hooks/)、[Graph 指南](https://java2ai.com/docs/frameworks/graph-core/quick-start/)。AgentScope 指定 dev 文档用于方向性核验，细项仍须在稳定版验证。

新增列依据：[Claude 权限](https://code.claude.com/docs/en/agent-sdk/permissions)、[会话](https://code.claude.com/docs/en/agent-sdk/sessions)、[DeepSeek 官方介绍](https://www.deepseek.com/harness/)、[架构文档](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md)。未核验不代表不支持。

## 05 LangChain 详细评估

### 开发方式与适合的团队

通常从统一模型接口、工具定义和 `create_agent` 开始，用 Middleware（中间件）处理模型选择、上下文、工具调用和其他横切行为。需要更精细控制时进入 LangGraph 层，而不是在复杂提示词里模拟所有流程。[Middleware 官方说明](https://docs.langchain.com/oss/python/langchain/middleware/overview)

**本报告判断：**它适合业务试验较多、需要替换模型和连接器的团队。开发者可以快速组合能力，但需管理模型提供方差异、集成包版本以及默认 Agent 行为。工具 schema 合法不等于业务输入正确，异常包装也不能统一所有下游语义。

### 优势与代价

- 优势：标准 Agent 入口减少样板代码；与 LangGraph 的路径连贯；模型和工具生态有利于快速对照试验。
- 代价：抽象层次多，故障可能发生在模型包、工具适配、中间件、图状态或服务层，需要端到端追踪。
- 不宜：把高度确定的审批链、订单交易链全部交给高层默认循环，却不设计显式状态和副作用账本。

### 生产验收重点

验证同一工具在不同模型下的参数生成、错误恢复和并行调用行为；Middleware 的授权是否发生在动作前；是否存在子 Agent 或直连工具绕过；模型切换是否改变结构化输出、工具选择和历史消息要求。

LangChain 的生态优势不意味着必须购买 LangSmith。LangSmith 是配套产品，部署和商业约束应独立确认；核心库的开源许可不能代替配套服务条款。[LangSmith 部署选项](https://docs.langchain.com/langsmith/platform-setup)

## 06 LangGraph 详细评估

### 状态建模与恢复机制

图中的状态更新、并行写入和合并规则是应用设计的一部分。Checkpointer 保存执行状态，使中断、查看状态和恢复成为可能；开发环境的内存存储不能替代生产持久后端。[Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)

`interrupt` 允许挂起并在提供恢复值后继续。恢复时节点可能重新执行，因此中断之前的副作用要幂等或隔离到适当任务单元。不能把“从中断恢复”理解为 CPU 指令从暂停位置原样继续。[Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)

### 优势与代价

**本报告判断：**LangGraph 适合流程中既有模型判断，又有确定性路由和需要检查的状态转移。对复杂审批、反复检索、多阶段研究等场景，显式图通常比堆叠提示词更容易评审。

代价在于需要维护 State Schema（状态结构）、Reducer（合并规则）、子图边界和版本迁移。图越细，状态写入和调试负担越大；把每个函数都做成节点未必有价值。图是逻辑编排，不代表每个节点都应成为独立微服务。

### 多 Agent 与企业工作流的界限

子图适合隔离任务上下文与协调步骤；跨服务的长期定时、补偿和业务交易，则需要额外考虑企业工作流。若引入 Temporal，明确外层负责业务生命周期，内层负责有界 Agent 子任务；不允许两边同时重试同一外部写操作。

### 必测场景

节点完成后保存失败、并行节点部分失败、同一线程同时写入、在审批前后分别崩溃、工具已提交但响应丢失、旧图运行时发布新图。通过测试再判断恢复等级，不以“支持 Checkpoint”直接给高可靠评分。

## 07 AgentScope 2.x 详细评估

### 应按新版重新判断

AgentScope 2.0 的整体方向覆盖 Agent 与服务化，而旧 Runtime 仓库公告建议迁移到主项目。新方案应评估主项目发布版本，旧系统应盘点 AgentApp、协议适配、状态后端和部署脚本，再逐步迁移。[Runtime 官方公告](https://github.com/agentscope-ai/agentscope-runtime)

发布页 v2.0.8 包含 A2A、Pipeline、上下文压缩等更新；但部分最新文档已经指向 dev 版本。报告不把开发文档中的接口细节认定为稳定发行版的保证。[发布记录](https://github.com/agentscope-ai/agentscope/releases)

### 开发与多 Agent

当前结构更适合以 Agent、Toolkit、事件与服务工作区为中心组织能力。Agent Team 文档描述主 Agent 与工作者之间的任务协作；这是可复用的协作方式，实际是否优于固定图取决于任务独立性、上下文和验证器。[Agent Team 文档](https://docs.agentscope.io/versions/2.0.9dev/en/deploy/agent-team)

**本报告判断：**研究助手、工具丰富的企业助手、工作区任务和服务化试点值得优先评估；若业务要求非常明确的状态转移图，应同时比较其 Pipeline 与 LangGraph 的控制复杂度，不预设自由委派更优。

### 权限、服务与恢复边界

Permission System 提供工具/资源控制与用户确认路径，但它仍需接入企业用户、资源和审批凭据。绕过确认的模式不能在生产中被当作默认便利开关。[权限文档](https://docs.agentscope.io/versions/2.0.9dev/en/building-blocks/permission-system/overview)

服务层有状态、会话和部署相关能力；PoC 要确认哪些数据只在内存、哪些持久化，后台任务和子 Agent 在跨副本恢复时如何处理。多租户 API 的存在不等于通过了租户隔离验收。[服务架构](https://docs.agentscope.io/versions/2.0.9dev/en/deploy/agent-service)

### 主要风险与验证重点

集成面更广时，必须分别评估本地工具、沙箱后端、MCP、Skill、服务入口和后台任务的安全边界。选择需要的模块而不是全开；对原 Runtime 的迁移，先还原旧数据与执行语义，再评估新增能力。文档中“production-ready”属于项目定位，不替代企业压测。

## 08 Spring AI Alibaba 详细评估

### 与 Spring AI 的关系

Spring AI 主要提供模型接入、工具、向量存储与其他 AI 基础抽象。Spring AI Alibaba 在其上提供 Agent 和 Graph 等能力。已有 Spring Boot 系统可复用依赖注入、配置、监控与业务服务，减少 Python 服务的额外接口成本。[Spring AI 参考](https://docs.spring.io/spring-ai/reference/)、[Alibaba 概览](https://java2ai.com/docs/overview/)

### 编程模式

简单 Agent 从 `ReactAgent` 开始；多步协作使用顺序、并行、路由或循环模式；需要直接控制时使用 Graph API。其图层负责状态、条件边、并发与子图等流程机制，适合把现有业务函数与模型节点组合。[Graph 官方指南](https://java2ai.com/docs/frameworks/graph-core/quick-start/)

Hooks / Interceptors 是上下文与调用控制的扩展点。需要验证拦截发生顺序、异步异常传播、恢复后重复进入的行为，以及工具结果是否在记录前脱敏。[Hooks 官方指南](https://java2ai.com/docs/frameworks/agent-framework/tutorials/hooks/)

### 对 Java 企业的价值

**本报告判断：**价值主要来自组织和系统适配，而不是 Java 必然更快。已有权限服务、数据库、消息中间件和运维体系可以直接复用；同一团队能维护业务工具与 Agent。对以数据科学实验为主的团队，Python 的数据处理依赖可能更方便，需要按总成本比较。

### Admin、Graph 和云能力分开验收

仓库包含 Admin、Studio、Graph、Agent Framework 和 Sandbox 等模块，管理界面不是引入核心库即自动获得。选择自建 Console 或复用 Admin 时，检查租户权限、审批、审计导出、模型密钥、依赖服务与商业边界。[仓库模块说明](https://github.com/alibaba/spring-ai-alibaba)

A2A / Nacos 集成也不要求所有企业都引入 Nacos。若已有注册发现方案，应先验证是否能独立接入所需协议；不为了一个 Agent 强制重构整个服务治理体系。

### 版本风险

稳定版本与 2.0 里程碑的 Spring 底座不同；升级需要依赖收敛、JDK/Boot 兼容测试和旧任务恢复测试。已有 AgentScope 适配的发布说明值得关注，但不能跨版本直接拼装。[发布记录](https://github.com/alibaba/spring-ai-alibaba/releases)

## 09 同一业务场景下如何实现与比较

基准场景：员工提出“查询有权限的工单，分析处理情况，申请批量关闭”。前半段只读，后半段有审批和副作用。

| 步骤 | LangChain + LangGraph | AgentScope 2.x | Spring AI Alibaba | Claude Agent SDK | DeepSeek Harness |
|---|---|---|---| --- | --- |
| 任务创建 | 平台 Run + 图调用关联 | 平台 Run 关联会话/服务执行 | 平台 Run + Graph 调用关联 | 平台 Run 关联 SDK 会话与执行 | 平台 Run 关联 Profile、会话与版本 |
| 授权检索 | 受控工具 + 模型接口 | Toolkit 中注册受控工具 | Spring 工具服务 + Agent | 受控工具 / MCP 接业务服务 | 工具插件接同一业务服务 |
| 形成提案 | 结构化结果 + 图状态 | 结构化结果 + 服务状态 | ReactAgent 结果 + Graph State | SDK 结果形成业务草稿 | 结果和会话事件形成业务草稿 |
| 人工批准 | 挂起机制 + 独立审批记录 | 确认机制/平台审批，验证持久性 | HITL/Hook + 独立审批记录 | Hook / 回调 + 独立审批记录 | 审批策略 + 独立审批记录 |
| 真正写入 | 统一工具执行器 | 同一工具执行器 | 同一工具执行器 | 同一工具执行器 | 同一工具执行器 |
| 崩溃恢复 | Checkpoint + 操作账本 | 状态后端 + 恢复能力测试 + 账本 | Graph 存储 + 恢复测试 + 账本 | 会话恢复 + 操作账本与结果确认 | 会话日志 / 状态后端 + 操作账本 |
| 业务验收 | 比较下游真实工单状态 | 相同 | 相同 | 相同 | 相同 |

不能让各框架使用不同模型、不同工具权限和不同样本后比较成功率。统一业务执行器的好处，是把模型/框架差异与交易安全差异分开。

在 LangGraph 中可把授权、检索、提案、等待、提交、对账做成显式状态；AgentScope 侧比较“预置 Agent 服务 + 确认路径”是否能表达同等边界；Java 侧比较 Graph 与已有业务服务组合的维护成本。新增 Claude SDK 方案验证权限回调与外部流程的边界；DeepSeek Harness 方案验证插件组合、会话持久性与更新兼容。该设计是待实现的 PoC，不是已运行的结果。

## 10 其他常用框架应放在哪个候选位置

| 候选 | 值得评估的情况 | 本次定位与边界 |
|---|---|---|
| AgentScope Java | Java 团队需要以 Agent 循环、工作区和服务化为中心 | 与 Spring AI Alibaba 并列对照；Python/Java 版本分别管理 |
| LangChain4j | JVM 项目希望使用较直接的模型、工具与 RAG 抽象 | 可作为较轻量的 Java 接入基线 |
| Pydantic AI | Python 团队强调类型化输入输出与较简洁应用接口 | 与 LangChain 高层接口对照，不只比较代码行数 |
| LlamaIndex | 数据接入、索引、知识工作流是主要难点 | 可是数据层候选，也有 Agent 能力；不要简化成只做 RAG |
| CrewAI | 团队希望按角色、任务与流程快速组织协作 | 同样测试确定性、状态恢复和协作成本，不以角色数判断效果 |
| Google ADK | 已有相关云生态或希望复用其 Agent 开发体系 | 按目标语言、运行环境与部署约束单独验证 |

本表为扩展候选扫描，调研深度低于主要比较对象，不据此做统一排名。官方入口：[AgentScope Java](https://github.com/agentscope-ai/agentscope-java)、[LangChain4j](https://docs.langchain4j.dev/)、[Pydantic AI](https://pydantic.dev/docs/ai/overview/)、[LlamaIndex](https://developers.llamaindex.ai/python/framework/)、[CrewAI](https://docs.crewai.com/)、[ADK](https://adk.dev/)。

## 11 开源许可、成本与运维比较

| 项目层 | 核验许可/边界 | 实际成本重点 |
|---|---|---|
| LangChain 核心 | MIT | 集成包维护、执行服务、数据库与运维 |
| LangGraph 核心 | MIT | 持久状态、并发执行、恢复和图版本管理 |
| AgentScope 核心 | Apache-2.0 | 状态/工作区/沙箱/服务部署及版本迁移 |
| Spring AI Alibaba 核心 | Apache-2.0 | Spring 依赖栈、配套模块和平台整合 |
| Claude Agent SDK | 官方概览列明适用的 Anthropic 商业条款；组件许可按各 LICENSE 区分 | SDK / 运行环境、模型调用、隔离、存储和企业集成 |
| DeepSeek Harness | 主仓库 MIT；第三方依赖分别核对 | 预览版迁移、插件治理、运行环境及模型服务 |
| 配套 SaaS、托管部署、模型和沙箱服务 | 不由核心仓库许可统一覆盖 | 授权方式、部署条款、调用费和数据留存 |

许可依据：[LangChain LICENSE](https://github.com/langchain-ai/langchain/blob/master/LICENSE)、[LangGraph LICENSE](https://github.com/langchain-ai/langgraph/blob/main/LICENSE)、[AgentScope 仓库](https://github.com/agentscope-ai/agentscope)、[Spring AI Alibaba 仓库](https://github.com/alibaba/spring-ai-alibaba)。新增依据：[Claude SDK 官方条款说明](https://code.claude.com/docs/en/agent-sdk/overview)、[DeepSeek 仓库许可](https://github.com/deepseek-ai/deepseek-harness)。上线按锁定版本及其实际依赖许可证复核。

**不要写未经同条件验证的“Java 比 Python 快几倍”或“某框架吞吐最高”。**模型调用通常占据显著延迟，但并发连接、阻塞工具、状态写入和队列也会成为瓶颈。报告区分框架开销与模型耗时，用实测决定。

总成本包含模型、嵌入/重排、工具服务、数据库、观测、沙箱、人工评审和研发运维。一个更省 token、但导致人工返工增加的方案未必更便宜；集成框架减少自建工作，也可能增加升级耦合。

## 12 可执行的 PoC 与评分方法

### 先做硬门槛

必须满足部署位置、目标语言、模型能力、许可证、动作前授权、租户隔离与关键写操作对账。任一关键安全门槛失败，不以其他性能分数抵消。

### 三种负载与故障集

| 测试组 | 场景 | 指标/判定 |
|---|---|---|
| 基础工具 | 只读问答、结构化输出、缺参数、无可用工具 | 任务成功、格式、工具误选、拒答质量 |
| 状态与恢复 | 多步检索、挂起审批、进程退出、断线重连 | 恢复范围、重复动作、状态一致、恢复耗时 |
| 多 Agent | 独立并行研究、子任务失败、共享资源冲突 | 对单 Agent 的增益、协调成本、结果可追溯 |
| 安全 | 恶意工具结果、跨租户 ID、越权参数、撤销后恢复 | 关键用例中禁止操作不得执行；不代表证明绝对安全 |
| 运维 | 模型 429、网关超时、数据库故障、观测失效 | 限流、背压、降级、丢失记录和人工接管 |
| SDK 权限 | Claude 允许规则、禁用规则、Hook 与人工输入组合 | 验证哪些调用不经过回调；服务端授权仍有效 |
| Harness 组合 | DeepSeek Profile / 插件替换、卸载、上下文模式对照 | 资源释放、配置可复现、事件留存与外部动作不重放 |
| 版本 | 模型/Prompt/图/Skill 更新与回滚 | 旧任务处理、兼容性、结果与账本可复现 |

建议两层测试：使用模拟模型/工具测框架开销与确定性边界；使用同一真实模型测完整业务效果。样本按任务类型与风险分层，报告置信区间和失败样例，避免只拿几个成功示例评估。

示例评分权重：可靠性 25%、业务质量 25%、团队适配 20%、治理可控 15%、总成本 10%、退出能力 5%。权重为建议，需经项目评审调整；本报告未填写虚构得分。

### 建议两周验证节奏

第 1–2 天冻结任务集、模型、权限与工具接口；第 3–5 天实现单 Agent 与只读基线；第 6–8 天加入审批、持久化和故障注入；第 9–10 天执行同条件负载、回归和迁移检查。复杂企业接入可能超过两周，该节奏不是完整平台交付承诺。

最终产物应包括锁定依赖清单、代码、任务样本、原始指标、失败轨迹、成本明细、差异表和 ADR（Architecture Decision Record，架构决策记录）。

## 13 面向企业能力平台的推荐组合

### 方案 A：Python 工程控制优先

现有网关/SSO → 企业 Run API → LangChain 高层 Agent 或 LangGraph 显式图 → 受控工具网关与模型网关 → 业务系统/知识服务。配套持久数据库、观测和评测；跨天业务事务按需接入现有工作流。

适合需要精细调试、流程边界清楚、愿意自己建设控制面和治理集成的团队。不要再为同一循环额外套 AgentScope；确有独立能力时以服务契约集成。

### 方案 B：AgentScope 集成验证优先

以 AgentScope 2.x 稳定版的 Agent 与服务能力为基础，复用企业 SSO、模型网关与业务工具。优先验证服务状态、工作区、用户确认、后台任务和租户边界。对于其自带的目录/记忆/UI，与企业已有资产选择一个权威管理方。

适合希望降低服务化胶水代码、同时接受较宽框架集成面的团队。先验证需要的能力再开启更多模块；旧 Runtime 迁移单列工作包。

### 方案 C：Java / Spring 整合优先

Spring Boot 控制服务 → Spring AI Alibaba Agent Framework / Graph → Spring AI 模型与工具 → 企业能力网关和业务系统。用同样的业务集与 AgentScope Java 对照。只做固定问答时也应保留更轻量的 Spring AI / LangChain4j 基线。

适合已有 JVM 工程、运维和业务积累的团队。不要为了复用全部生态而无条件引入 Admin、Nacos 与新沙箱；新增模块必须有明确需求与负责人。

### 方案 D：Claude 执行能力嵌入优先

企业 Run API → 隔离的 SDK 执行环境 → Claude Agent SDK → 受控企业工具 / MCP → 业务服务。保留外部审批、账本和任务状态，明确 SDK 会话与企业 Run 的映射。适合复用文件与工具任务能力的团队；需要自定义复杂图时可作为图内有界阶段，避免双层重试。

### 方案 E：DeepSeek Harness 插件化试点

企业试点入口 → 固定版本的 Profile → 模型 / 工具 / 权限 / 存储插件 → 企业服务。先验证只读任务，再验证审批写入与恢复；插件或模式升级须跑相同回归集。适合研究运行策略、工具组合和可替换环境的团队，预览版本的变化成本单列。

### 对既有指南的影响

将原“LangGraph 默认参考”解释为 Python 显式编排方案，而非全企业唯一默认；新增 AgentScope 2.x 与 AgentScope Java 候选；将 Spring AI Alibaba 从补充项提升为 Java 平台主候选；单独记录旧 AgentScope Runtime 的迁移状态。新增 Claude Agent SDK 作为嵌入式执行候选、DeepSeek Harness 作为插件化运行环境候选，避免只按语言把它们塞入同一类框架。统一能力目录、授权、业务账本和评测集仍应独立于框架保留。

## 14 评审时应坚持的判断

Function Calling（函数调用）和 ReAct（Reasoning and Acting，推理与行动）已经是这类框架的基本能力，真正的选型差异在于：谁控制循环，谁保存状态，谁能阻止动作，谁负责故障恢复，以及这些机制能否与企业已有系统配合。

没有一个框架因为提供 Memory（记忆）、Checkpoint（检查点）、MCP 或多 Agent 就自动成为完整企业 AI 平台。先选一个主执行体系，保留框架外的身份、授权、幂等、审计和评测资产，使用真实业务验证后再扩展。

## 15 Claude Agent SDK 详细评估

### 适用性与开发方式

SDK 将执行循环和工具环境交给应用调用，开发者通过 Python / TypeScript 消费运行消息并连接企业能力。与普通模型客户端相比，少写执行循环；与 LangGraph 相比，重点是复用运行行为，不是把所有业务状态显式画成图。[Quickstart](https://code.claude.com/docs/en/agent-sdk/quickstart)

**本报告判断：**代码维护、文件分析、研究与工具执行助手值得优先验证。企业已拥有成熟流程时，可把 SDK 封装成一个阶段；不能因此省略任务 API、认证、工作区隔离、预算与业务状态管理。

### 权限与恢复的关键差异

`allowed_tools` 属于自动许可规则，不应当作唯一工具可用性白名单。`canUseTool` 不是所有调用的必经入口；全调用策略参考 `PreToolUse` Hook，真实业务授权仍在服务端。必须测试自动许可、禁用、人工输入及模式组合。[权限规则](https://code.claude.com/docs/en/agent-sdk/permissions)、[Hooks](https://code.claude.com/docs/en/agent-sdk/hooks)

Session 可用于上下文恢复与分叉，但不等于事务恢复；外部提交成功而回执丢失仍需幂等与结果确认。部署评审要记录 SDK、运行二进制、模型和会话存储版本。[Sessions](https://code.claude.com/docs/en/agent-sdk/sessions)

### 优势、成本与退出路径

优势是复用既有运行能力及工具生态；成本是理解其默认行为、权限与部署要求，并维护与企业控制面的边界。跨框架迁移优先保留工具协议、业务账本和评测集，不承诺内部会话格式可无损通用。

建议 PoC：固定同一支持助手任务集，比较从请求到业务结果的成功率和成本；同时测拒绝写入、子任务失败、取消和重启。演示代码能完成任务不能替代这些验收。

## 16 DeepSeek Harness 详细评估

### 定位与运行组合

这里是官方 dsh 项目，不是 DeepSeek 模型 API 的泛称，也不是 LangChain Deep Agents。官方当前为 Developer Preview（开发者预览版），明确提示兼容性变化。应固定版本或提交、Profile、插件与模型共同验证。[官方仓库](https://github.com/deepseek-ai/deepseek-harness)

Cordis 负责插件组合与生命周期，Profile 与 Bundle 组织启动配置，模型、工具、会话、循环等能力可替换。当前架构包含 SDK 启动路径，不能只当作浏览器产品，也不是纯 Python 图编排库。[Architecture](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md)

### 模式与工程价值

官方展示标准、PTC（Programmatic Tool Calling，程序化工具调用）、极简与创造等模式。PTC 通过程序组织工具操作，可能减少部分往返，但带来执行资源、错误传播和授权边界的额外验证。模式名称不代表成功率排名。[官方介绍](https://www.deepseek.com/harness/)

**本报告判断：**适合希望探索工具粒度、上下文装配、插件与运行模式的团队。价值在运行层可组合，不是保证某模型更强。每次只改变一个因素，在同模型和总预算下做对照，才能判断优化来自哪里。

### 事件、恢复与插件边界

会话事实与运行事件具有不同职责，不应假设每个事件都持久化。轨迹有助定位，但回放默认应读取既有记录，不重新执行外部写工具。插件卸载释放注册与资源，不自动撤销已提交业务；插件化不等于不可信插件间的强隔离。

### 主要风险与退出路径

预览版升级、配置覆盖、插件依赖、状态兼容与沙箱后端是重点。试点先只读再审批写入，保存配置快照和回归结果。对企业稳定接口使用独立服务协议，不直接把插件内部对象作为跨系统契约。可以在明确范围内验收使用，不将“开发者预览”视为永久否定，也不将功能展示当稳定承诺。

### 六个主要对象的评审归纳

LangChain 侧重高层应用组合；LangGraph 侧重显式状态控制；AgentScope 侧重 Agent 与服务化；Spring AI Alibaba 侧重 Spring 企业集成与图；Claude Agent SDK 侧重复用执行循环；DeepSeek Harness 侧重可替换运行组合。选型比较的共同结果应是业务任务质量、可靠性和总成本，而不是把抽象层不同的产品做无依据的总分排名。

---

报告核验日期不代表未来版本兼容承诺。

© HansenUp · 内容采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh-hans) 许可。
