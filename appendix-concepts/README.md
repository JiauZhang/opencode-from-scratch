# 附录：核心概念速查表

> 本文档提供 Opencode 关键概念的一页速查，涵盖核心抽象、源码位置和关键 API。

---

## 1. Effect-TS 核心概念

| 概念 | 定义 | 核心源码位置 | 关键代码 / API |
|------|------|-------------|----------------|
| **Effect** | 惰性、可组合的异步计算描述，类型为 `Effect<A, E, R>`（成功值 / 错误 / 依赖环境） | `effect` 库（`packages/core/src/effect/`） | `Effect.succeed()` / `Effect.fail()` / `Effect.gen(function* () { ... })` / `Effect.provide(layer)` |
| **Context** | 类型安全的依赖注入容器，通过 `Context.Tag` 标识服务 | `@effect/context` | `Context.Service<Self, Interface>()(tagString)` / `yield* MyService` 获取依赖 |
| **Layer** | 依赖的声明式组合，描述 Service 的构建和依赖关系 | `@effect/layer` | `Layer.effect(Service, effect)` / `Layer.mergeAll(l1, l2)` / `Effect.provide(layer)` |
| **Schema** | 运行时类型验证和编解码，从定义自动衍生 TypeScript 类型 | `@effect/schema` | `Schema.Struct({...})` / `Schema.decodeSync(schema)(data)` / `Schema.encodeEffect(schema)(data)` |
| **Stream** | 惰性、可组合的异步数据流，用于处理 LLM 流式响应等场景 | `@effect/stream` | `Stream.fromIterable()` / `Stream.runForEach(fn)` / `Stream.map().pipe(Stream.filter())` |
| **Fiber** | 轻量级并发单元，类似 Go 的 goroutine，支持结构化并发 | `@effect/fiber` | `Effect.fork(program)` / `Fiber.join(fiber)` / `Fiber.interrupt(fiber)` / `FiberSet.run(fibers, task)` |
| **Scope** | 资源生命周期管理，自动获取和释放资源 | `@effect/scope` | `Effect.scoped` / `Effect.acquireUseRelease(acquire, use, release)` |

---

## 2. Agent 系统

| 概念 | 定义 | 核心源码位置 | 关键代码 / API |
|------|------|-------------|----------------|
| **Agent** | AI 角色定义，包含 name、mode、permissions、model 等属性 | `packages/opencode/src/agent/agent.ts` | `Schema.Struct({ name, mode, permission, model, prompt, ... })` |
| **AgentV2.Info** | Agent 信息的 Schema 定义，所有字段通过 Schema 严格约束 | `packages/opencode/src/agent/agent.ts` | `Schema.Struct({ name: Schema.String, mode: Schema.Literals(["subagent", "primary", "all"]), permission: PermissionV1.Ruleset, ... })` |
| **PermissionV2.Ruleset** | 权限规则集，由 `PermissionV2.Rule` 数组组成，后匹配优先 | `packages/schema/src/permission.ts` | `Schema.Array(Rule)` / `Rule = { action, resource, effect }` / `effect: "allow" \| "deny" \| "ask"` |
| **Agent 模式** | 决定 Agent 的使用场景：primary 主 Agent / subagent 子 Agent / all 两者皆可 | `packages/opencode/src/agent/agent.ts` | `mode: Schema.Literals(["subagent", "primary", "all"])` |

---

## 3. Session Runner

