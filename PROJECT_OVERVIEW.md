# DeepSeek Harness（dsh）项目说明文档

> 本文档由 AI 通读仓库（源码、README、docs/、组合配置、门禁脚本）后整理生成，定位是**项目总览 + 维护导航 + 使用手册**，与 `docs/` 下的正式文档互补。
> 适用检出：`deepseek-harness-master`（`@deepseek-ai/dsh-root` v0.1.1-rc.2，developer preview）。
> 维护提示：本文件是**手工维护的导航文档**，而 `docs/` 下的 `tool-catalog.md`、`config-catalog.md`、`module-graph.md`、`persistence-catalog.md`、`apps/cli/composition.md` 等是**由 `scripts/gen-*.ts` 生成的**（`pnpm run gen-*` / `verify-*` 成对门禁），遇到不一致时以生成文档和源码为准。

---

## 1. 项目功能综述

### 1.1 项目定位

DeepSeek Harness（`dsh`）是 DeepSeek 开源的一款**插件化 AI 智能体（Agent）运行框架**。它把"一切皆插件"贯彻到极致：模型适配器、工具注册表、会话日志、沙箱策略、审批流程、甚至 agent 主循环本身都是可插拔插件，全部可通过配置文件替换。

底层框架是 vendored 的 [Cordis](https://github.com/cordiverse/cordis)（见 [cordis-primer](docs/cordis-primer.md)）：插件向共享 Context 贡献**服务（Service）**、**类型化事件（Event）**和**可逆副作用（Effect）**。注册是效应，插件卸载时自动回卷，因此没有"必须打补丁的特权内核"——扩展就是挂一个插件。

### 1.2 核心能力清单

- **多形态运行**：Web GUI（`dsh web`，默认 http://127.0.0.1:3080）、一次性任务（`dsh --profile headless "任务"`）、ACP 自动化服务器、JSON-RPC/SDK 驱动（TypeScript SDK + Python SDK）。
- **完整编码 Agent 能力**：文件读写/编辑/检索（`read/write/edit/read_image/glob/grep/str_replace_editor`）、Shell 执行（bash/pwsh，一次性与持久 PTY 两种）、代码运行（Code Mode，worker 线程/CPython）、LSP 语义导航、网页搜索/抓取（DeepSeek/Exa/Perplexity + HTTP fetch）。
- **编排与委派**：子代理（进程内 spawn/fork、ACP 子进程、Claude Code、Codex、DSH SDK 多后端）、工作流（worker 线程跑模型编写脚本、fan-out 子代理）、Ralph 新鲜代理循环、后台任务（`job_*`）、同会话目标（goal，含 `/goal` 与自动续跑轮次）。
- **人类协作平面**：命令（`/compact`、`/goal`、`/plan`、`/feedback` 等）、审批与权限预设、`ask_user_question` 提问工具、消息级反馈（赞/踩）、计划模式（plan）。
- **数据平面**：事件溯源会话日志（JSONL/SQLite 双后端，Zstd/打包压缩）、会话投影（统计、标题、缓存）、SQLite 全文检索会话查询、跨会话 `@session` 引用、`@file` 引用、附件（内容寻址存储）、溢出存储（spill）、非会话存储中枢、设置/凭据/匿名身份/遥测（OTel）。
- **自指能力（Self-modification）**：agent 可检视并挂载自己的 Cordis 插件运行时（`cordis_*` 工具集 + Host/Client 动态插件 Runner），Web UI 中可实时定义/运行/回滚插件。
- **安全与治理**：进程沙箱（Linux bwrap→Landlock、macOS Seatbelt、Windows ACL 受限令牌）、文件系统观察策略（读前必读、受控写）、工具超时强制、会话检查点、权限预设（`workspace-write` / `danger-full-access` / `read-only`）。
- **可观测与回放**：会话遥测（OTel，默认关闭）、运行时不变量注册表、无 key 快照回放测试体系。

### 1.3 总体架构

#### 1.3.1 组合分层：bundle → profile → patch

一个运行的 `dsh` 是启动时按**有序分层**组合出来的插件树：

1. **bundle（包层）**：可安装的 Cordis 配置行 + 代码的分发格式。`@deepseek-ai/dsh-base`（每个 profile 的第一层：模型适配器、工具、持久化、沙箱/审批、设置/凭据、遥测、核心子代理提供者）、`@deepseek-ai/dsh-web-app`（浏览器面）、`@deepseek-ai/dsh-headless`（一次性任务、无服务器）。
2. **profile（档案层）**：`$DSH_HOME/profiles/<name>/`，含 `package.json`（`dsh.profile.bundles` 有序 bundle 列表 + 外部插件依赖）与 `cordis.patch.yml`（用户自己的 patch 层）。`web`/`headless` 首次使用自动从模板初始化。
3. **patch（补丁层）**：按序叠加——各 bundle 的 patch → profile 的 `cordis.patch.yml` → `$DSH_HOME/cordis.patch.yml`（机器级偏好，优先级更高）→ `--patch <path>` 覆盖。patch 按行 id 定位，**整体替换该行的 `config`**（不做深合并），或 `insert` 新行。

用 `dsh --profile web --dump-config` 可查看本机实际组合的完整插件树（每行注释标注来源文件）。base bundle 的全部插件行见 [apps/cli/composition.md](apps/cli/composition.md)。

#### 1.3.2 Host / Client 双平面

- **Host 平面**：Node.js 进程内，负责文件、网络、命令、Agent/Session、持久化、沙箱/审批、模型路由、HTTP 服务（`packages/host/*`）。
- **Client 平面**：浏览器页面内，负责主题、布局、聊天 UI、工具卡片、设置页（`packages/client/*` 的 `ui-*` 包）。Client 通过私有 JSON 方法（`@Remote` / Typert RPC）调用 Host 业务方法。
- 双平面各自合并 Cordis `Context`，因此 TypeScript 工程用**两个聚合**（`tsconfig.host.json` / `tsconfig.client.json`）隔离编译。

#### 1.3.3 能力接缝（Capability Seam）

可替换能力统一为三件套：**Service Definition**（抽象 `Service` 声明接口，如 `ctx.shell`、`ctx.fs`、`ctx.llm`）、**Service Provider**（实现，如 `dsh-bash-local` vs `dsh-bash-sandbox`）、**Consumer**（使用方，通常是模型工具，如 `dsh-tool-bash`）。换一个 Provider 即整体切换能力——例如把 fs/subprocess 指向远程沙箱，Bash、PTY、LSP 一起迁移。完整图见 [docs/capability-seams.md](docs/capability-seams.md)。

#### 1.3.4 事件体系

事件是扩展点，分三大域：

- **会话事件**（durable）：`turn/*`、`step/*`、`user/message`、`assistant/*`、`tool/*`——追加进会话日志并广播，重载后仍可重建。
- **Agent 事件**（live）：`agent/*`——携带活的 `Agent`（inbox、step、status、request、validation、continuation）。
- **能力事件**（policy）：`fs/*`、`tools/*`、`telemetry/*` 等，把策略/适配器挂到接缝上。

派发模式四种：`emit`（观察）、`waterfall`（环绕中间件，必须 `next()` 委托）、`parallel`（并行）、`serial`（串行）。完整生产者/消费者见 [docs/event-producer-consumer.md](docs/event-producer-consumer.md)。

### 1.4 核心功能流程

#### 1.4.1 回合 / 步骤模型

- **step（步骤）**：一次模型请求 + 该响应触发的工具执行。
- **turn（回合）**：零或多个 step，从认领首个输入开始，到"不再欠任何工作"结束。

```
turn/start
  认领下一步输入 + 一条排队消息
  组装提示词分节 + 工具 schema
  -> agent/pre-step                  拒绝 | enter(messages)
     拒绝，或首个 enter 被改写为空 -> 关闭回合（未消费 step）
     step/start
     将进入的消息追加为 user/message
     从日志派生模型历史
     agent/request -> llm/stream -> assistant/chunk* -> assistant/message
     tool/call* -> tools/pre-execute -> tools/execute -> tools/post-execute -> tool/result*
     step/end
     工具要求再来一次，或新输入到达 -> 认领 -> 下一个 step
  -> agent/turn-stopping
turn/end
```

其中 `agent/pre-step`、`agent/request`、`llm/stream`、`tools/*` 是 waterfall（监听者须调用 `next()`）；`agent/turn-stopping` 是 serial（无 `next()`）。时序图见 [docs/agent-lifecycle.md](docs/agent-lifecycle.md)，工具管线见 [docs/tool-execution-pipeline.md](docs/tool-execution-pipeline.md)。

#### 1.4.2 会话日志机制

会话日志是模型所见上下文的唯一来源：`deriveMessages()` 从日志投影模型历史，原始 `assistant/chunk` 事件保留回放与 UI 保真。**"模型可见 ⟺ 已记录"**是硬性不变量——任何进入模型请求的内容必须能从日志重建，因此新增模型可见输入必须新增会话事件类型（扩展 `SessionEventMap`，见 [docs/subsystems/session.md](docs/subsystems/session.md)）。

#### 1.4.3 端到端时序（Web 模式一次任务）

```
浏览器输入
  → dsh-client-connection (/api fetch/SSE) → dsh-host-apiproxy (Typert RPC 网关)
  → core/session 认领输入（追加 session 事件）
  → core/agent-loop：组装 system-prompt 分节 + tools schema
  → llm 适配器（llm-deepseek / llm-pi-ai）流式返回 → assistant/chunk* 落日志并推回浏览器
  → 模型调用工具 → core/tools 执行管线（pre/execute/post 事件、沙箱/审批策略）
     → 能力提供者（fs/shell/subprocess/web/skill/subagent/jobs/…）
  → 结果回注 → 直到回合结束
  → session-persistence-jsonl/sqlite 落盘、投影/标题/遥测按需更新
```

### 1.5 模块化实现技术栈综述

| 领域 | 技术选型 |
|---|---|
| 语言/工程 | TypeScript（`strict` 全开、ESM-only、Node `^22.19 || >=24`）、pnpm 11.7.0 monorepo（227 个 workspace 包） |
| 框架 | vendored Cordis 4.0.0-rc.7（DI/插件/事件/效应）+ cosmokit、schemastery（配置 schema 校验）、logger-console、loader/include/group/timer/hmr 插件 |
| 类型即契约 | Typert：从 TS 类型图生成 Host/Client 两端 Remote 声明与运行时反射（`@Remote` 方法跨平面调用） |
| 持久化 | JSONL 事件日志（打包 chunk）、SQLite（`node:sqlite`，FTS 全文检索、packed rows + Zstd）、JSON 存储文件、YAML 设置/凭据 |
| 模型接入 | DeepSeek chat-completions（fetch + SSE/eventsource-parser）、pi-ai 多 provider 适配器（OpenAI/Anthropic 等目录 + 兼容开关） |
| Web | 后端 `node:http`（自研路由/升级）+ API 网关；前端 Vite 构建 + React 18（无状态管理框架，对象服务直接驱动）+ `--dsw-*` CSS token 主题 |
| 进程/沙箱 | 子进程接缝（execFile/spawn/PTY）、Linux bwrap → Landlock（native C11 addon，见 `native/`）、macOS Seatbelt、Windows ACL 受限令牌（koffi） |
| 测试 | Vitest 4（单元/覆盖率 perFile 100%、e2e、快照回放、Web/Web-perf/Web-stress 六套配置）、无 key ACP 快照、LLM mock/replay |
| 静态质量 | oxlint + tsgolint（类型感知）、jscpd 克隆检测、knip 未使用检查、publint、NodeNext 消费检查、JSDoc 强制 |
| 门禁/CI | `scripts/run-gates.ts` 依赖图调度 15 个聚合模式；GitHub Actions（static/coverage/consumers/Wine-Windows 阻塞/原生 Windows 观测）+ 发布族工作流 |
| 文档 | VitePress 文档站（中英双语配对 + `.i18n.yaml` 一致性记录）、生成式目录（tool/config/module-graph/persistence）、Agent Notes（RFC 式决策记录） |
| 其它语言 | Python SDK（stdio 换行 JSON-RPC，uv 管理）、原生 Landlock 启动器（C11 + 静态 musl 预编译二进制） |

---

## 2. 目录结构与模块说明

### 2.1 根目录文件一览

| 路径 | 作用 |
|---|---|
| `package.json` | 根包（`@deepseek-ai/dsh-root`）：workspaces、全部脚本门面（build/test/lint/verify/gen/check/demo/dsh） |
| `pnpm-workspace.yaml` | 工作区成员 + `linkWorkspacePackages` + vendor overrides + 构建脚本白名单 |
| `tsconfig.json` / `tsconfig.base.json` / `tsconfig.host.json` / `tsconfig.client.json` / `tsconfig.base.client.json` | 双聚合 Project Reference 工程（详见 2.11） |
| `tsdown.config.ts` | 双面打包（`DSH_BUILD_FACE=host/client`，host 面跑 Typert） |
| `vitest*.ts` | 六套测试配置（单元/覆盖率、e2e、快照、web、web-perf、web-stress）+ 共享 setup |
| `.oxlintrc.json` / `.oxlintrc.staged.json` / `.jscpd.json` / `knip.json` / `lefthook.yml` | 静态质量与本地钩子 |
| `README.md` / `README.zh.md` / `CONTRIBUTING.*` / `LICENSE` / `THIRD_PARTY_NOTICES.md` / `BRAND_GUIDELINES.*` | 对外文档与合规 |
| `AGENTS.md` / `CLAUDE.md` | 给 agent 的仓库工作守则（CLAUDE.md 是符号链接） |
| `.github/` / `.gitlab-ci.yml` | CI（见 2.12） |
| `.agents/` / `.claude/` / `patches/` / `.dsh-build/` | Agent Notes/Skills、Claude 技能链接、pnpm 补丁、构建记录（gitignored） |

### 2.2 packages/：50 个包组、227 个包

分组规则与发布预期见 [packages/README.md](packages/README.md)（下表即其 1:1 映射）。每个包的完整契约（config 字段、Service 名、事件）以各包 `README.md` 与生成文档（[config-catalog](docs/config-catalog.md)、[tool-catalog](docs/tool-catalog.md)）为准；本文给出一行职责便于定位。

#### 2.2.1 核心脊柱（core / api / typert / boot / bundle）

| 包 | 职责 |
|---|---|
| `core/session` | 事件溯源会话日志与内存存储（`ctx.sessions`、`SessionEventMap`、`deriveMessages()`）——**全仓最核心的包**，src 含 `types.ts`（事件类型）、`surface.ts`（表面节点）、`chunk-rows.ts`（chunk 打包）、`json.ts`、`repair.ts`、`preparation.ts` |
| `core/agent` | `Agent` 接口、注册表、initiator scope、`agent/*` 事件词汇（src：`types.ts`、`inbox.ts`、`dispatch.ts`、`consumed-work.ts`、`model-selection.ts`） |
| `core/agent-loop` | **具体 agent 循环驱动**（实现 `Agent` 接口，驱动 session/turn/step 生命周期；src：`agent.ts`、`runtime-context.ts`、`tool-calls.ts`、`constants.ts`） |
| `core/tools` | 工具注册表与受守护执行管线（`ctx.tools`、`ToolDefinition`、schema DSL、pre/execute/post 事件；src：`schema.ts`、`json-schema.ts`、`presentation.ts`、`code-mode.ts`、`py-types.ts`、`ts-types.ts`） |
| `core/system-prompt` | 提示词分节/工具 schema 组装注册表（`ctx.systemPrompt`） |
| `core/scope` | 每 agent 作用域注册原语（`createScope`、作用域过滤派发、shadowing；含 `scoped-events.generated.ts`） |
| `core/agent-default-model` | 部署默认模型选择（`ctx.agentDefaultModel`） |
| `core/agent-tool-presentation` | agent 预设携带的工具呈现形态选择（native/code/both） |
| `api/gateway` | 双端 Typert RPC 端点（Host 提供 `ctx.typertGateway`，Client 消费 `/client`） |
| `api/remotes` | 双端 BFF：Host 侧 Agent/Session 身份策略；Client 侧导入生成的 `/remote` 声明（**唯一 split tsconfig 包**） |
| `typert/generator` | TS 工程分析器 + Typert 生成器（`FaceModel`/`TypeGraph`，tsdown 插件） |
| `typert/loader` | Loader 集成（加载生成产物） |
| `typert/protocol` | 编译器无关声明（`@Remote`/`@RemoteScope` 基类、类型图 schema） |
| `typert/registry` | 运行时注册表（`ctx.typert`，注册各包面的反射与 Zod schema） |
| `boot/app-boot` | app bin 共享启动胶水（profile 组合、`--dump-config`、bundle 解析） |
| `boot/cmdline` | 启动器交给 app 的命令行不可变快照（`ctx.cmdlineArgs`） |
| `bundle/base` | 每个 profile 的第一层：`cordis.patch.yml` 插入全部基础插件行（80+ 行，见 [composition.md](apps/cli/composition.md)） |
| `bundle/headless` | 一次性任务 bundle（headless-startup/headless-runner，无 HTTP/浏览器） |
| `bundle/web-app` | 浏览器面 bundle（Web host 行 + 浏览器插件名册 + agent 预设名册） |

#### 2.2.2 能力接缝组（llm / web / shell / terminal / subprocess / sandbox / fs / lsp / code-runtime / e2b / skill / mcp / spill / attachment / storage / credentials）

| 组 | 包与职责 |
|---|---|
| `llm` | `llm`（接缝 `ctx.llm`，Message/StreamChunk 词汇）、`llm-deepseek`（DeepSeek chat-completions，fetch+SSE）、`llm-pi-ai`（多 provider 目录适配器）、`llm-retry`（agent/request-error 重试）、`token-meter`（可重放 token 计量 `ctx.tokenMeter`） |
| `web` | `web`（接缝 `ctx.web`）、`web-search-deepseek` / `web-search-exa` / `web-search-perplexity`（搜索提供者）、`web-fetch-http`（抓取提供者）、`tool-web`（`web_search`/`web_fetch`） |
| `shell` | `shell`（接缝 `ctx.shell`）、`bash-local` / `bash-sandbox`、`pwsh-local` / `pwsh-sandbox`（四提供者，按平台 `!!js process.platform` 互斥）、`shell-env`（`DSH_*` 可信 shell 变量注册表）、`tool-bash` / `tool-pwsh`（一次性工具）、`tool-bash-persistent` / `tool-pwsh-persistent`（PTY 持久工具） |
| `terminal` | `terminal`（接缝 `ctx.terminals`，owner 隔离 PTY）、`terminal-bash`（bash PTY 后端）、`tool-terminal`（`terminal_open/send/read/signal/close/list`） |
| `subprocess` | `subprocess`（接缝 `ctx.subprocess`，显式 SpawnSpec、偏移输出读取）、`subprocess-local`（本地提供者） |
| `sandbox` | `sandbox`（接缝 `ctx.sandbox`，SandboxMode）、`sandbox-local`（bwrap→Landlock→Seatbelt 选择）、`sandbox-policy`（策略归属 `ctx.sandboxPolicy`）、`sandbox-windows-acl`（Windows ACL 受限令牌） |
| `fs` | `fs`（接缝 `ctx.fs`，12 原语）、`fs-local`、`fs-sandbox`（沙箱强制后端）、`fs-observation-policy`（读前必读/受控写）、`tool-fs`（read/read_image/write/edit）、`tool-fs-search`（glob/grep，打包 ripgrep）、`tool-str-replace-editor` |
| `lsp` | `lsp`（接缝 `ctx.lsp`，4 操作）、`lsp-stdio`（通用 stdio 后端）、`tool-lsp`（`lsp` 工具） |
| `code-runtime` | `code-runtime`（接缝 `ctx.codeRuntime`）、`code-runtime-worker-thread`（worker 线程实现）、`code-runtime-python`（CPython 子进程实现） |
| `e2b` | `e2b`（共享沙箱生命周期）、`fs-e2b` / `subprocess-e2b`（E2B 远程实现）——**POC 级别** |
| `skill` | `skill`（注册表 `ctx.skills`）、`skill-filesystem`（本地提供者）、`skill-badge`（dsh-badge 技能）、`tool-skill`（catalog + `skill` 工具） |
| `mcp` | `mcp-client`（MCP 客户端桥，把外部 MCP 服务器工具注册到 `ctx.tools`；默认不启用任何服务器） |
| `spill` | `spill`（接缝 `ctx.spillStore`）、`spill-local`（本地实现）、`spill-policy`（超大工具结果转存策略） |
| `attachment` | `attachment`（接缝 `ctx.attachments`，规范化图片→`ImageAttachmentRef`）、`attachment-local`（内容寻址存储 `<DSH_HOME>/attachments/v1/objects/<sha256>`） |
| `storage` | `storage`（非会话数据中枢 `ctx.storage`）、`storage-domain`（域表单 `ctx.storageDomain`）、`storage-json` / `storage-sqlite`（后端） |
| `credentials` | `credentials`（接缝 `ctx.credentials`，CredentialRef 永不存值）、`credentials-local`（环境 → `.credentials.yaml` → `.env` 分层）、`authorization`（授权接缝 `ctx.authorization`） |

#### 2.2.3 数据与会话组（session / session-query / settings / identity / workspace / compaction / context / preset / plan / goal / schedule / todo / feedback / guard / hooks）

| 组 | 包与职责 |
|---|---|
| `session` | `session-persistence`（接缝）、`session-persistence-jsonl`（JSONL 后端，打包 chunk ~60% 更小）、`session-persistence-sqlite`（SQLite 后端，packed rows + Zstd）、`session-checkpoint-policy`（语义检查点）、`session-projection`（投影注册表 `ctx.sessionProjections`）、`session-projection-cache`（持久缓存）、`session-stats`（统计投影）、`session-telemetry`（接缝）+ `session-telemetry-otel`（OTLP 后端）、`session-title` + `session-title-llm` + `session-title-first-prompt-llm` + `session-title-all-prompts-llm`（标题） |
| `session-query` | `session-query`（接缝 `ctx.sessionQuery`：精确读取/关系追溯/语义过滤）、`session-query-sqlite`（SQLite FTS 提供者）、`tool-session-query`（`session_search`/`session_event_read`/`session_event_search`/`session_event_trace`/`session_trace`）、`session-log-export`（`/export` 下载） |
| `settings` | `settings`（接缝 `ctx.settings`）、`settings-file`（YAML/JSON 文件 + 热重载） |
| `identity` | `anonymous-user-id`（匿名 ID） |
| `workspace` | `workspace`（工作区注册表 `ctx.workspaceRegistry`） |
| `compaction` | `compaction`（接缝 `CompactionEngine`）、`compaction-basic`（基础后端）、`compaction-tool-result-pruner`（`ctx.toolResultPruner`）、`command-compact`（`/compact`） |
| `context` | `agent-instructions`（AGENTS.md 加载，65,536 字节预算）、`file-reference` + `file-reference-local`（`@file`）、`session-reference`（`@session` 快照）、`time-context`、`tmux-context` |
| `preset` | `agent-presets`（每会话 agent 组合名册，`agent.cordis.yml`）、`persona`（人设行） |
| `plan` | `plan-mode`（计划模式，`/plan`、`exit_plan_mode`） |
| `goal` | `goal`（事件溯源目标状态 `ctx.goals`）、`goal-round-driver`（目标轮驱动）、`command-goal`（`/goal`）、`tool-goal`（`create_goal`/`get_goal`/`update_goal`） |
| `schedule` | `schedule`（会话级定时提醒，`schedule_create/list/delete`） |
| `todo` | `tool-todo`（`todo_write`） |
| `feedback` | `command-feedback`（`/feedback`）、`message-feedback`（消息级反馈 `ctx.messageFeedback`） |
| `guard` | `repeat-tool-reminder`（重复调用提醒）、`timeout-policy`（工具超时强制执行器） |
| `hooks` | `hook-protocol`（Claude Code/Codex 线协议库，非插件）、`hooks-claude-code`、`hooks-codex`（hook 桥） |

#### 2.2.4 编排与人类协作组（subagent / workflow / jobs / interaction / extensions / experimental / runtime-diagnostics）

| 组 | 包与职责 |
|---|---|
| `subagent` | `subagent`（接缝 `ctx.subagents`）、`subagent-in-process-driver` + `subagent-spawn-in-process`（新会话子代理）+ `subagent-fork-in-process`（继承父会话前缀）、`subagent-acp`（ACP 子进程）、`subagent-claude-code` / `subagent-codex` / `subagent-dsh-sdk`（外部产品提供者，可选 bundle）、`tool-subagent`（`subagent`）、`tool-subagent-control`（`send_message`/`interrupt_agent`/`list_agents`）、`tool-subagent-report`（`report`） |
| `workflow` | `workflow`（接缝 `ctx.workflowEngine`）、`workflow-worker-thread`（每运行一个 worker 线程）、`tool-workflow`（`workflow`）、`tool-ralph`（`ralph` 新鲜代理循环） |
| `jobs` | `jobs`（接缝 `ctx.jobs`）、`jobs-local`（进程内实现，默认每 owner 10 并发）、`tool-jobs`（`job_list`/`job_output`/`job_kill`） |
| `interaction` | `commands`（人类命令注册表 `ctx.commands`）、`permission-presets`（权限预设 `ctx.permissionPresets`）、`user-approval`（审批接缝 `ctx.approval`）、`user-questions`（问题接缝 `ctx.userQuestions`）、`tool-ask-user`（`ask_user_question`） |
| `extensions` | `cordis-host-runner`（动态插件 Host 半：vm 沙箱+生命周期）、`cordis-client-runner`（浏览器半）、`tool-cordis`（`cordis_define`/`cordis_run`/`cordis_stop`/`cordis_undefine`/`cordis_inspect_*`）、`ui-cordis`（动态插件面板） |
| `experimental` | `agent-team`（Agent 团队域）、`tool-agent-team`（团队工具）——**不发布** |
| `runtime-diagnostics` | `invariants`（运行时不变量注册表 `ctx.invariants`） |

#### 2.2.5 Web 组（host / client）与工具链组（sdk / acp / util / test-support / examples）

| 组 | 包与职责 |
|---|---|
| `host` | `webserver`（`node:http` 路由服务器 `ctx.webServer`，{host,port}）、`apiproxy`（API 网关 `/api`，TS 契约可浏览器导入）、`frontend-static`（SPA dist 服务器）、`directory-picker`（接缝）+ `-auto`（自适应选择）+ `-browse`（应用内浏览）+ `-native`（原生选择器）、`plugin-inventory`（Loader 树只读投影） |
| `client` | 见下方子表 |
| `sdk` | `protocol`（换行分隔 JSON-RPC 2.0 线协议）、`client`（TS SDK，子进程驱动）、`server`（`jsonrpc` 插件，stdio 服务） |
| `acp` | `acp`（ACP 自动化服务器，JSON-RPC stdio） |
| `util` | `brand`（`Branded<B>`）、`home-paths`（DSH_HOME 路径）、`launch-environment`（分层环境快照）、`atomic-write`、`native-command`（无 shell execFile）、`output-retention`、`timeout`——零依赖工具 |
| `test-support` | `agent-loop-testkit`、`acp-snapshot`（快照套件）、`llm-mock-server`、`llm-replay`、`loader-smoke`、`client-runtime`（jsdom slot 测试运行时） |
| `examples` | `agent-spine-demo`（默认 agent 脊柱 bundle）、`acp-demo`（ACP 演示）、`jsonrpc-demo`（JSON-RPC 演示 bin） |

**client 组（40 个包）**：

- 内核层：`web`（启动内核 `AppWebEntry`）、`modules`（浏览器模块系统，`window.__DSH_BOOT__`）、`runtime`（浏览器 cordis 启动 + 对象服务：SlotRegistry/SessionRuntime）、`connection`（传输：fetch/SSE + `ctx.connection`）、`hmr`（浏览器热重载）、`locale`（zh/en）
- 壳与渲染：`ui-layout`（三栏 AppFrame）、`ui-sidebar`（侧栏）、`ui-renderer`（React 渲染层）、`ui-theme`（`--dsw-*` token）、`ui-slots`（slot 注册表核心）、`ui-primitives`（纯 React 原子组件）、`ui-brand-official`（官方品牌位）
- 会话面：`ui-conversation`（聊天域：流式尾部、作曲器）、`ui-tool`（工具卡片）、`ui-trajectory`（轨迹账本）、`ui-attachment`（附件）、`ui-deliverables`（产出文件）、`ui-workflow-run`（工作流运行节点）、`ui-message-feedback`（赞/踩）、`ui-user-questions`（问题弹窗）
- 输入与引用：`ui-input-trigger`（`/` 与 `@` 触发管线）、`ui-commands`（命令 UI）、`ui-skill`（技能来源）、`ui-reference`（@file/@session）、`ui-subagent`（子代理导航）、`ui-directory-picker-browse` / `ui-directory-picker-native`
- 设置与状态：`ui-settings`（基座）、`ui-settings-general`（设置壳）、`ui-settings-models`（模型页）、`ui-settings-plugins`（插件配置页）、`ui-settings-plugin-inventory`（插件列表）、`ui-permission-presets`（权限）、`ui-model-selection`（模型选择）、`ui-agent-preset`（预设选择）、`ui-jobs`（任务列表）、`ui-goal`（GoalBar）、`ui-plan`（计划状态）、`ui-workspace`（工作区浏览/选择）、`ui-cordis`（动态插件面板）

### 2.3 apps/

| 目录 | 作用 |
|---|---|
| `apps/cli/` | **`dsh` 命令入口**。`src/bin.ts`（只加载所选 runner）、`src/args.ts`（命令语法）、`src/profile-boot.ts`（profile 组合/启动）、`src/dump-config.ts`（`--dump-config`）、`src/plugin.ts`（`dsh plugin` 转发 pnpm）、`src/process-shutdown.ts`（优雅停机）；`reference/README.md`（CLI 行为完整参考）；`config/agent-presets/{standard,code,cordis,minimal}/`（随部署的 agent 预设，`preset.yml` + `agent.cordis.yml`，cordis 预设还带内置 skills）；`composition.md`（生成的 base 组合图） |
| `apps/web/` | Web 前端 Vite 壳：`src/main.ts`（入口，注入 `window.__DSH_BOOT__`）、`public/`、`tests/`（e2e+快照）、`stress-tests/`。**不是独立应用**，必须由 `dsh web` 注入启动环境 |

### 2.4 examples/（可运行组合，非构建目标）

每个示例叶子含真实 `cordis.yml`（由 Loader 直接 boot 的"应用级组合"）+ 无 key 冒烟与有 key e2e：

| 叶子 | 内容 |
|---|---|
| `acp-agent/` | 最完整的应用级组合：ACP 服务器 + 沙箱/审批/持久化 + 子代理/工作流/ralph/todo + 30+ 快照场景（code-mode、pty、image、session-query 等） |
| `headless-agent/` | 一次性编码 agent（stdout 只输出结果），含 settings/credentials 文件提供者 |
| `jsonrpc-agent/` | Python SDK 经 JSON-RPC 驱动；`minimal.cordis.yml` 是**最小可运行组合范例**（llm-deepseek + sandbox + subprocess + terminal + fs-local + agent-spine-demo + 持久化） |
| `web-cordis/` | 自指 agent overlay：`dsh web --patch` 挂载 `cordis-host-runner` + `tool-cordis`，钉端口 3081 |
| `web-schedule/` | 可选的会话级定时提醒 overlay |
| `mcp-memory/` | 可选 MCP 记忆服务器 overlay（engram/memorix/mcp-reference-memory） |

### 2.5 scripts/（164 个文件：构建、门禁与生成器）

- **构建**：`build.ts`（profile → client 构建环境 → build:lib/build:web → 写 `.dsh-build/` 记录）、`clean.ts`（安全清理）、`dev-web.ts`（client 包 HMR 重建 watcher）
- **门禁编排**：`run-gates.ts`——15 个聚合模式（`ci-primary`/`ci-static`/`ci-coverage`/`ci-snapshot`/`ci-artifacts`/`ci-consumers`/`ci-windows-*`/`node-compat`/`check-all`/`hygiene`/`doc-sync`），依赖图校验 + 并发调度
- **生成器（gen-x / verify-x 成对，`--check` 模式断言产物新鲜）**：`gen-cordis-catalog.ts`（docs 各页 Cordis API 区段）、`gen-tool-catalog.ts`（真实启动工具插件收集 schema → docs/tool-catalog.md）、`gen-config-catalog.ts`（→ docs/config-catalog.md）、`gen-persistence-catalog.ts`（→ docs/persistence-catalog.md + known-event-types）、`gen-module-graph.ts`、`gen-doc-graphs.ts`（含 apps/cli/composition.md）、`gen-client-catalog.ts`、`gen-cordis-inspect-catalog.ts`、`gen-scoped-events.ts`、`gen-third-party-notices.ts`
- **校验类**：`verify-cordis-config.ts`（组合配置门禁：`!!js` 元数据限制、示例解析、源码面解析、预设隔离、client 双半一致性）、`verify-runtime-closure.ts`、`verify-package-invariants.ts`、`verify-export-jsdoc.ts`、`verify-type-equiv.ts`、`verify-md-wrap.ts` / `verify-md-links.ts` / `verify-doc-budgets.ts`、`verify-agent-note-*.ts`、`publint-all.ts`、`doc-typecheck.ts`、`rescope-vendor.ts` 等
- **发布**：`release/`（bump/pack/tarball/publish/verify/verify-packed-install）、`publish-npm-baseline.ts`
- **测试基建**：`test-invariants.ts`（vitest 全局 setup）、`coverage-partitions.ts`、`snapshots/`、`fixtures/`

### 2.6 docs/（219 个 md，中英双语配对）

- **架构入口**：`architecture.md`（改 `packages/` 前必读）、`cordis-primer.md`、`capability-seams.md`、`glossary.md`、`agent-lifecycle.md`、`tool-execution-pipeline.md`、`event-producer-consumer.md`、`development.md`、`testing.md`、`defensive-patterns.md`、`api-gateway.md`、`rescope.md`
- **子系统参考**：`subsystems/`（每个子系统一页，含生成式 Cordis API 区段：core/session/tools/system-prompt/llm-streaming/fs/shell/subprocess/sandbox/terminal/lsp/code-runtime/skill/compaction/subagent/workflow/jobs/web/spill/storage/…）
- **生成目录**：`tool-catalog.md`、`config-catalog.md`、`module-graph.md`、`persistence-catalog.md`、`cordis-api/`
- **用户文档**：`user/guide/`（Web UI 使用、模型配置 providers、Python SDK）、`user/develop/`（插件开发：basic/framework/practice）
- **教程**：`cordis-tutorial/`（7 讲）、`cookbook/`（加包/加工具/加 LLM 适配器/加会话节点/加设置卡片/vendor 同步/代码评审）
- **事故事后分析**：`postmortem/`（4 篇）

### 2.7 vendor/（vendored Cordis 源码内置）

9 个包，全部重命名进 `@deepseek-ai` scope、钉住上游 commit，本地改动逐条登记在 `vendor/README.md`：`cordis`（4.0.0-rc.7，含生命周期加固等 18 条本地改动）、`cosmokit`、`schemastery`、`loader`、`include`、`group`、`timer`、`hmr`、`logger-console`。同步流程见 `vendor/README.md`；改 `vendor/*/src` 必须同 commit 更新清单（`scripts/check-vendor-manifest.sh` 强制）。

### 2.8 native/

`landlock-run/`：`@deepseek-ai/node-addon-landlock-run`——约 300 行 C11 静态 musl 的 Landlock 启动器（先自我限制再 exec，ruleset 跨 execve 继承，fail-closed）。`packages/entry`（JS API + 源码）、`packages/linux-x64` / `packages/linux-arm64`（预编译二进制平台包）、`scripts/`（构建/发布/校验）、`docs/`（含跨仓 CLI 契约）。

### 2.9 python/

- `python/sdk/`（`deepseek-harness-sdk`）：高层 turns API + 低层 JSON-RPC 客户端（`src/deepseek_harness/{api,client,models,errors}.py`），uv 管理
- `python/sdk-runtime/`（`deepseek-harness-runtime-bin`）：内置运行时的**部署根**——单 exe 构建的依赖闭包 + 默认 agent 配置（`runtime/cordis.yml`），wheel-only
- 构建：`scripts/build-exe-for-python-sdk.ts`（产物落 `dist-exe/` 并同步进 sdk-runtime）；发布 Linux x64/arm64 + macOS 14+ arm64

### 2.10 .agents/（Agent Notes 与 Skills）

- `notes/`：RFC 式决策记录，路径双轴 `{lifecycle}/{class}/yyyy-mm-dd-topic.md`；lifecycle 为 `proposed/implemented/rejected/archived`，class 为 `feature/bug-fix/simplification/architecture/process/testing`（闭集，门禁强制）；implemented 1120 篇、archived 287 篇、proposed 52 篇
- `skills/`：11 个可复用技能（`dsh-code-review`、`dsh-doc-standards`、`dsh-pre-push-checks`、`dsh-prose-standard`、`dsh-translate-docs` 等，每个一个目录 + `SKILL.md`）
- `.claude/` 是指向 `.agents/skills` 的符号链接，Claude Code 与 DSH 共享技能源

### 2.11 根配置文件（工程化细节）

- **TypeScript 双聚合**：`tsconfig.json` 只是 solution（`files: []` + references，禁止 include/files）；`tsconfig.base.json` 是编译契约 + 解析门面（~240 行 `paths` 指向 `src` 面，无 include 故对 vitest/tsx 全匹配）；`tsconfig.host.json`（Host 聚合：packages/examples/scripts/website + api/remotes 的 Host 面）与 `tsconfig.client.json`（Client 聚合：`packages/client/*` + apps/web + api/remotes 的 Client 面）因 Cordis `Context` 双面合并而必须分离
- **构建**：`tsc -b tsconfig.host.json` → `tsdown --env.DSH_BUILD_FACE host`（含 Typert 生成）→ `tsc -b tsconfig.client.json` → `tsdown --env.DSH_BUILD_FACE client` → `pnpm run build:web`；`build:official` 是 CI/发布等价物
- **测试**：`vitest.config.ts`（单元 + 覆盖率，perFile 100%，豁免清单集中在 GUI/扩展包）、`vitest.e2e.config.ts`（真 API，无 key 自跳过）、`vitest.snapshot.config.ts`（keyless 回放，replay/record/refresh 三模式）、`vitest.web*.ts`
- **本地钩子**（lefthook）：pre-commit（翻译配对校验、Agent Note 校验、oxlint 暂存修复、THIRD_PARTY_NOTICES 再生成、空白检查、vendor manifest 守卫）、pre-push（`pnpm run typecheck`）
- **knip/jscpd/oxlint**：未使用检查、克隆检测、类型感知 lint

### 2.12 CI（.github/ + .gitlab-ci.yml）

- `workflows/ci.yml`：PR 主工作流——node-24 static、coverage、consumers、node-compat 矩阵、python-sdk/runtime、Wine-Windows 阻塞位、原生 Windows 观测位、聚合 `all-checks-passed`
- `ci-master.yml`：master 专用（serial-windows 待机、wine-apt-cache 播种、runner 基准）
- 发布族：`release*.yml`、`release-vendor*.yml`、`python-release.yml`、`build-exe-for-python-sdk.yml`、`landlock-run*.yml`
- 其它：`e2e.yml`（真 API）、`e2b-e2e.yml`、`pi-ai-provider-e2e.yml`、`docs-pages.yml`、`issue-*.yml`
- `.gitlab-ci.yml`：仅 Python 发布（tag `python-v*` 触发）
- `.github/issue-management/`：Issue 生命周期策略 + 可执行测试

---

## 3. 使用手册

### 3.1 环境要求与安装

**要求**：Node.js `^22.19.0 || >=24.0.0`；Corepack 启用的 pnpm 11.7.0；Git 2.26+（钩子用）；可选 DeepSeek API key。

```sh
# 方式一：npm 直接运行（生产）
npx @deepseek-ai/dsh web            # 启动 Web UI（默认 http://127.0.0.1:3080 并打开浏览器）

# 方式二：源码检出
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install
pnpm run build
pnpm dsh web
```

`pnpm run build` 一次性生成产物；之后 `pnpm dsh <args>` 直接跑 TS 入口（无需重建，除非产物过期）。

### 3.2 CLI 命令参考

| 命令 | 用途 |
|---|---|
| `dsh web` | 别名 `--profile web`，启动 Web GUI；后续参数归 web app：`--host`、`--port`、可重复 `--trusted-host`、`--no-open` |
| `dsh --profile headless "任务文本"` | 一次性运行一个持久会话，stdout 打印最终答案，`completed` 退出 0 否则 1 |
| `dsh --profile <name>` | 启动 `$DSH_HOME/profiles/<name>`；web/headless 首用自动初始化，其它 profile 需先 `dsh plugin` 创建 |
| `dsh plugin --profile <name> <pnpm args>` | 在 profile 目录内转发 pnpm（add/remove/update…），成功后按 `dsh.bundle` 声明调和 bundle 列表 |
| `dsh --profile web --dump-config` / `--dump-default-config` | 打印组合后的插件树（含每行来源注释），不启动 |
| `dsh --profile web --patch ./extra.yml` | 追加 patch 覆盖层 |
| `dsh --help` / `dsh -V` | 启动器自身的帮助/版本（出现在 app 参数边界之前） |

要点：

- 启动器只解析自己的 flags（`--profile`/`--patch`/`--dump-*`），**第一个不认识的 token 之后全部交给 app**（`ctx.cmdlineArgs` 不可变快照，可被多个插件解析）。
- 典型示例：`dsh --profile web --port 8080`（--port 属 web app）、`dsh web --no-open`、`dsh web --patch ./extra.cordis.yml`。
- 优雅停机：首个 SIGINT/SIGTERM 给插件树最多 5 秒回卷；SIGINT 退出码 130，SIGTERM 0；第二次信号强制退出。
- `--host 0.0.0.0` 暂不支持（显式 usage error）。

### 3.3 配置体系

#### 3.3.1 数据目录 DSH_HOME

`resolveDshHome()`：显式配置 > `$DSH_HOME` > `~/.dsh`（Windows 上亦为 `%USERPROFILE%\.dsh`）。常用子项：

| 路径 | 内容 |
|---|---|
| `$DSH_HOME/profiles/<name>/` | 每个 profile：`package.json`（`dsh.profile.bundles` + 外部插件依赖）、`cordis.patch.yml`（用户 patch 层）、`node_modules/` |
| `$DSH_HOME/profiles/node_modules/` | 安装级依赖回退（自动愈合的符号链接） |
| `$DSH_HOME/cordis.patch.yml` | 机器级 patch 层（所有 profile 共享，优先级高于 profile 层） |
| `$DSH_HOME/settings.yaml` | 用户设置文档（模型、权限、预设等命名空间；热重载） |
| `$DSH_HOME/.credentials.yaml` | 凭据文档（保存后不再返回明文，只回脱敏描述符；热重载） |
| `$DSH_HOME/attachments/v1/objects/<sha256-prefix>/<sha256>` | 附件内容寻址存储 |
| `$DSH_HOME/storages/` | storage-json 后端数据 |
| `$DSH_HOME/.agent-presets/` | 用户自建 agent 预设（`agent.cordis.yml`；与 shell 同等信任级别） |

#### 3.3.2 组合层与 patch 优先级

生效顺序（后层胜出、整体替换 config、不深合并）：

```
bundle 层（dsh.profile.bundles 顺序：base → web-app/headless → 外部 bundle）
  → profile 的 cordis.patch.yml
  → $DSH_HOME/cordis.patch.yml
  → --patch <path>（argv 顺序）
```

`!!js` 表达式允许出现在 `config` 与 `disabled` 字段（如 `!!js process.platform === 'win32'`、`!!js ctx.webStartup.port ?? 3080`）；其它元数据字段保持字面量。patch 对 `cordis.patch.yml` 的编辑**热重载**生效；bundle 成员变更需重启 profile。

#### 3.3.3 环境变量参考表

| 变量 | 作用 |
|---|---|
| `DEEPSEEK_API_KEY` / `DEEPSEEK_BASE_URL` | DeepSeek 适配器凭据与端点（也支持根目录 `.env`） |
| `DEEPSEEK_SEARCH_BASE_URL` | web_search 的 DeepSeek 搜索端点 |
| `DSH_HOME` | 数据目录根（默认 `~/.dsh`） |
| `DSH_PERMISSION_MODE` | 进程级权限回退（如 `workspace-write` / `danger-full-access` / `read-only`） |
| `DSH_TOOLS_MODE` | 工具呈现形态：`native` / `code` / `both`（Web 与 headless 组合行读取；其它值启动失败） |
| `DSH_TELEMETRY_MODE` | `FULL`（OTLP 全量投影事件）或 `FEEDBACK_ONLY`（仅反馈时上传日志后缀） |
| `DSH_TELEMETRY_OTLP_URL` / `DSH_TELEMETRY_DISABLED` | 遥测收集器 / 权威硬关闭（CI 设 `1`） |
| `DSH_MODEL` / `DSH_CONTEXT_WINDOW` / `DSH_CWD` / `DSH_SYSTEM_PROMPT` | 示例组合中的模型/上下文/工作目录/系统提示覆盖（`!!js process.env...` 读取） |
| `DSH_SESSION_ROOT` / `DSH_SNAPSHOT_SESSIONS_ROOT` | JSONL 持久化根 / 快照会话根 |
| `DSH_SNAPSHOT` | 快照测试模式（`record`/`refresh`/`replay`） |
| `DSH_E2E_MAX_WORKERS` / `DSH_GATE_CONCURRENCY` / `DSH_COVERAGE_*` | 测试/门禁并行度 |
| `DSH_CLIENT_*` / `DSH_CLIENT_BUILD_PROFILE` / `DSH_CLIENT_COMMIT_HASH` | 构建期 client 环境（`build:official` 使用；`ui-brand-official` 仅在 `official` 生效） |
| `SSH_CONNECTION` / `SSH_TTY` | 非空则抑制自动开浏览器（SSH 启动） |
| `NODE_USE_ENV_PROXY=1` | 让支持代理的 Node 版本遵循 HTTP(S)_PROXY |
| `DSH_WEB_URL` / `DSH_WEB_MODE` | web-runtime 发布进 agent shell 环境的变量（`ctx.shellEnv`） |

#### 3.3.4 用户配置：settings.yaml 与 .credentials.yaml

- **模型**：Web UI **Settings → Models** 录入 DeepSeek key → 存 `$DSH_HOME/.credentials.yaml`（写后只读回）；或"Add provider"选目录内 provider（Anthropic/OpenAI 等，pi-ai 目录）；或"Add a custom provider"（Provider ID 永久、base URL、协议、凭据、模型列表）。高级：`llm-pi-ai.providers.<id>` 下配置 `input: [text, image]`（视觉模态）、`compat.supportsDeveloperRole` / `compat.maxTokensField`（OpenAI 兼容网关兼容开关）——直接编辑 `$DSH_HOME/settings.yaml`。
- **settings.yaml 结构**：按命名空间分节（`llm-pi-ai`、`permission`、`locale` 等），外部编辑热发布；文档见 [docs/user/guide/providers.md](docs/user/guide/providers.md) 与生成目录 [config-catalog](docs/config-catalog.md)。
- **凭据解析优先级**（credentials-local）：继承环境 → `$DSH_HOME/.credentials.yaml` → 调用目录 `.env` → `$DSH_HOME/.env`；托管文档不会注入 `process.env`。

#### 3.3.5 权限预设与沙箱

- 新会话默认 `workspace-write` 预设（bash/fs 写操作限制在会话工作区 + 平台临时根；读与网络不限制）；`danger-full-access`（全权 + 审批 `never`）、`read-only` 亦可选。审批策略默认 `ask`（无人应答则 fail-closed），`never` 自动拒绝（CI/无人值守立场）。
- 沙箱后端：Linux 优先 bwrap（私有 PID 命名空间，隐藏宿主进程），失败降级 Landlock（保留进程可见性）；macOS Seatbelt；Windows ACL 受限令牌（`sandbox-local` → `sandbox-windows-acl` 链）。base bundle 按平台互斥挂载 bash/pwsh 栈（win32 上 pwsh-sandbox + tool-pwsh，其余 bash-sandbox + tool-bash）。
- 自定义：`cordis.patch.yml` 覆盖 `permission-presets`/`sandbox-policy`/`user-approval` 行 config（替换整行）。

#### 3.3.6 Agent 预设（preset）

Web 会话创建时可选 agent 预设（`apps/cli/config/agent-presets/`，默认 `standard`）：

| 预设 | 内容 |
|---|---|
| `standard` | 功能完整编码 agent：persona、agent-instructions、tool-bash/pwsh、tool-fs(+search)、tool-str-replace-editor、skill、plan、goal、subagent、jobs、workflow/ralph、todo、web、时间/文件/session 引用等 |
| `code` | Code Mode 形态（DSH_TOOLS_MODE=code 视角的工具集） |
| `cordis` | 自指扩展：额外挂 `tool-cordis` 与 cordis 开发 skills，供运行时检视/挂载自身插件 |
| `minimal` | 极简：固定系统提示 + 仅持久 `bash` + `str_replace_editor` |

用户自定义预设放 `$DSH_HOME/.agent-presets/<id>/`（`preset.yml` 元数据 + `agent.cordis.yml` 组合；服务行须在 `isolate` realm 内）。预设行 id 不得与 base/web-app 已启用行重复（`verify-cordis-config` 强制）。

### 3.4 Web UI 使用流程

1. `npx @deepseek-ai/dsh web`（或源码 `pnpm dsh web`）→ 自动打开 http://127.0.0.1:3080。
2. **Settings → Models** 配置模型（见 3.3.4），保存后立即生效、无需重启。
3. **Choose workspace** 添加并选择项目目录（会话作曲器在未选工作区前不可用）。
4. 新建会话（可选预设），发送任务。agent 可读/改文件、跑命令、委派子代理、维护计划；超出权限策略的操作会弹出审批。
5. 会话记录持久化于 `$DSH_HOME`（JSONL 默认），可 fork/续聊/导出（`/export`）；轨迹（trajectory）、目标（GoalBar）、后台任务、模型切换等均在 UI 中可用。

### 3.5 模块功能调用流程

#### 3.5.1 模型工具（tool）目录

完整 schema 见生成文档 [docs/tool-catalog.md](docs/tool-catalog.md)；下表按实现包归组：

| 实现包 | 工具 |
|---|---|
| `dsh-tool-fs` | `read`、`read_image`、`write`、`edit` |
| `dsh-tool-fs-search` | `glob`、`grep` |
| `dsh-tool-str-replace-editor` | `str_replace_editor` |
| `dsh-tool-bash` / `dsh-tool-bash-persistent` | `bash` |
| `dsh-tool-pwsh` / `dsh-tool-pwsh-persistent` | `pwsh` |
| `dsh-tool-terminal` | `terminal_open`、`terminal_send`、`terminal_read`、`terminal_signal`、`terminal_close`、`terminal_list` |
| `dsh-tool-lsp` | `lsp` |
| `dsh-tools` | `run_code`（Code Mode） |
| `dsh-tool-web` | `web_search`、`web_fetch` |
| `dsh-tool-skill` | `skill` |
| `dsh-tool-subagent` | `subagent` |
| `dsh-tool-subagent-control` | `send_message`、`interrupt_agent`、`list_agents` |
| `dsh-tool-subagent-report` | `report` |
| `dsh-tool-workflow` | `workflow` |
| `dsh-tool-ralph` | `ralph` |
| `dsh-tool-jobs` | `job_kill`、`job_list`、`job_output` |
| `dsh-tool-goal` | `create_goal`、`get_goal`、`update_goal` |
| `dsh-tool-todo` | `todo_write` |
| `dsh-plan-mode` | `exit_plan_mode` |
| `dsh-tool-ask-user` | `ask_user_question` |
| `dsh-tool-session-query` | `session_search`、`session_event_read`、`session_event_search`、`session_event_trace`、`session_trace` |
| `dsh-schedule` | `schedule_create`、`schedule_delete`、`schedule_list` |
| `dsh-tool-cordis` | `cordis_define`、`cordis_inspect_list`、`cordis_inspect_query`、`cordis_inspect_self`、`cordis_run`、`cordis_stop`、`cordis_undefine` |
| `dsh-experimental-tool-agent-team` | `spawn_teammate`、`team_task_*`、`wait_agent`、`followup_task` 等 |

#### 3.5.2 扩展点与调用流程（来自 [docs/architecture.md](docs/architecture.md#where-new-behavior-goes)）

| 目标 | 机制 |
|---|---|
| 加模型提供者 | 在 `ctx.llm` 注册适配器（参考 `llm-deepseek` / `llm-pi-ai`，教程 [adding-an-llm-adapter](docs/cookbook/adding-an-llm-adapter.md)） |
| 加模型能力（工具） | 在 `ctx.tools` 注册；schema 自动加入提示词组装（教程 [adding-a-tool](docs/cookbook/adding-a-tool.md)） |
| 给单个会话不同能力集 | 编写 agent 预设（`agent.cordis.yml`；服务行需 `isolate` realm） |
| 加 shell 执行 | 注册 `ctx.shell` 后端；本地实现经 `ctx.subprocess` spawn |
| 加持久终端 | 注册 `ctx.terminals` 后端 + `dsh-tool-terminal` |
| 加人类命令 | 注册 `ctx.commands`（如 `/compact`、`/goal` 的实现方式） |
| 加后台工作 | 注册 `ctx.jobs`；`job_*` 工具收集/停止 |
| 加文件访问/策略 | 注册 `ctx.fs` 提供者或监听 `fs/*` 事件 |
| 限制派生进程 | 使用 `ctx.sandbox` 后端；消费方 spawn 前包装 argv |
| 拦截请求/工具/回合 | `agent/*` 或 `tools/*` 事件；`agent/turn-stopping` 停止回合 |
| 加模型可见上下文 | `agent.inject()`（下一个被接受的请求落地） |
| 加 UI/编辑器集成 | 驱动 `ctx.agents`、渲染 `session/event` |
| 加 Web 聊天节点 | 注册 `ConversationNodeDefinition` + keyed 渲染器 |
| 加持久会话状态 | 扩展 `SessionEventMap`；从日志渲染与回放 |
| 生成会话标题 | 注册唯一 `ctx.sessionTitle` 提供者 |
| 同会话目标 | 使用 `ctx.goals`，经 `agent/*` 继续 |
| Fork 活会话 | `ctx.sessions.fork(source, boundary?, childSessionId?)` |

新包/工具的完整流程见 [docs/cookbook/adding-a-package.md](docs/cookbook/adding-a-package.md)（含命名规则、README 门禁、Model Experience 格式）。

#### 3.5.3 自动化调用方式

- **Headless**：`dsh --profile headless "任务"`（见 3.2）。
- **JSON-RPC / SDK**：`dsh-sdk-jsonrpc-server` 经 stdio 换行分隔 JSON-RPC 服务 agent；TypeScript SDK（`packages/sdk/client`）与 Python SDK（`python/sdk`，文档 [docs/user/guide/python-sdk.md](docs/user/guide/python-sdk.md)）均可驱动。最小组合范例：`examples/jsonrpc-agent/minimal.cordis.yml`。
- **ACP**：`dsh-acp` 提供自动化专用 Agent Client Protocol 服务器（`pnpm run demo:acp` 从源码跑）。
- **动态 Cordis 插件（自指）**：模型通过 `cordis_define`/`cordis_run` 等在**当前进程**临时扩展运行时（Host 半 + Client 半，需审批）；这不是常规扩展路径，见 [docs/subsystems/extensions.md](docs/subsystems/extensions.md)。

### 3.6 开发与测试命令速查

```sh
pnpm install                 # 安装（含 lefthook 钩子）
pnpm run build               # 构建全部产物（tsc → tsdown 双面 → web）
pnpm run typecheck           # Host+Client 全量类型检查（先生成 Typert 契约）
pnpm run test                # 单元测试
pnpm run test:coverage       # CI 覆盖率门禁（packages 每文件 100%）
pnpm run test:e2e            # 真 API 测试（无 DEEPSEEK_API_KEY 自跳过）
pnpm run test:snapshot       # keyless 快照回放；test:snapshot:record 重新录制（需 key）
pnpm run lint / lint:fix     # oxlint（类型感知）
pnpm run duplication         # jscpd 克隆检测
pnpm run check:all           # 本地全部门禁（依赖图调度）
pnpm run doc-sync            # 文档门禁（生成目录 + 预算 + 链接）
pnpm run hygiene             # knip + publint + 约束 + 许可证等
pnpm dsh --profile headless "任务"   # 源码跑一次任务（需 key）
pnpm run demo:cordis / demo:acp       # 自指 / ACP 演示（需 key）
pnpm run dev:web             # client 包 HMR 重建 watcher（配合 dsh web）
```

### 3.7 示例组合速览

用 `dsh web --patch examples/web-cordis/cordis.yml` 体验自指 agent（端口 3081）；`examples/headless-agent`、`examples/acp-agent`、`examples/jsonrpc-agent` 是三种自动化形态的最小可运行组合；`examples/mcp-memory/*.cordis.yml` 演示 MCP 记忆集成。

---

## 4. 后续优化与升级建议

### 4.1 稳定与兼容（当前为 developer preview，破坏性变更频繁）

- **建立版本化兼容契约**：`session/SESSION_FORMAT_VERSION`、SQLite `SCHEMA_VERSION` 已有单调版本机制，建议为 patch 层、插件 API、事件类型引入同类机制并配迁移工具；1.0 前为高频破坏点（事件类型、`ctx` key、bundle 结构）出一份"迁移手册"。
- **bundle/插件版本锁定**：`dsh plugin` 依赖 pnpm 解析，建议增加"profile 锁文件"与升级演练（当前已有 `release:verify-packed-install`，可推广到 profile 粒度）。
- **Windows 一等公民化**：Windows 用 pwsh 栈 + ACL 沙箱，与 POSIX 的 bash/bwrap/Landlock 行为存在差异（进程可见性、临时目录语义），建议在 [docs/user/guide](docs/user/guide) 增加平台差异对照页，并继续推进 `--host 0.0.0.0` 等能力。

### 4.2 性能

- **会话数据面**：JSONL 已做 chunk 打包（~60% 更小），SQLite 后端有 packed rows + Zstd；建议增加日志分段归档/冷热分层，避免单会话日志无限增长。
- **检索**：Web 默认 `session-query-sqlite` `openAt: never`（内存索引、首搜才开）；建议提供可选的持久 FTS 索引文件与增量刷新，减少启动成本与内存占用。
- **投影与缓存**：`session-projection-cache`（每 200 事件/5s 落盘）已可调；可评估把标题/统计等投影合并进同一写批。
- **流式与上下文**：`spill`/`compaction`/`tool-result-pruner` 已控制上下文膨胀；建议为 `agent-instructions` 预算、工具结果上限提供按会话可调档位，并做一次真实长会话的 token 消耗基线。

### 4.3 安全

- **动态插件沙箱**：`cordis-host-runner` 的 `node:vm` 被文档明示"不是安全边界"（可触达所有注入能力，等同 shell 访问）。建议提供"受限模式"选项：子进程/容器内运行动态插件、按能力最小授权，或至少加审批提示的强度分级。
- **MCP**：默认不启用任何 MCP 服务器是正确立场；建议对 `dsh-mcp-client` 提供每服务器"信任级别 + 工具白名单"配置，并记录服务器命令注入风险。
- **凭据**：CredentialRef 机制（配置只存引用、不存值）很好；建议为 `.credentials.yaml` 增加可选加密（OS keychain 后端），并将 `credentials-local` 的四层解析文档化为可审计的决策点。

### 4.4 工程化与可维护性

- **覆盖率豁免清单**：perFile 100% 门禁的豁免集中在 `client/ui-*`、`extensions/`、`typert/generator`（GUI debt 标注）。建议按季度收缩豁免，优先补 `extensions/*`（动态插件是安全敏感面）与 `typert/generator` 的关键路径。
- **文档体系**：生成目录 + 预算门禁已很完善；建议把本导航文档纳入 `docs/` 的引用（或迁入 `docs/` 并注册预算），并在根 README 加导航链接，避免"导航文档游离于门禁之外"。
- **Agent Notes 规模**：implemented 1120 篇偏多；`dsh-archive-agent-notes` 技能已可归档，建议设定期性（如每季度）归档复盘，保持活跃笔记可检索。
- **CI 成本**：PR 工作流 3 条 Linux 作业 + Wine + 原生 Windows；建议为纯 docs/纯 scripts 变更做 path 过滤，缩短 PR 反馈周期。

### 4.5 生态与体验

- **插件分发**：已有 `dsh-plugin` GitHub topic 与 `dsh plugin add` 流程；建议补充官方插件模板/脚手架（`pnpm create dsh-plugin`）、插件市场页与版本兼容矩阵。
- **UI 体验**：`dsh web` 的 client 插件 HMR 需要额外的 `pnpm run dev:web` watcher 才有热更新；建议把 watcher 并入 `dsh web --dev` 模式，提升插件作者体验。
- **TUI/编辑集成**：CLI 参考中已有 tui profile 的设想（`dsh --profile tui --resume <id>`）；可考虑内置官方 TUI 与 VS Code 扩展（经 JSON-RPC/SDK 已有通道）。
- **更多模型后端**：base 默认仅 DeepSeek 适配器 + pi-ai 目录；建议默认 bundle 里可选启用 Anthropic/OpenAI 目录行（当前需用户自行 patch），并补充 OpenAI 兼容网关的 FAQ（providers.md 已有 compat 开关，可做成 UI 表单字段）。

---

## 附录

### A. 术语速查

seam（能力接缝）、scope（agent 作用域）、turn/step/round（回合/步骤/轮）、goal round（目标轮）、Ralph loop（新鲜代理循环）、patch（补丁层）、profile（档案）、bundle（包层）、preset（agent 预设）、CredentialRef（凭据引用）、spill（溢出存储）、compaction（压缩）。完整术语见 [docs/glossary.md](docs/glossary.md)。

### B. docs/ 导航

- 读架构：`architecture.md` → `subsystems/`（逐子系统类型与 Cordis API）→ 包 README。
- 查配置：`config-catalog.md`（生成，勿手改）；查工具：`tool-catalog.md`（生成）。
- 写扩展：`cordis-tutorial/`（7 讲）→ `cookbook/`（加包/工具/适配器/节点/设置卡）。
- 决策历史：`.agents/notes/`（implemented 为现行事实，archived 冻结）。

### C. 生成式文档与门禁对照

| 文档 | 生成器 | 校验命令 |
|---|---|---|
| `docs/tool-catalog.md` | `gen-tool-catalog.ts` | `pnpm run verify-tool-catalog` |
| `docs/config-catalog.md` | `gen-config-catalog.ts` | `pnpm run verify-config-catalog` |
| `docs/persistence-catalog.md` | `gen-persistence-catalog.ts` | `pnpm run verify-persistence-catalog` |
| `docs/module-graph.md` | `gen-module-graph.ts` | `pnpm run verify-module-graph` |
| `docs/subsystems/*`（Cordis API 区段） | `gen-cordis-catalog.ts` | `pnpm run verify-cordis-catalog` |
| `apps/cli/composition.md` | `gen-doc-graphs.ts` | `pnpm run verify-doc-graphs` |
| `packages/core/scope/src/scoped-events.generated.ts` | `gen-scoped-events.ts` | `pnpm run verify-scoped-events` |
| `THIRD_PARTY_NOTICES.md` | `gen-third-party-notices.ts` | `pnpm run verify-third-party-notices` |

修改对应源码后请运行 `pnpm run gen-*` 重新生成并提交产物；`doc-sync` 会整体校验。
