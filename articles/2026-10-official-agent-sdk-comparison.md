# 四家官方 Agent SDK 怎么选：Google ADK、Claude Agent SDK、OpenAI Agents SDK、DeepSeek Harness

> 作者：hansenup（GitHub [@HansenUp](https://github.com/HansenUp)）
> 信息截至：2026-10-29（北京时间）
> 性质：基于官方文档、GitHub 仓库和发布记录的案头调研。没有逐个在生产环境使用过，也没做性能实测。

模型厂商们都下场做 Agent 开发包了。Google 有 ADK，Anthropic 把 Claude Code 做成了库，OpenAI 有 Agents SDK，DeepSeek 也开源了 DeepSeek Harness。

之前我写过一份企业 Agent 框架的调研报告，核心是四个问题：**谁控制循环、谁保存状态、谁能在动作前拦住它、出故障了谁负责恢复**。这篇用同一把尺子量一量这四家"官方"的东西，再加一个官方 SDK 特有的问题：**它会把你往哪家模型、哪朵云上拉。**

先打个预防针：这四个东西不在同一层，按总分排名没有意义。下文给的是"什么场景更合适"，不是冠军。

---

## 目录

- [TL;DR](#tldr)
- [一、四个问题 + 一个绑定问题](#一四个问题--一个绑定问题)
- [二、Google ADK](#二google-adk)
- [三、Claude Agent SDK](#三claude-agent-sdk)
- [四、OpenAI Agents SDK](#四openai-agents-sdk)
- [五、DeepSeek Harness](#五deepseek-harness)
- [六、完整对比表](#六完整对比表)
- [七、场景决策指南](#七场景决策指南)
- [八、企业落地注意事项](#八企业落地注意事项)
- [九、我没核实到的东西](#九我没核实到的东西)
- [十、说明与免责](#十说明与免责)
- [参考资料](#参考资料)

---

## TL;DR

- **它们不在同一层。** Google ADK 是全栈开发框架（5 种语言，有图式工作流、评测和部署）。Claude Agent SDK 是"把 Claude Code 当库用"，循环跑在 CLI 子进程里。OpenAI Agents SDK 是原语很少的轻量运行时。DeepSeek Harness 是"一切皆插件"的 Agent 运行层（连循环本身都能换），目前是开发者预览。
- **选型先看四件事：** 谁控制循环、谁保存状态、谁能在动作前拦截、谁负责恢复。再看它把你往哪家模型、哪朵云、哪个追踪后台上拉。
- **粗略的场景对应：**
  - 已在 Google Cloud，或团队主力是 Java / Go / Kotlin → **Google ADK**
  - 需要一个现成能读写文件、跑命令、改代码的执行体 → **Claude Agent SDK**，作为一个阶段嵌进你的系统，放在隔离容器里
  - Python / TS 的业务编排，审批多、要暂停续跑 → **OpenAI Agents SDK**，长流程配持久执行引擎
  - 研究 harness 架构、想用 DeepSeek 模型 → **DeepSeek Harness**，只做受控试点，官方明确说不能当生产可用
  - 要做模型无关的企业平台 → 任何一家都只当"执行层"，**状态、审批、审计、幂等留在你自己这层**
- **最容易踩的坑：** 把"能恢复"当成"有事务"；Claude Agent SDK 的会话默认写在本地盘；OpenAI 的追踪默认开启并上报 OpenAI；0.x / 预览版不锁版本。

### 一张选型速查表

| 你的情况 | 优先看 | 理由 | 要小心 |
|---|---|---|---|
| 已在 Google Cloud；Java / Go / Kotlin 团队 | Google ADK | 5 种语言；一条命令部署到 Google Cloud；内置 `adk eval` | 2.0 新能力以 Python 为先，其他语言进度不一 |
| 要能读写文件、跑命令的"干活"Agent | Claude Agent SDK | 直接复用 Claude Code 的工具、循环和上下文管理 | 只能用 Claude 模型；子进程 + 本地盘状态，要自己处理持久化和隔离 |
| Python / TS 业务流程，审批、暂停、续跑 | OpenAI Agents SDK | 原语少；`needs_approval` + `RunState` 序列化；有 Temporal / Dapr / Restate 集成 | 追踪默认上报 OpenAI；非 OpenAI 模型的第三方适配器是 beta |
| 研究插件化 harness；用 DeepSeek 模型 | DeepSeek Harness | 循环、工具、会话日志全是插件，可替换 | 开发者预览，只有预发布版本，官方说会有破坏性变更、不可视为生产可用 |
| 跨系统、跨天的长事务 | 任一 SDK + 企业工作流引擎 | SDK 负责"想"和"做一步"，工作流负责"记账"和"重试" | 只让一个组件负责重试 |

---

## 一、四个问题 + 一个绑定问题

调工具、接 MCP、多 Agent 协作、人工审批，这四家基本都能做，功能清单上的"✅"看不出区别。区别在于下面这几个问题的答案。

1. **谁控制循环？** 是你进程里的一个对象在驱动"想 → 调工具 → 再想"，还是 SDK 拉起一个子进程，你只收消息流？前者容易嵌入、调试，后者开箱能力强，但多了一层进程边界。
2. **谁保存状态？** 会话、工具结果、中间产物存在哪：内存、本地文件、你的数据库，还是厂商的服务端？容器重启、横向扩容时它还在不在？
3. **谁能在动作前拦住它？** 写文件、调外部 API、转账之前，有没有一个可靠的钩子能拒绝或者请人审批？审批要等很久的时候，运行能不能先停下来、序列化，之后再续上？
4. **出故障谁负责恢复？** 进程崩了、模型超时了、审批隔夜了，从哪里续跑？续跑时外部系统已经执行过的动作会不会再执行一次？

外加一个官方 SDK 才特别需要问的问题：

5. **它把你往哪拉？** 模型（只能用自家，还是模型无关）、部署（是否有一条命令上自家云）、观测（追踪默认发到哪里）。**"模型无关"不等于"平台无关"**：会话、追踪、托管放在哪，才是真正的绑定点。

---

## 二、Google ADK

官方站点：[adk.dev](https://adk.dev/) ｜ 仓库：[google/adk-python](https://github.com/google/adk-python)、[adk-java](https://github.com/google/adk-java)、[adk-go](https://github.com/google/adk-go)、[adk-js](https://github.com/google/adk-js)、[adk-kotlin](https://github.com/google/adk-kotlin)

### 定位

官方说法是开源的 Agent 开发框架，"Build production agents, not prototypes"，提供 Python、TypeScript、Go、Java、Kotlin 五种语言（见 [adk.dev](https://adk.dev/)）。在四家里，它最像传统意义上的"框架"：从写 Agent、编排工作流、评测、调试，到部署，一条龙都有。

### 架构与循环

- 循环由你进程里的 Runner 驱动 Agent 执行，你可以直接在自己的服务里调用。
- ADK 2.0 加入了 **Workflow Runtime**：用图来编排，支持路由、并行分发 / 汇合、循环、重试、状态和人工介入，可以嵌套（见 [adk-python README](https://github.com/google/adk-python)）。官网的说法是"把确定性代码和自适应的 AI 推理编织在一起"。对企业来说，这意味着可以把"必须按顺序做的步骤"写死，只把需要判断的部分交给模型。
- 官网 FAQ 强调了上下文管理：会话、记忆、工具输出、产物会被组装成结构化视图，自动过滤无关事件、总结较早的轮次、懒加载产物。

### 状态 / 会话 / 记忆

- `SessionService` 有多种实现：内存版、`VertexAiSessionService`（Google Cloud 托管）、`DatabaseSessionService`（PostgreSQL / MySQL / SQLite 等），见 [Sessions](https://adk.dev/sessions/) 和 [Session](https://adk.dev/sessions/session/)。
- `MemoryService` 负责跨会话的长期记忆，见 [Memory](https://adk.dev/sessions/memory/)。
- Python v2.13.0 新增了 MongoDB 的 Session / Memory 服务，标注为实验性（见 [v2.13.0 发布说明](https://github.com/google/adk-python/releases/tag/v2.13.0)）。

### 工具与 MCP

工具支持 MCP、OpenAPI，也内置 Google 生态的工具，例如 Google Search（见 README 和官网示例）。

### 多 Agent

支持多 Agent 层级（父 Agent 委派给子 Agent），2.0 加入了用于 Agent 间委派的 Task API。跨进程、跨框架的 [A2A](https://adk.dev/a2a/) 支持目前标注为实验性，覆盖 Python / Go / Java。

### HITL / 审批

- 有回调和插件机制，可以在模型调用、工具调用前后插入逻辑。
- **Tool Confirmation**（工具调用前请人确认）：[文档页](https://adk.dev/tools-custom/confirmation/)仍标注为 Experimental，但 Python v2.13.0 把 `TOOL_CONFIRMATION` 等 12 个特性从实验性提升为稳定（见 [v2.13.0 发布说明](https://github.com/google/adk-python/releases/tag/v2.13.0)）。其他语言的状态我没有逐一核对。
- Workflow Runtime 本身也支持人工介入节点。

### 恢复

Python 1.16+ 可以配置 Resumability，按 invocation ID 续跑中断的运行（见 [Resume](https://adk.dev/runtime/resume/)）。

### 追踪与评测

- 观测基于 OpenTelemetry（日志、指标、追踪），见 [Observability](https://adk.dev/observability/)。部署到 Google Cloud 时可以直接用 Cloud Trace。
- 内置评测：`adk eval`，见 [Evaluate](https://adk.dev/evaluate/)。v2.13.0 的评测把模型、工具和框架耗时拆开统计。
- ⚠️ v2.13.0 有一个**破坏性变更**：遥测默认改用 OTel GenAI 语义约定 v2，原来的 `call_llm` span 被移除。如果你的监控大盘依赖旧 span 名，升级前要检查。

### 部署

官网 FAQ 写的是"deploy anywhere"：可以自己容器化部署，也可以一条命令部署到 Google Cloud 的 Agent Runtime（Agent Platform）、Cloud Run 或 GKE，部署后直接获得托管基础设施、认证和 Cloud Trace。

### 语言 / 版本 / 许可（2026-10-29 核对，日期为 GitHub 发布时间 UTC）

| 语言 | 最新版本 | 发布日期 |
|---|---|---|
| Python | v2.13.0（PyPI 一致，需 Python 3.11+） | 2026-10-23 |
| Java | v1.13.0 | 2026-10-21 |
| Go | v1.9.0；v2 线 v2.6.0 | 2026-10-15；2026-10-13 |
| TypeScript | main-v2.4.0（npm `@google/adk` 2.4.0） | 2026-10-28 |
| Kotlin | v1.5.0 | 2026-10-28 |

许可：全部是 Apache-2.0。
GitHub Star 快照（2026-10-29）：Python 约 22.0k、Go 约 8.9k、Java 约 1.8k、JS 约 1.5k、Kotlin 约 0.3k。

模型：针对 Gemini 优化，但模型无关，可以接 Claude、LiteLLM、Ollama、vLLM 等，见 [Models](https://adk.dev/agents/models/)。

### 最适合的场景

- 已经在 Google Cloud 上，想一条龙解决开发、部署、追踪。
- Java / Go / Kotlin 团队。四家里只有 ADK 官方覆盖了这些语言。
- 流程里确定性的部分多，想用图式工作流把它们写死，只把判断交给模型。
- 需要内置评测。

### 优点

- 语言覆盖最广，发版节奏快（Python 最近三个版本间隔约 10 天）。
- 框架能力完整：工作流、多 Agent、会话 / 记忆、评测、部署都有官方方案。
- 模型无关，会话存储可以自选，包括自建数据库。

### 缺点

- 概念和 API 面最大，学习成本也最高。
- 各语言能力不完全对齐，2.0 的新能力多数先出现在 Python。
- 最顺滑的路径是 Google Cloud，离开它，部署、追踪都要自己补。

### 注意事项

- 升级前读发布说明。v2.13.0 的遥测 span 变更就是破坏性的。
- 注意"实验性"标记。文档页和代码里的稳定性标记可能不同步（Tool Confirmation 就是例子），以你所用版本的代码为准。
- `VertexAiSessionService` 很方便，但它把会话放进了 Google Cloud。如果要求可迁移，选数据库实现。

---

## 三、Claude Agent SDK

官方文档：[Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview) ｜ 仓库：[claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python)、[claude-agent-sdk-typescript](https://github.com/anthropics/claude-agent-sdk-typescript)

### 定位

官方一句话："Build production AI agents with Claude Code as a library"。它提供与 Claude Code 相同的工具、Agent 循环和上下文管理，可以在 Python 和 TypeScript 里编程调用。

官方文档里有一张很有用的对照表，把 Anthropic 自家的几种形态分得很清楚：

- **Agent SDK**：在你自己运维的进程里嵌入 Claude Code 的 Agent。
- **Claude Code CLI**：终端里的交互式使用。
- **Client SDK**：直接调 Claude API，循环你自己写，或者用 beta 的 tool runner。
- **Managed Agents**：由 Anthropic 托管的 harness，会话跑在 Anthropic 管理的云沙箱，或你自己基础设施上的自托管沙箱。

所以，如果你想要的是"自己掌控循环"，Anthropic 自己给的答案是用 Client SDK，而不是 Agent SDK。

### 架构与循环

- `query()` 会拉起一个 `claude` CLI 子进程，通过 stdio 通信。这个子进程持有 shell、工作目录，以及本地磁盘上的 JSONL 会话记录（见 [Hosting](https://code.claude.com/docs/en/agent-sdk/hosting)）。
- **一个会话对应一个子进程。** 循环在子进程里跑，你的代码收的是消息流。
- 其他语言可以直接把 CLI 当子进程跑，用 `-p` 和 `--output-format json` 驱动同一个循环（见 overview）。

### 状态 / 会话 / 记忆

- 会话记录默认写在本地（`~/.claude/projects/` 下的 JSONL）。官方 hosting 文档写明：容器重启后本地状态会丢失。
- 要跨机器，可以配 SessionStore 适配器做镜像。注意它只镜像会话记录，并且镜像写入是 best-effort 的，失败会发出 `mirror_error`（见 [Hosting](https://code.claude.com/docs/en/agent-sdk/hosting)）。
- 会话支持 resume 和 fork（见 [Sessions](https://code.claude.com/docs/en/agent-sdk/sessions)）。会话保存的是**对话，不是文件系统**。文件改动的回滚要靠单独的 file checkpointing。
- 和 Claude Code 一样，会从项目的 `.claude/` 和 `~/.claude/` 加载 skills、命令和记忆（CLAUDE.md）。

### 工具与 MCP

- 内置工具开箱即用：读、写、编辑文件，运行命令，搜索网页。这是它和另外几家最大的区别。
- 支持 [MCP](https://code.claude.com/docs/en/agent-sdk/mcp) 连接外部工具和数据源。
- 支持插件：把 skills、agents、hooks、MCP server 打包，按本地路径加载。

### 多 Agent

支持 [Subagents](https://code.claude.com/docs/en/agent-sdk/subagents)：为聚焦的子任务派生专门的子 Agent。

### HITL / 审批

权限评估有固定顺序（见 [Permissions](https://code.claude.com/docs/en/agent-sdk/permissions)）：

**Hooks → deny 规则 → ask 规则 → 权限模式 → allow 规则 → `canUseTool` 回调**

deny 优先级最高，连 bypass 模式也绕不过它。[Hooks](https://code.claude.com/docs/en/agent-sdk/hooks) 可以在生命周期的关键点运行自定义代码。

这套机制很细，但顺序必须吃透。比如你以为 `canUseTool` 能兜住所有调用，实际上前面的 allow 规则或权限模式可能已经放行了。

### 追踪与评测

- 由 CLI 导出 OpenTelemetry 数据。追踪（traces）目前是 beta，需要设置 `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1`（见 [Observability](https://code.claude.com/docs/en/agent-sdk/observability)）。
- 内置评测：我没有在 Agent SDK 文档里找到。这不代表没有，只是我没找到。

### 部署

- 在你自己的容器 / 机器里运行子进程。官方 hosting 文档专门讲了容器化时的注意点。
- 不想自己运维可以考虑 Managed Agents，但它是另一个产品，不是 Agent SDK 的部署选项。

### 语言 / 版本 / 许可（2026-10-29 核对）

| 语言 | 最新版本 | 发布日期（UTC） |
|---|---|---|
| Python | v0.2.174（PyPI 一致） | 2026-10-24 |
| TypeScript | v0.3.312（npm latest 一致） | 2026-10-27 |

- **许可：** Python 仓库的 LICENSE 文件是 MIT，npm 包写的是 "SEE LICENSE IN README.md"。官方文档写明：**使用 Claude Agent SDK 受 Anthropic 商业服务条款约束**，具体组件以各自的 LICENSE 文件为准。所以别简单当成"MIT 开源随便用"。
- **模型：** Claude 模型，通过 Anthropic API，或 Amazon Bedrock、Google Cloud 上的提供方端点。
- **登录方式：** 官方明确说，未经批准，第三方开发者不得在自己的产品里提供 claude.ai 登录或其额度，要用 API key 认证。
- **品牌：** 官方有品牌指南，你的产品不能叫 "Claude Code" 或模仿 Claude Code 的视觉。

GitHub Star 快照（2026-10-29）：Python 约 8.4k、TypeScript 约 1.8k。

### 最适合的场景

- 需要一个**现成的、会干活的执行体**：代码修改、文件处理、调研整理、运维脚本。
- 想把它当作流水线里的**一个阶段**（"拿到任务 → 在隔离工作区里做完 → 交回结果"），而不是整个系统的大脑。
- 团队已经在用 Claude Code，想把同样的能力（skills、hooks、MCP）带进自己的服务。

### 优点

- 开箱能力最强，不用自己实现文件、shell、搜索这些工具。
- 权限 / Hooks 体系细粒度，deny 优先的设计符合企业直觉。
- 会话可以 resume / fork，skills、插件生态和 Claude Code 共享。

### 缺点

- 只能用 Claude 模型。
- 子进程架构多了一层进程边界：资源占用、并发模型、故障排查都比纯库更复杂。
- 状态默认在本地盘，横向扩容和容器重启都要自己处理。
- 版本号迭代极快（TS 版几乎隔天一版），需要严格锁版本。

### 注意事项

- **一定放在隔离环境里跑**：它真的会执行命令、改文件。
- **容器化先想清楚会话持久化。** SessionStore 只做 best-effort 镜像，要监控 `mirror_error`。
- 会话 ≠ 文件系统快照。"恢复会话"不会把文件恢复到当时的状态。
- 检查商业条款是否适合你的分发方式，尤其是给你自己的客户用的时候。

---

## 四、OpenAI Agents SDK

官方文档：[openai.github.io/openai-agents-python](https://openai.github.io/openai-agents-python/) ｜ 仓库：[openai-agents-python](https://github.com/openai/openai-agents-python)、[openai-agents-js](https://github.com/openai/openai-agents-js)

### 定位

官方说法："lightweight, easy-to-use package with very few abstractions"，是早期实验项目 Swarm 的生产级升级。核心原语只有三个：

- **Agents**：带指令和工具的 LLM
- **Agents as tools / Handoffs**：Agent 之间委派任务
- **Guardrails**：校验输入和输出

文档里还有一句很坦诚的话：**如果你想自己掌控循环、工具分发和状态处理，就直接用 Responses API**。SDK 是给"想让运行时帮你管轮次、工具执行、guardrail、handoff、会话"的场景准备的。

### 架构与循环

- Runner 在你自己的进程里跑一个内置循环，直到任务完成，或者被审批打断（见 [Running agents](https://openai.github.io/openai-agents-python/running_agents/)）。
- 除了普通 Agent，还有 Sandbox agents（在隔离工作区里运行，支持可恢复的沙箱会话）、Realtime / Voice agents。

### 状态 / 会话 / 记忆

[Sessions](https://openai.github.io/openai-agents-python/sessions/) 有很多可选后端：SQLite、Redis、SQLAlchemy、MongoDB、Dapr、AdvancedSQLite，也有 `OpenAIConversationsSession`（存在 OpenAI 服务端），还有做自动压缩的 compaction session。

### 工具与 MCP

- 函数工具：任意 Python 函数可以自动生成 schema，用 Pydantic 校验。
- 支持 MCP server 工具调用，v0.24.0 还会把 MCP 的 annotations 暴露在转换后的工具上（见 [v0.24.0 发布说明](https://github.com/openai/openai-agents-python/releases/tag/v0.24.0)）。
- 也有 OpenAI 托管的工具（hosted tools）。

### 多 Agent

两种方式：**Handoffs**（把对话交给另一个 Agent）和 **Agents as tools**（经理式编排，把其他 Agent 当工具调用）。官方有专门的 Agent orchestration 页面讲怎么选。

### HITL / 审批

- 给工具标记 `needs_approval`，运行到这里会暂停，结果里带 interruptions。你把 `RunState` 序列化存起来，人工批准或拒绝后再续跑（见 [Human in the loop](https://openai.github.io/openai-agents-python/human_in_the_loop/)）。这在整个运行范围内生效，包括 handoff 之后和嵌套的 `as_tool` 调用。
- v0.24.0 的一个细节：有审批挂起时，自动压缩会推迟执行。
- Guardrails 可以和 Agent 并行做输入 / 输出校验，不通过就快速失败。

在四家里，这是我觉得**最接近企业审批流心智模型**的设计：暂停、存盘、隔天再续。

### 恢复

- `RunState` 可序列化后续跑。
- 长流程官方给了持久执行集成：Temporal、Dapr、Restate（见 [Running agents](https://openai.github.io/openai-agents-python/running_agents/)）。

### 追踪与评测

- **追踪默认开启**，数据发到 OpenAI 的 Traces 后台。可以加自定义 processor 发到别处，也可以关闭（见 [Tracing](https://openai.github.io/openai-agents-python/tracing/)）。
- 对使用 Zero Data Retention（ZDR）的组织，追踪功能不可用。
- 官方首页说追踪可以对接 OpenAI 的评测、微调、蒸馏工具。

### 部署

自托管，SDK 就是个库，跑在你自己的服务里。OpenAI 这边的托管能力主要体现在模型、Conversations 会话和追踪后台上。

### 语言 / 版本 / 许可（2026-10-29 核对）

| 语言 | 最新版本 | 发布日期（UTC） |
|---|---|---|
| Python | v0.24.0（PyPI 一致，Python ≥3.10） | 2026-10-15 |
| TypeScript / JS | `@openai/agents` 0.21.0 | 2026-10-20 |

许可：MIT。
GitHub Star 快照（2026-10-29）：Python 约 30.4k、JS 约 4.0k。

模型：默认用 Responses API 调 OpenAI 模型。非 OpenAI 模型通过适配器接入，其中第三方适配器标注为 beta；也有 MultiProvider（见 [Models](https://openai.github.io/openai-agents-python/models/)）。

### 最适合的场景

- Python / TS 的业务编排：客服、工单、审批类流程。
- 审批多、需要"暂停 → 存盘 → 隔天续跑"。
- 长流程，愿意配一个持久执行引擎（Temporal 等）。

### 优点

- 原语少、上手快，代码读起来就是普通 Python / TS。
- HITL 设计成熟：运行范围内的审批、可序列化的 `RunState`。
- 会话后端选择多，可以完全放在自己的存储里。
- 许可最省心（MIT）。

### 缺点

- 只有 Python 和 TS / JS。
- 默认路径深度绑定 OpenAI：Responses API、Traces 后台、Conversations。
- 本身没有"会干活"的内置工具集（文件、shell 这类要靠 Sandbox agents 或自己实现）。
- 0.x 版本，API 仍在演进。

### 注意事项

- **先决定追踪去哪。** 默认开启并上报 OpenAI。如果你的数据合规要求不允许出境或外发，上线前就要关掉或换 processor。
- `OpenAIConversationsSession` 把会话放在 OpenAI 服务端，方便，但也意味着状态不在你手里。
- 非 OpenAI 模型的第三方适配器是 beta，生产前要专门测。

---

## 五、DeepSeek Harness

官方页面：[deepseek.com/harness](https://deepseek.com/harness) ｜ 仓库：[deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) ｜ 文档：[deepseek-harness.github.io/deepseek-harness](https://deepseek-harness.github.io/deepseek-harness/)

### 先确认：它是真的，而且是官方的

DeepSeek 官网有 [DeepSeek Harness 页面](https://deepseek.com/harness)（标注预览版），直接链接到 GitHub 仓库和文档。仓库 README 写的是 "an open-source agent harness developed by DeepSeek AI"。

（小细节：`deepseek-ai` 这个 GitHub 组织没有显示"已验证"标识，我判断它是官方仓库，依据是官网直链。）

### 定位

严格说，它**不是一个传统意义上的 SDK**，而是一个 Agent 运行层（harness）/ 应用：

- 有 Web UI（`npx @deepseek-ai/dsh web`，默认 `127.0.0.1:3080`）和桌面端（官网列出 macOS Apple 芯片版和 Windows 版），基于 Electron。
- 同时提供 TypeScript 和 Python SDK，用来以编程方式驱动这个运行层。

README 第一屏就写着：**developer preview，"THERE WILL BE COMPATIBILITY-BREAKING CHANGES."**

### 架构与循环

核心理念是 **everything-is-a-plugin**，基于 [Cordis](https://github.com/cordiverse/cordis) 构建，设计思路见论文 [_A Programming Paradigm for Spatiotemporal Composability_](https://arxiv.org/abs/2608.25512)。

根据 [architecture.md](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/architecture.md)：

- 模型适配器、工具注册、会话日志，**连 Agent 循环本身**都是插件，可以通过配置替换。
- 用 **profile** 组合插件，内置模板有 `web`、`headless`、`sdk`、`sdk-minimal`、`acp`。
- TS 和 Python SDK 都是**拉起 `dsh --profile sdk` 子进程**，走 stdio 上的 JSON-RPC 通信。Python SDK 的运行时 wheel 里打包了 `dsh` CLI。
- 文件系统和子进程提供方共享同一个"执行世界"，指向远程沙箱时，Bash、PTY、LSP 会一起迁过去。

所以"谁控制循环"的答案是：**一个可替换的插件，跑在 dsh 进程里**。这在四家里最激进，也最适合拿来研究"harness 应该怎么拆"。

### 状态 / 会话

- 会话日志是追加写的事件日志（JSONL），原则是"模型看到的都要能从日志重建"（Model-visible means logged）。
- fork、resume、会话记录、遥测、持久化都从这些已落定的事件派生。
- ⚠️ 架构文档自己写了一个边界：**进程在事件落定之前硬崩溃，不会留下这次尝试的持久记录**。

### 工具与 MCP

- 有 MCP 客户端插件（`dsh-mcp-client`），工具命名形如 `mcp__github__create_issue`。默认不启用任何 server，MCP 的 prompt 模板暂不支持。
- 工具执行有统一管道（见 [tool-execution-pipeline.md](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/tool-execution-pipeline.md)）。

### 多 Agent

- 有 subagent provider 接口：既可以是新的子 Agent，也可以是委派到另一个产品里的一轮对话。
- **Agent Teams** 是实验性、需要手动开启的功能，提供花名册、任务状态和收件箱消息。官网也把"自动授权审查"和"智能体团队"标为实验性。

### HITL / 审批

工具执行前先经过 `tools/pre-execute` 瀑布（hooks、权限、沙箱），再到 `ctx.approval` 一次性审批。**没有审批方，或者审批无法应答时，调用会被拒绝**，也就是默认拒绝。这个默认值我很喜欢。

### 追踪与评测

- 有 OpenTelemetry 相关插件，遥测是基础 bundle 的一部分。
- 内置评测：未核实（仓库里有 BENCHMARK.md，我没细读）。

### 部署

本地桌面端、本地 Web、headless 一次性运行、SDK 子进程、ACP。我没有找到官方的云端托管服务。

官方 [SAFETY.md](https://github.com/deepseek-ai/deepseek-harness/blob/main/SAFETY.md) 的原话要点：

- 这是实验性的开发者预览，**没有经过安全审计**，**不能被视为安全的或可用于生产的**。
- 沙箱、审批提示和权限控制能降低风险，但**不保证隔离**，也防不住项目本来就有权限访问的资源。
- 建议使用一次性虚拟机、容器或专用环境。

### 语言 / 版本 / 许可（2026-10-29 核对）

- 运行层：Node.js。SDK：TypeScript、Python（`deepseek-harness-sdk`，Python ≥3.10）。
- **只有预发布版本**：
  - GitHub 最新：`dsh-v0.2.2-alpha.2`（2026-10-28 UTC），之前是 `dsh-v0.2.1-rc.4`。
  - npm `@deepseek-ai/dsh`：`latest` = 0.2.1-rc.2，`next` = 0.2.1-rc.4，`alpha` = 0.2.2-alpha.2。
  - PyPI `deepseek-harness-sdk`：0.2.1rc2。
- 许可：MIT。仓库创建于 2026-08-13。
- GitHub Star 快照（2026-10-29）：约 264.9k，四家里最多。还在预览版就有这么多 Star，热度可见一斑，但 Star 不等于生产就绪 😅

模型：有官方 DeepSeek 适配器，另有 `llm-pi-ai` 适配器可以路由到多个提供方、OpenAI 兼容网关和自托管服务。

### 最适合的场景

- 研究"一个 Agent harness 应该怎么拆"：循环、工具管道、会话日志、审批、沙箱，每一块都能换掉看看。
- 主力用 DeepSeek 模型，想用官方的运行层做受控试点。
- 想要一个本地桌面 / Web 的 Agent 工具，个人或小团队在隔离环境里用。

### 优点

- 架构最开放：连循环都可替换，适合二次开发和研究。
- 事件日志式的会话设计清晰，fork / resume 有统一来源。
- 默认拒绝的审批语义。
- MIT 许可，模型不锁死。

### 缺点

- 开发者预览，只有预发布版本，官方明确会有破坏性变更。
- 官方明确：未经安全审计，不可视为生产可用。
- 插件化带来的配置面和学习成本不小（profile、bundle、patch 层叠）。
- 生态、文档、社区实践都还在早期。

### 注意事项

- **不要接触生产数据和生产凭据。** 按官方建议，放在一次性虚拟机或容器里。
- 锁死版本，并留意 npm 的 `latest` / `next` / `alpha` 标签指向不同版本。
- 进程硬崩溃时，未落定的那次尝试不会留在日志里，别指望它替你做"恰好一次"。

---

## 六、完整对比表

> 信息截至 2026-10-29。"未核实"表示我没在官方资料里确认，不代表没有。

| 维度 | Google ADK | Claude Agent SDK | OpenAI Agents SDK | DeepSeek Harness |
|---|---|---|---|---|
| 定位 | 全栈 Agent 开发框架 | Claude Code 作为库 | 轻量 Agent 运行时 | 插件化 Agent 运行层 / 应用（带 SDK） |
| 成熟度 | 稳定版（Py 2.x） | 0.x，快速迭代 | 0.x | 开发者预览，仅预发布 |
| 语言 | Python、TS、Go、Java、Kotlin | Python、TS（其他语言可跑 CLI） | Python、TS / JS | Node 运行时；TS、Python SDK |
| 最新版（UTC） | Py 2.13.0（10-23） | Py 0.2.174（10-24）；TS 0.3.312（10-27） | Py 0.24.0（10-15）；JS 0.21.0（10-20） | 0.2.2-alpha.2（10-28） |
| 许可 | Apache-2.0 | 受 Anthropic 商业条款约束 | MIT | MIT |
| 谁控制循环 | 你进程里的 Runner；可用图式 Workflow | CLI 子进程（一会话一进程） | 你进程里的 Runner | 可替换的循环插件，在 dsh 进程中；SDK 用 JSON-RPC 驱动 |
| 状态存哪 | SessionService：内存 / 数据库 / Google Cloud / MongoDB（实验） | 本地 JSONL；SessionStore 镜像（best-effort） | Sessions：SQLite / Redis / SQLAlchemy / MongoDB / Dapr / OpenAI 服务端 | 追加写事件日志（JSONL） |
| 长期记忆 | MemoryService | CLAUDE.md / skills 等文件 | 未作为独立原语核实 | 未核实 |
| 动作前拦截 | 回调 / 插件；Tool Confirmation（Py 2.13 转稳定） | Hooks → deny → ask → 模式 → allow → canUseTool | `needs_approval` + Guardrails | pre-execute 瀑布 + 一次性审批，默认拒绝 |
| 长时间审批 | Workflow HITL 节点；Resumability | 未专门核实 | 序列化 `RunState` 后续跑 | 未专门核实 |
| 恢复 | Resumability（按 invocation ID） | 会话 resume / fork；文件靠 checkpointing | RunState 续跑；Temporal / Dapr / Restate | 从日志 resume / fork |
| 内置工具 | Google 生态工具等 | 文件、shell、网页搜索等（最强） | 函数工具、托管工具、Sandbox agents | 插件提供 |
| MCP | 支持 | 支持 | 支持 | 支持（客户端插件，默认不启用 server） |
| 多 Agent | 层级委派、Task API、A2A（实验） | Subagents | Handoffs、Agents as tools | Subagent provider；Agent Teams（实验） |
| 追踪 | OpenTelemetry；Cloud Trace | OpenTelemetry（traces beta） | 默认开启、上报 OpenAI；可自定义 | OpenTelemetry 插件 |
| 内置评测 | `adk eval` | 未找到 | 对接 OpenAI 评测工具 | 未核实 |
| 模型 | Gemini 优化，模型无关 | 仅 Claude | 默认 OpenAI；其他走适配器（第三方 beta） | DeepSeek 适配器 + 多提供方路由 |
| 托管 / 部署 | 自建或一条命令上 Google Cloud | 自建容器；托管另有 Managed Agents | 自建 | 本地桌面 / Web / headless / SDK |
| 主要绑定点 | Google Cloud（部署、会话、追踪） | Claude 模型 + 商业条款 | OpenAI 模型、追踪后台、Conversations | 预览期 API 稳定性 |
| Star 快照（10-29） | Py 约 22.0k | Py 约 8.4k | Py 约 30.4k | 约 264.9k |

---

## 七、场景决策指南

### 场景 1：Java / Go 团队，已经在 Google Cloud

**→ Google ADK。** 四家里只有它官方覆盖 Java、Go、Kotlin。部署到 Agent Runtime / Cloud Run / GKE 有官方路径，Cloud Trace 和 `adk eval` 也是现成的。
**注意：** 先确认你要用的 2.0 能力在你的语言版本里是否已经有了；会话存储如果要求可迁移，就选数据库实现。

### 场景 2：需要"会干活"的执行体（改代码、处理文件、跑脚本）

**→ Claude Agent SDK。** 文件、shell、搜索这些工具开箱即用，权限体系也细。
**怎么用比较稳：** 把它当成流水线里的一个阶段。上游系统负责任务分发和审批，它在**一次性的隔离工作区**里完成任务，产出物交回上游。会话需要持久化就配 SessionStore 并监控镜像错误。
**注意：** 只能用 Claude 模型；确认商业条款适配你的分发方式。

### 场景 3：Python / TS 业务编排，审批多

**→ OpenAI Agents SDK。** 原语少，`needs_approval` + `RunState` 的暂停 / 续跑很贴合审批流。会话放进自己的 Redis / SQL。
**长流程：** 配 Temporal / Dapr / Restate，让持久执行引擎负责重试和超时。
**注意：** 上线前先把追踪的去向定下来。

### 场景 4：模型无关的企业 Agent 平台

**→ 没有哪一家能直接当"平台"。** 我的思路是：

- 平台层自己掌握：身份、授权、审批、审计日志、幂等键、会话 / 状态存储、评测数据。
- SDK 只当"执行层"，可以并存：编排用 ADK 或 OpenAI Agents SDK，需要干活的阶段调用 Claude Agent SDK，研究或特定模型场景试 DeepSeek Harness。
- 追踪统一走 OpenTelemetry 落到自己的后端（ADK、Claude、DeepSeek Harness 都有 OTel；OpenAI 用自定义 processor）。

这样哪天换模型或换 SDK，平台层的东西都还在。

### 场景 5：研究、试点

**→ DeepSeek Harness**（研究 harness 架构，或者主力是 DeepSeek 模型）。
**边界：** 一次性虚拟机或容器，不接生产数据和生产凭据，锁死版本，接受随时有破坏性变更。

### 场景 6：跨系统、跨天的长事务

**→ 任一 SDK + 企业工作流引擎。** SDK 负责"决定下一步做什么"和"做一步"，工作流负责记账、补偿和重试。**只让一个组件负责重试**，否则 SDK 重试一次、工作流再重试一次，外部系统就会收到重复请求。

---

## 八、企业落地注意事项

1. **锁版本，读发布说明再升级。** 除了 ADK，其他三家都是 0.x 或预发布，Claude TS SDK 几乎隔天一版。ADK 是稳定版，但 v2.13.0 照样有遥测方面的破坏性变更。用 lockfile 锁精确版本，升级前跑一遍自己的回归和评测。
2. **恢复 ≠ 事务。** resume、RunState、Resumability 恢复的是 Agent 的"思路"，不会撤销外部系统里已经发生的写入。DeepSeek Harness 的文档也写明了，硬崩溃前未落定的尝试不会被记录。
3. **幂等要你自己做。** 所有有副作用的工具（写库、发消息、调支付）都要带幂等键，并在你的系统里记账。SDK 不替你做"恰好一次"。
4. **授权在 SDK 外面也要有一道。** 审批钩子是好东西，但它运行在 Agent 进程里。真正的权限边界应该在工具背后的服务上：最小权限的凭据、按任务签发的短期 token、资源级别的访问控制。
5. **审计日志存在自己手里。** 谁在什么时候批准了哪个动作、Agent 实际调用了什么、返回了什么，这些要写进你自己的不可篡改存储，别只依赖 SDK 的会话文件或厂商后台。
6. **注意追踪带来的数据外发。** 追踪里通常有提示词、工具入参和返回值。OpenAI Agents SDK 默认上报 OpenAI 后台；用 Google Cloud 就会进 Cloud Trace。先按数据分级决定追踪去哪，必要时做脱敏或关闭。
7. **隔离执行环境。** Claude Agent SDK 和 DeepSeek Harness 都会真实执行命令、修改文件。用一次性容器或虚拟机，网络出口白名单，不挂载生产凭据。
8. **认真对待"实验性"和"预览"标记。** ADK 的 A2A、MongoDB 会话，Claude 的 traces，OpenAI 的第三方模型适配器，DeepSeek Harness 整体，都带着这类标记。上生产前把它们列成清单，逐项评估。
9. **许可和条款要读。** Apache-2.0、MIT 和"受商业条款约束"不是一回事，尤其是你要把 Agent 能力提供给自己客户的时候。
10. **Star 只是热度。** Star 最多的那个还在开发者预览。选型看的是上面这些问题的答案，不是数字。

---

## 九、我没核实到的东西

为了不瞎写，下面这些我在官方资料里没有确认，所以要么没写，要么在文中标了"未核实"：

- **DeepSeek Harness**
  - 是否有内置评测（仓库里有 BENCHMARK.md，我没细读）
  - 是否有官方云端托管服务（没找到）
  - 是否支持 A2A（没找到）
  - 长时间审批能否序列化后续跑（没专门核实）
  - `deepseek-ai` GitHub 组织没有"已验证"标识；我认定它是官方仓库，依据是 deepseek.com/harness 直链
- **Claude Agent SDK**：没找到内置评测；没专门核实审批能否跨进程挂起很久再续。
- **OpenAI Agents SDK**：没把"长期记忆"作为独立原语核实（Sessions 是会话级的）。
- **Google ADK**：除 Python 外，其他语言版本的 Workflow Runtime、Tool Confirmation 稳定性没有逐一核对。
- **全部四家**：没有任何性能、稳定性、成本方面的实测数据。

---

## 十、说明与免责

- 这是**案头调研**，信息来自各家官方文档、GitHub 仓库、发布记录和包管理器页面，核对日期 2026-10-29。
- 我**没有**在生产环境逐个使用这四家 SDK，**没有**做性能测试。文中的"适合场景""优缺点"是基于文档的判断，欢迎指正。
- 版本号、功能状态变化很快，落地前请按你实际使用的版本重新核对。
- GitHub Star 数是 2026-10-29 的实时快照，只代表热度。
- 本文仅为个人学习分享，不代表任何组织的观点。

如果你在生产里用过其中某一家，踩过什么坑，非常欢迎在 Issues 里告诉我。我也想知道我哪里写错了 🙏

---

## 参考资料

所有链接核对日期：2026-10-29。

**Google ADK**
- 官网与 FAQ：https://adk.dev/
- Sessions：https://adk.dev/sessions/ ，https://adk.dev/sessions/session/
- Memory：https://adk.dev/sessions/memory/
- Models：https://adk.dev/agents/models/
- Tool Confirmation：https://adk.dev/tools-custom/confirmation/
- Resume：https://adk.dev/runtime/resume/
- Observability：https://adk.dev/observability/
- Evaluate：https://adk.dev/evaluate/
- A2A：https://adk.dev/a2a/
- adk-python：https://github.com/google/adk-python ，v2.13.0 发布说明：https://github.com/google/adk-python/releases/tag/v2.13.0
- adk-java：https://github.com/google/adk-java/releases
- adk-go：https://github.com/google/adk-go/releases
- adk-js：https://github.com/google/adk-js/releases
- adk-kotlin：https://github.com/google/adk-kotlin/releases

**Claude Agent SDK**
- Overview：https://code.claude.com/docs/en/agent-sdk/overview
- Permissions：https://code.claude.com/docs/en/agent-sdk/permissions
- Sessions：https://code.claude.com/docs/en/agent-sdk/sessions
- Hosting：https://code.claude.com/docs/en/agent-sdk/hosting
- Observability：https://code.claude.com/docs/en/agent-sdk/observability
- Subagents：https://code.claude.com/docs/en/agent-sdk/subagents
- MCP：https://code.claude.com/docs/en/agent-sdk/mcp
- Hooks：https://code.claude.com/docs/en/agent-sdk/hooks
- Python SDK：https://github.com/anthropics/claude-agent-sdk-python
- TypeScript SDK：https://github.com/anthropics/claude-agent-sdk-typescript ，npm：https://www.npmjs.com/package/@anthropic-ai/claude-agent-sdk

**OpenAI Agents SDK**
- 文档首页：https://openai.github.io/openai-agents-python/
- Running agents：https://openai.github.io/openai-agents-python/running_agents/
- Sessions：https://openai.github.io/openai-agents-python/sessions/
- Human in the loop：https://openai.github.io/openai-agents-python/human_in_the_loop/
- Tracing：https://openai.github.io/openai-agents-python/tracing/
- Models：https://openai.github.io/openai-agents-python/models/
- Python 仓库：https://github.com/openai/openai-agents-python ，v0.24.0：https://github.com/openai/openai-agents-python/releases/tag/v0.24.0
- JS 仓库：https://github.com/openai/openai-agents-js ，npm：https://www.npmjs.com/package/@openai/agents

**DeepSeek Harness**
- 官网页面：https://deepseek.com/harness
- 仓库：https://github.com/deepseek-ai/deepseek-harness
- 文档：https://deepseek-harness.github.io/deepseek-harness/
- SAFETY.md：https://github.com/deepseek-ai/deepseek-harness/blob/main/SAFETY.md
- architecture.md：https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/architecture.md
- tool-execution-pipeline.md：https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/tool-execution-pipeline.md
- Releases：https://github.com/deepseek-ai/deepseek-harness/releases
- npm：https://www.npmjs.com/package/@deepseek-ai/dsh
- PyPI：https://pypi.org/project/deepseek-harness-sdk/
- Cordis：https://github.com/cordiverse/cordis ；论文：https://arxiv.org/abs/2608.25512