| 概念 | 定义 | 核心源码位置 | 关键代码 / API |
|------|------|-------------|----------------|
| **SessionRunner** | Agent 核心循环的执行引擎，驱动 LLM 调用和工具执行的完整编排 | `packages/core/src/session/runner/` | `run({ sessionID, force })` / `runTurn(sessionID, promotion, step)` 单轮调用 |
| **SessionInput** | 会话输入队列，支持 steer（即时干预）和 queue（排队消息）两种模式 | `packages/core/src/session/` | `SessionInput.hasPending(db, id, "steer")` / `SessionInput.promoteSteers(db, events, id, cutoff)` |
| **SessionStore** | 会话持久化存储，负责会话的 CRUD 和状态管理 | `packages/core/src/session/` | `SessionStore.Service` / `getSession(sessionID)` 加载会话 |
| **SessionContextEpoch** | 系统上下文快照，管理 SystemContext 的代际切换和持久化 | `packages/core/src/system-context/` | `SessionContextEpoch.initialize(db, loadSystemContext(), sessionID)` / `SessionContextEpoch.prepare(...)` |
| **SessionCompaction** | 上下文压缩，当 token 超出模型窗口时自动生成摘要并压缩历史 | `packages/core/src/session/compaction.ts` | `compactIfNeeded({ model, system, messages, tools })` / `compactAfterOverflow(input)` |

---

## 4. 工具系统

| 概念 | 定义 | 核心源码位置 | 关键代码 / API |
|------|------|-------------|----------------|
| **Tool.make** | 创建工具定义，通过 Schema 声明输入/输出类型，自动生成 LLM 可理解的工具描述 | `packages/core/src/tool/tool.ts` | `Tool.make({ description, input: Schema, output: Schema, execute: (input, context) => Effect })` |
| **ToolRegistry** | 工具注册表，管理工具的注册、物化和调度，是工具系统的核心管理中心 | `packages/core/src/tool/registry.ts` | `register(tools)` / `materialize(permissions?)` 返回 `Materialization` |
| **Materialization** | 工具物化，根据 Agent 权限过滤工具，生成 LLM 可理解的定义和可执行函数 | `packages/core/src/tool/registry.ts` | `{ definitions: ToolDefinition[], settle: (input) => Effect<Settlement> }` |
| **BuiltInTools** | 内置工具集合，通过 `BuiltInTools.node` 统一注册 12+ 种核心工具 | `packages/core/src/tool/builtins.ts` | `ReadTool, WriteTool, EditTool, BashTool, GrepTool, GlobTool, QuestionTool, SkillTool, WebFetchTool, WebSearchTool, ...` |
| **Permission** | 工具权限检查，在 materialize 阶段根据 Ruleset 过滤工具 | `packages/core/src/permission.ts` | `evaluate(action, resource, ruleset)` / `whollyDisabled(action, rules)` / `Permission.merge(defaults, agent, user)` |

---

## 5. LLM 集成

| 概念 | 定义 | 核心源码位置 | 关键代码 / API |
|------|------|-------------|----------------|
| **LLMClient** | LLM 调用服务，提供统一的 request / stream / generate / generateObject 接口 | `packages/llm/src/llm.ts` | `LLM.request(input)` / `LLM.stream(request)` / `LLM.generateObject({ model, messages, schema })` |
| **Route** | 路由定义，组合 protocol（协议）+ endpoint（端点）+ auth（认证）+ framing（帧分割）四个维度 | `packages/llm/src/route/client.ts` | `Route.make({ id, protocol, endpoint, auth, framing })` / `route.with(patch)` 不可变更新 |
| **LLMRequest** | 规范化请求，包含 model、system、messages、tools、toolChoice 等字段 | `packages/llm/src/` | `LLM.request({ model, system, messages, tools })` |
| **Provider** | 提供者定义，包含 provider ID 和 model 工厂函数，支持 30+ LLM 提供商 | `packages/llm/src/provider.ts` | `Provider.make({ id, model })` / `Provider.configure(input)` 配置模式 |
| **LLMEvent** | LLM 流式事件，包含 text-delta / reasoning-delta / tool-call / tool-result / finish 等类型 | `packages/llm/src/` | `{ type: "text-delta", delta: string }` / `{ type: "tool-call", id, name, input }` / `{ type: "finish", finish: FinishReason }` |

---

## 6. 插件系统

| 概念 | 定义 | 核心源码位置 | 关键代码 / API |
|------|------|-------------|----------------|
| **Plugin** | 插件定义，包含 id 和初始化逻辑（effect 或 setup 双 API） | `packages/plugin/src/v2/effect/plugin.ts` | `define({ id: string, effect: (context) => Effect<void> })` / Promise API: `define({ id, setup: (context) => Promise<void> })` |
| **PluginContext** | 插件上下文，聚合所有钩子（hooks）和插件域能力 | `packages/plugin/src/v2/effect/context.ts` | `{ options, agent, aisdk, catalog, command, integration, plugin, reference, skill }` |
| **Registration** | 插件注册/注销，每个钩子注册返回 `Registration`，调用 `dispose()` 清理 | `packages/plugin/src/v2/effect/registration.ts` | `{ dispose: Effect<void> }` / `{ reload: () => Effect<void> }` |
| **Hook** | 系统扩展点，通用模式：注册回调 → 触发时调用 → 返回 Registration 用于清理 | `packages/plugin/src/v2/effect/` | `Hooks<Spec>` 映射类型 / `context.agent.transform(callback)` / `context.command.register(callback)` |

---

## 7. MCP 集成

| 概念 | 定义 | 核心源码位置 | 关键代码 / API |
|------|------|-------------|----------------|
| **MCP Client** | MCP 协议客户端，基于 `@modelcontextprotocol/sdk`，管理 MCP 服务器连接 | `packages/opencode/src/mcp/index.ts` | `new Client({ name, version }, { capabilities })` / `client.connect(transport)` / `client.setRequestHandler(ListRootsRequestSchema, handler)` |
| **Transport** | 传输层，支持三种协议：StdioClientTransport（本地进程）、StreamableHTTPClientTransport（HTTP 流）、SSEClientTransport（SSE） | `packages/opencode/src/mcp/index.ts` | `StdioClientTransport({ command, args, cwd })` / `StreamableHTTPClientTransport(url, { authProvider })` / `SSEClientTransport(url, { authProvider })` |
| **Status** | 连接状态管理，通过联合类型精确描述 5 种状态 | `packages/opencode/src/mcp/index.ts` | `Schema.Union([StatusConnected, StatusDisabled, StatusFailed, StatusNeedsAuth, StatusNeedsClientRegistration])` |
| **OAuth** | 认证流程，支持 OAuth 令牌持久化存储和管理 | `packages/opencode/src/mcp/auth.ts` | `McpAuth` / `McpOAuthProvider` / `McpBrowser` 浏览器打开授权 |

---

## 8. 配置系统

| 概念 | 定义 | 核心源码位置 | 关键代码 / API |
|------|------|-------------|----------------|
| **Config.Info** | 配置信息模型，所有字段通过 Effect-TS Schema 定义，均为可选 | `packages/core/src/config.ts` | `Schema.Class<Info>("Config.Info")({ shell, model, default_agent, permissions, agents, mcp, compaction, ... })` |
| **Config 层级** | 配置层级结构：global（`~/.config/opencode/`）→ project（向上查找）→ `.opencode/`（最高优先级） | `packages/core/src/config.ts` | 全局 < 项目文件 < `.opencode` 目录，越具体优先级越高 |
| **Config 加载** | 向上搜索 + 合并策略：从当前目录向上查找 `opencode.jsonc` 和 `.opencode` 目录 | `packages/core/src/config.ts` | `fs.up({ targets, start, stop })` / `latest(entries, key)` 查找最后一个匹配值 |

---

## 9. 事件系统

| 概念 | 定义 | 核心源码位置 | 关键代码 / API |
|------|------|-------------|----------------|
| **EventV2** | 事件服务，基于 Durable Event Sourcing 模式，提供事件发布、订阅、投影和回放 | `packages/core/src/event.ts` | `publish(definition, data)` / `subscribe(definition)` / `project(definition, projector)` / `replay(event)` |
| **Durable Event** | 持久化事件，具有 aggregate（聚合根）和 version（版本号）属性，保证顺序一致性 | `packages/schema/src/event.ts` | `Event.define({ type, durable: { version, aggregate }, schema })` / `{ aggregateID, seq, version }` |
| **Payload** | 事件载荷，包含 id、type、data、durable、location 等字段 | `packages/schema/src/event.ts` | `{ id, type, data, durable?: { aggregateID, seq, version }, location?, metadata? }` |
| **Projector** | 事件投影，在事件持久化时同步执行投影逻辑，更新读模型 | `packages/core/src/event.ts` | `project(definition, (event) => Effect<void>)` |

---

## 10. 系统上下文

| 概念 | 定义 | 核心源码位置 | 关键代码 / API |
|------|------|-------------|----------------|
| **SystemContext** | 系统上下文容器，管理模型可读的上下文信息（系统指令、环境信息、项目信息等） | `packages/core/src/system-context/index.ts` | `make(source)` 创建上下文源 / `combine(values)` 组合多个源 |
| **Source** | 可刷新的类型化上下文源，每个源有独立的 key、codec、load、baseline、update 方法 | `packages/core/src/system-context/index.ts` | `Source<A> = { key, codec, load, baseline, update, removed? }` |
| **Snapshot** | 持久化快照，每个源的值被序列化为 `SourceSnapshot`，所有快照组合成 `Snapshot` | `packages/core/src/system-context/index.ts` | `SourceSnapshot = { value: Json, removed?: string }` / `Snapshot = Record<Key, SourceSnapshot>` |
| **initialize / reconcile / replace** | 生命周期三阶段：首次初始化（加载所有源）→ 增量更新（比较值变化）→ 完全替换（Schema 不兼容时） | `packages/core/src/system-context/index.ts` | `initialize(value)` / `reconcile(generation, value)` / `replace(generation, value)` |

---

## 11. 关键路径

| 路径 | 流程 | 关键源码位置 |
|------|------|-------------|
| **启动流程** | CLI（`packages/opencode/src/index.ts`）→ Bootstrap（`packages/opencode/src/cli/bootstrap.ts`）→ InstanceRuntime.load（`project/instance-runtime.ts`）→ SessionRunner.run（`packages/core/src/session/runner/llm.ts`） | `packages/opencode/src/index.ts` → `cli/bootstrap.ts` → `project/instance-runtime.ts` → `core/src/session/runner/llm.ts` |
| **Agent 循环** | 用户输入 → SystemContext 构建 → LLM 推理调用 → 检测 tool_use → 工具注册表查找 → 权限检查 → 工具执行 → 持久化结果 → 继续循环 | `packages/core/src/session/runner/llm.ts` — `runTurn()` 单轮编排 |
| **工具调用** | LLM 返回 `tool_use` 事件 → ToolRegistry.materialize 查找定义 → Permission 检查 → Schema.decode 解码输入 → 执行工具 → Schema.encode 编码输出 → 返回 tool_result | `packages/core/src/tool/registry.ts` — `materialize()` + `settle()` / `packages/core/src/tool/tool.ts` — `settle()` 执行链 |
| **配置加载** | 全局配置（`~/.config/opencode/`）→ 项目配置文件（从 CWD 向上查找 `opencode.jsonc`）→ `.opencode/` 目录配置 → 合并（通用配置正向叠加，规则配置反向叠加） | `packages/core/src/config.ts` — `layer` + `latest()` 函数 |
| **上下文压缩** | 估算 token → 超过阈值（context - buffer）→ 选择压缩点 → 构建摘要 Prompt → 调用 LLM 生成摘要 → 发布 Compaction 事件 → 从压缩后的历史重建 | `packages/core/src/session/compaction.ts` — `compactIfNeeded()` + `compactAfterOverflow()` |

---

> 本速查表涵盖了全书 13 章的核心概念。如需深入了解某个主题，请返回对应章节阅读完整内容。