# Claude Code Agent Framework Analysis

> 分析版本: 2026-04-09
> 项目路径: `/home/0668001400/AI/code/claude-code_source`

---

## 目录

1. [概述](概述)
2. [Agent 创建流程](#2-agent-创建流程)
3. [Agent 定义数据结构](#3-agent-定义数据结构)
4. [Agent 存储与加载](#4-agent-存储与加载)
5. [Agent 运行机制](#5-agent-运行机制)
6. [Fork 子 Agent 机制](#6-fork-子-agent-机制)
7. [Agent 恢复机制](#7-agent-恢复机制)
8. [内置 Agent 详解](#8-内置-agent-详解)
9. [工具池与权限控制](#9-工具池与权限控制)
10. [MCP 集成](#10-mcp-集成)
11. [关键设计模式](#11-关键设计模式)
12. [可迁移到 LangChain 的模式](#12-可迁移到-langchain-的模式)

---

## 1. 概述

Claude Code 的 Agent 系统分为两个层面：

| 层面 | 目录 | 职责 |
|------|------|------|
| **管理侧** | `components/agents/` | 用户创建、编辑、配置 Agent（UI + 生成逻辑） |
| **执行侧** | `tools/AgentTool/` | Agent 实际运行时的核心引擎 |

**核心文件一览：**

<details>
<summary>📄 代码块</summary>

```
tools/AgentTool/
├── AgentTool.tsx          # 入口：工具定义 + call() 执行
├── runAgent.ts            # 核心：runAgent() 生成器
├── forkSubagent.ts        # Fork 实现
├── resumeAgent.ts         # 恢复已暂停的 Agent
├── prompt.ts              # AgentTool 的提示词设计
├── loadAgentsDir.ts       # 加载 .claude/agents/ 下的 Agent 定义
├── builtInAgents.ts       # 内置 Agent 列表
├── constants.ts           # 常量定义
├── UI.tsx                 # 运行时 UI 渲染
├── agentToolUtils.ts      # 工具函数
├── agentColorManager.ts   # Agent 颜色管理
├── agentMemory.ts         # Agent 记忆
├── built-in/              # 内置 Agent 定义
│   ├── generalPurposeAgent.ts
│   ├── exploreAgent.ts
│   ├── planAgent.ts
│   ├── verificationAgent.ts
│   ├── codeReviewAgent.ts
│   ├── statuslineSetup.ts
│   └── claudeCodeGuideAgent.ts

components/agents/
├── AgentEditor.tsx        # Agent 编辑器
├── AgentsList.tsx         # Agent 列表
├── generateAgent.ts       # AI 生成 Agent 配置
├── validateAgent.ts       # Agent 验证
├── types.ts               # 类型定义
├── utils.ts               # 工具函数
├── agentFileUtils.ts      # 文件操作
└── new-agent-creation/   # 创建向导
```

</details>

---

## 2. Agent 创建流程

### 2.1 创建方式

#### 方式一：向导创建

通过 7 步向导创建：

<details>
<summary>📄 代码块</summary>

```
TypeStep → DescriptionStep → PromptStep → ToolsStep
→ ModelStep → MemoryStep → ColorStep → ConfirmStep
```

</details>

关键文件：`components/agents/new-agent-creation/`

#### 方式二：AI 生成

用户描述需求 → AI 生成 `identifier + whenToUse + systemPrompt` 三件套

**核心流程** (`generateAgent.ts`):

<details>
<summary>📄 typescript 代码块</summary>

```typescript
// 用户输入："我需要一个代码审查agent"
const result = await generateAgent(userPrompt, model, existingIdentifiers, signal)

// 返回结构
{
  identifier: "code-reviewer",           // Agent 类型唯一标识
  whenToUse: "Use this agent when...",  // 何时使用此 Agent
  systemPrompt: "You are..."              // 系统提示词
}
```

</details>

**生成提示词设计** (`AGENT_CREATION_SYSTEM_PROMPT`):

<details>
<summary>📄 typescript 代码块</summary>

```typescript
const AGENT_CREATION_SYSTEM_PROMPT = `You are an elite AI agent architect...

When a user describes what they want an agent to do, you will:

1. **Extract Core Intent**: Identify fundamental purpose and success criteria
2. **Design Expert Persona**: Create compelling expert identity
3. **Architect Comprehensive Instructions**: System prompt that:
   - Establishes clear behavioral boundaries
   - Provides specific methodologies and best practices
   - Anticipates edge cases
   - Defines output format expectations
4. **Optimize for Performance**: Include decision-making frameworks...
5. **Create Identifier**: Concise, descriptive, lowercase with hyphens
6. **Example agent descriptions**: Include whenToUse examples
7. **Agent Memory Instructions**: Optional memory update instructions
```

</details>

### 2.2 创建后的存储

创建完成后，Agent 定义以 Markdown 格式存储在 `.claude/agents/` 目录：

<details>
<summary>📄 markdown 代码块</summary>

```markdown
<!-- .claude/agents/code-reviewer.md -->
---
name: code-reviewer
description: Use this agent when you need to review code for bugs,
             style issues, and security vulnerabilities.
model: sonnet
tools: ['Read', 'Glob', 'Grep', 'Bash']
permissionMode: plan
maxTurns: 50
---

You are an expert code reviewer. Your role is to analyze code
for potential issues and provide constructive feedback.
```

</details>

---

## 3. Agent 定义数据结构

### 3.1 核心类型 (`loadAgentsDir.ts`)

<details>
<summary>📄 typescript 代码块</summary>

```typescript
// 基础类型 - 所有 Agent 的公共字段
type BaseAgentDefinition = {
  agentType: string           // 唯一标识，如 "code-reviewer"
  whenToUse: string           // 描述何时使用此 Agent
  tools?: string[]            // 允许的工具列表
  disallowedTools?: string[]  // 禁止的工具列表
  skills?: string[]          // 预加载的技能列表
  mcpServers?: AgentMcpServerSpec[]  // Agent 专用的 MCP 服务器
  hooks?: HooksSettings       // Session 钩子
  color?: AgentColorName      // UI 颜色
  model?: string              // 模型（sonnet/opus/haiku/inherit）
  effort?: EffortValue        // 努力程度
  permissionMode?: PermissionMode  // 权限模式
  maxTurns?: number           // 最大轮次
  baseDir?: string            // 基础目录
  criticalSystemReminder_EXPERIMENTAL?: string  // 每次用户 turn 注入的提醒
  requiredMcpServers?: string[]  // 必需的 MCP 服务器
  background?: boolean         // 是否总是后台运行
  initialPrompt?: string      // 首个用户 turn 追加的内容
  memory?: AgentMemoryScope   // 持久化记忆范围
  isolation?: 'worktree' | 'remote'  // 隔离模式
  omitClaudeMd?: boolean      // 是否省略 CLAUDE.md
}

// 内置 Agent - 动态提示词
type BuiltInAgentDefinition = BaseAgentDefinition & {
  source: 'built-in'
  baseDir: 'built-in'
  callback?: () => void
  getSystemPrompt: (params: { toolUseContext }) => string
}

// 自定义 Agent - 提示词通过闭包存储
type CustomAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: SettingSource  // 'userSettings' | 'projectSettings' | 'policySettings'
  filename?: string
}

// 插件 Agent
type PluginAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: 'plugin'
  plugin: string
}

// 联合类型
type AgentDefinition =
  | BuiltInAgentDefinition
  | CustomAgentDefinition
  | PluginAgentDefinition
```

</details>

### 3.2 Agent 定义字段详解

| 字段 | 类型 | 说明 |
|------|------|------|
| `agentType` | string | Agent 唯一标识 |
| `whenToUse` | string | Agent 描述，用于给主 Agent 参考何时使用 |
| `tools` | string[] | 允许使用的工具列表，`['*']` 表示全部 |
| `disallowedTools` | string[] | 禁止使用的工具 |
| `model` | string | 指定模型，可选 `sonnet`/`opus`/`haiku`/`inherit` |
| `permissionMode` | string | 权限模式：`acceptEdits`/`plan`/`bubble`/`bypassPermissions` |
| `maxTurns` | number | 最大对话轮次 |
| `getSystemPrompt()` | function | 返回系统提示词的闭包函数 |
| `skills` | string[] | 预加载的技能名称 |
| `mcpServers` | array | Agent 专用的 MCP 服务器配置 |
| `hooks` | object | Session 钩子配置 |
| `memory` | enum | 记忆范围：`user`/`project`/`local` |
| `isolation` | enum | 隔离模式：`worktree`/`remote` |
| `background` | boolean | 是否强制后台运行 |
| `omitClaudeMd` | boolean | 是否从上下文中省略 CLAUDE.md |

---

## 4. Agent 存储与加载

### 4.1 存储位置优先级

<details>
<summary>📄 loadAgentsDir.ts</summary>

```typescript
// loadAgentsDir.ts - getAgentDefinitionsWithOverrides()
// 优先级（后面的覆盖前面的）
const allAgentsList: AgentDefinition[] = [
  ...builtInAgents,     // 1. 内置 Agent
  ...pluginAgents,      // 2. 插件 Agent
  ...customAgents,       // 3. 自定义 Agent（.claude/agents/*.md）
]

// 同名 Agent ，后者覆盖前者
const agentMap = new Map<string, AgentDefinition>()
for (const agents of agentGroups) {
  for (const agent of agents) {
    agentMap.set(agent.agentType, agent)  // 重复 key 覆盖
  }
}
```

</details>

### 4.2 加载流程

<details>
<summary>📄 loadAgentsDir.ts</summary>

```typescript
// loadAgentsDir.ts
async function loadAgentDefinitions(cwd: string): Promise<AgentDefinitionsResult> {
  // 1. 加载 .claude/agents/ 目录下的 Markdown 文件
  const markdownFiles = await loadMarkdownFilesForSubdir('agents', cwd)

  // 2. 解析每个 Markdown 文件
  const customAgents = markdownFiles.map(({ filePath, baseDir, frontmatter, content, source }) => {
    return parseAgentFromMarkdown(filePath, baseDir, frontmatter, content, source)
  })

  // 3. 加载插件 Agent（并发进行）
  const pluginAgents = await loadPluginAgents()

  // 4. 加载内置 Agent
  const builtInAgents = getBuiltInAgents()

  // 5. 合并去重
  return {
    activeAgents: getActiveAgentsFromList(allAgentsList),
    allAgents: allAgentsList,
    failedFiles: parseErrors
  }
}
```

</details>

### 4.3 Markdown 文件解析

<details>
<summary>📄 typescript 代码块</summary>

```typescript
// parseAgentFromMarkdown() - 解析 frontmatter + content
function parseAgentFromMarkdown(filePath, baseDir, frontmatter, content, source) {
  // frontmatter 字段
  const agentType = frontmatter['name']           // 必需
  const whenToUse = frontmatter['description']   // 必需
  const tools = parseAgentToolsFromFrontmatter(frontmatter['tools'])
  const model = frontmatter['model']
  const permissionMode = frontmatter['permissionMode']
  const maxTurns = frontmatter['maxTurns']
  const memory = frontmatter['memory']
  const isolation = frontmatter['isolation']
  const skills = parseSlashCommandToolsFromFrontmatter(frontmatter['skills'])
  const mcpServers = frontmatter['mcpServers']
  const hooks = parseHooksFromFrontmatter(frontmatter['hooks'])

  // content 部分作为 system prompt
  const systemPrompt = content.trim()

  return {
    agentType,
    whenToUse,
    tools,
    getSystemPrompt: () => systemPrompt,  // 闭包
    source,
    // ... 其他字段
  }
}
```

</details>

---

## 5. Agent 运行机制

### 5.1 入口：AgentTool.call()

<details>
<summary>📄 AgentTool.ts</summary>

```typescript
// AgentTool.tsx
async call({
  prompt,           // 任务描述
  subagent_type,    // Agent 类型（可选）
  description,      // 简短描述
  model,            // 模型覆盖
  run_in_background, // 是否后台运行
  name,             // Agent 名称（用于 SendMessage）
  team_name,        // 团队名称
  mode,             // 权限模式
  isolation,        // 隔离模式
  cwd,              // 工作目录
}, toolUseContext, canUseTool, assistantMessage, onProgress) {

  // Step 1: 路由到对应 Agent 定义
  const effectiveType = subagent_type
    ?? (isForkSubagentEnabled() ? undefined : 'general-purpose')
  const isForkPath = effectiveType === undefined

  // Step 2: 找到对应的 AgentDefinition
  const selectedAgent = isForkPath
    ? FORK_AGENT
    : agents.find(a => a.agentType === effectiveType)

  // Step 3: 决定同步/异步运行
  const shouldRunAsync =
    run_in_background === true
    || selectedAgent.background === true
    || isCoordinator
    || forceAsync
    || isForkSubagentEnabled()
    || isProactiveActive()

  // Step 4: 设置工作树隔离
  let worktreeInfo = null
  if (effectiveIsolation === 'worktree') {
    worktreeInfo = await createAgentWorktree(`agent-${agentId.slice(0, 8)}`)
  }

  // Step 5: 构建消息
  let promptMessages: Message[]
  let enhancedSystemPrompt: string[] | undefined
  let forkParentSystemPrompt: SystemPrompt | undefined

  if (isForkPath) {
    // Fork: 继承父 Agent 上下文
    forkParentSystemPrompt = toolUseContext.renderedSystemPrompt
    promptMessages = buildForkedMessages(prompt, assistantMessage)
  } else {
    // 普通 Agent: 构建自己的提示词
    enhancedSystemPrompt = await enhanceSystemPromptWithEnvDetails([
      selectedAgent.getSystemPrompt({ toolUseContext })
    ], resolvedAgentModel, additionalWorkingDirectories)
    promptMessages = [createUserMessage({ content: prompt })]
  }

  // Step 6: 调用 runAgent 执行
  if (shouldRunAsync) {
    return await registerAsyncAgent({ ... })  // 后台运行
  } else {
    for await (const message of runAgent({ ... })) {
      yield message  // 同步运行，实时 yield
    }
  }
}
```

</details>

### 5.2 两种运行模式

#### 同步模式（默认）

<details>
<summary>📄 代码块</summary>

```
主Agent call()
  └── runAgent() [同步]
        ├── query() → LLM API
        └── yield message → 主Agent继续处理
```

</details>

- 主 Agent 等待子 Agent 完成
- 通过 `for await` 实时获取子 Agent 的输出

#### 异步模式（`run_in_background: true`）

<details>
<summary>📄 代码块</summary>

```
主Agent call()
  └── registerAsyncAgent() [立即返回]
        └── runAgent() [后台运行]
              └── 完成后通过 <task-notification> 通知
```

</details>

- 主 Agent 不用等待
- 结果通过 `<task-notification>` 消息异步返回

### 5.3 核心：runAgent() 生成器

<details>
<summary>📄 runAgent.ts</summary>

```typescript
// runAgent.ts - 核心执行引擎
async function* runAgent({
  agentDefinition,
  promptMessages,
  toolUseContext,
  isAsync,
  forkContextMessages,
  model,
  maxTurns,
  availableTools,
  allowedTools,
  useExactTools,
  worktreePath,
  description,
  contentReplacementState,
  onCacheSafeParams,
  onQueryProgress,
}): AsyncGenerator<Message, void> {

  // 1. 解析模型
  const resolvedAgentModel = getAgentModel(
    agentDefinition.model,
    toolUseContext.options.mainLoopModel,
    model,
    permissionMode
  )

  // 2. 创建 Agent ID
  const agentId = override?.agentId ?? createAgentId()

  // 3. 处理消息上下文
  const contextMessages: Message[] = forkContextMessages
    ? filterIncompleteToolCalls(forkContextMessages)
    : []
  const initialMessages: Message[] = [...contextMessages, ...promptMessages]

  // 4. 创建文件状态缓存
  const agentReadFileState = forkContextMessages !== undefined
    ? cloneFileStateCache(toolUseContext.readFileState)
    : createFileStateCacheWithSizeLimit(READ_FILE_STATE_CACHE_SIZE)

  // 5. 解析上下文（用户/系统）
  const [baseUserContext, baseSystemContext] = await Promise.all([
    override?.userContext ?? getUserContext(),
    override?.systemContext ?? getSystemContext(),
  ])

  // 6. 决定是否省略 CLAUDE.md（Explore/Plan 等只读 Agent）
  const shouldOmitClaudeMd = agentDefinition.omitClaudeMd && !override?.userContext

  // 7. 解析工具
  const resolvedTools = useExactTools
    ? availableTools
    : resolveAgentTools(agentDefinition, availableTools, isAsync)

  // 8. 初始化 Agent 专用 MCP 服务器
  const { clients: mergedMcpClients, tools: agentMcpTools, cleanup: mcpCleanup }
    = await initializeAgentMcpServers(agentDefinition, toolUseContext.options.mcpClients)

  // 9. 构建 Agent 选项
  const agentOptions: ToolUseContext['options'] = {
    isNonInteractiveSession: useExactTools
      ? toolUseContext.options.isNonInteractiveSession
      : isAsync ? true : toolUseContext.options.isNonInteractiveSession,
    tools: uniqBy([...resolvedTools, ...agentMcpTools], 'name'),
    mainLoopModel: resolvedAgentModel,
    thinkingConfig: useExactTools
      ? toolUseContext.options.thinkingConfig  // Fork: 继承父的 thinking
      : { type: 'disabled' },  // 普通 Agent: 禁用 thinking
    mcpClients: mergedMcpClients,
    // ... 其他选项
  }

  // 10. 创建子 Agent 上下文
  const agentToolUseContext = createSubagentContext(toolUseContext, {
    options: agentOptions,
    agentId,
    agentType: agentDefinition.agentType,
    messages: initialMessages,
    readFileState: agentReadFileState,
    abortController: agentAbortController,
    getAppState: agentGetAppState,
    shareSetAppState: !isAsync,  // 同步 Agent: 共享 setAppState
  })

  // 11. 预加载技能
  for (const skillName of agentDefinition.skills ?? []) {
    const skill = resolveSkill(skillName)
    const content = await skill.getPromptForCommand('', toolUseContext)
    initialMessages.push(createUserMessage({ content, isMeta: true }))
  }

  // 12. 执行查询循环
  try {
    for await (const message of query({
      messages: initialMessages,
      systemPrompt: agentSystemPrompt,
      userContext: resolvedUserContext,
      systemContext: resolvedSystemContext,
      toolUseContext: agentToolUseContext,
      maxTurns: maxTurns ?? agentDefinition.maxTurns,
    })) {
      // 实时 yield 消息
      yield message
    }
  } finally {
    // 清理
    await mcpCleanup()
    clearSessionHooks()
    killShellTasksForAgent()
    unregisterPerfettoAgent()
  }
}
```

</details>

---

## 6. Fork 子 Agent 机制

### 6.1 Fork vs 普通 Agent 的核心区别

| 特性 | Fork Agent | 普通 Agent |
|------|------------|------------|
| 上下文 | 继承父 Agent 完整上下文 | 从零开始，无历史 |
| 系统提示词 | 使用父 Agent 的（已渲染的） | 使用自己的 `getSystemPrompt()` |
| 工具池 | 与父完全相同 (`useExactTools`) | 根据 `tools`/`disallowedTools` 过滤 |
| 适用场景 | 快速委托工作 | 专门化的独立任务 |
| 提示词类型 | 指令式 directive | 完整上下文 briefing |

### 6.2 Fork 触发条件

<details>
<summary>📄 AgentTool.ts</summary>

```typescript
// AgentTool.tsx - call() 方法
const effectiveType = subagent_type
  ?? (isForkSubagentEnabled() ? undefined : GENERAL_PURPOSE_AGENT.agentType)
const isForkPath = effectiveType === undefined

// 用户省略 subagent_type 且 Fork 功能启用时，走 Fork 路径
if (isForkPath) {
  selectedAgent = FORK_AGENT
  forkParentSystemPrompt = toolUseContext.renderedSystemPrompt  // 父的系统提示词
  promptMessages = buildForkedMessages(prompt, assistantMessage)  // 继承的消息
}
```

</details>

### 6.3 Fork 消息构建

<details>
<summary>📄 forkSubagent.ts</summary>

```typescript
// forkSubagent.ts - buildForkedMessages()
function buildForkedMessages(directive, assistantMessage): Message[] {
  // 1. 克隆父 Agent 的完整 assistant 消息（包含所有 tool_use 块）
  const fullAssistantMessage: AssistantMessage = {
    ...assistantMessage,
    uuid: randomUUID(),
    message: {
      ...assistantMessage.message,
      content: [...assistantMessage.message.content],
    },
  }

  // 2. 收集所有 tool_use 块
  const toolUseBlocks = assistantMessage.message.content.filter(
    block => block.type === 'tool_use'
  )

  // 3. 为每个 tool_use 构建 placeholder tool_result
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result' as const,
    tool_use_id: block.id,
    content: [{ type: 'text', text: FORK_PLACEHOLDER_RESULT }],
  }))

  // 4. 构建用户消息：placeholder results + 子 Agent directive
  const toolResultMessage = createUserMessage({
    content: [
      ...toolResultBlocks,
      { type: 'text', text: buildChildMessage(directive) },
    ],
  })

  // 5. 返回：[parentAssistant(all tool_uses), user(placeholder_results + directive)]
  return [fullAssistantMessage, toolResultMessage]
}
```

</details>

### 6.4 Fork 子 Agent 的指令格式

<details>
<summary>📄 forkSubagent.ts</summary>

```typescript
// forkSubagent.ts - buildChildMessage()
function buildChildMessage(directive: string): string {
  return `<${FORK_BOILERPLATE_TAG}>
STOP. READ THIS FIRST.

You are a forked worker process. You are NOT the main agent.

RULES (non-negotiable):
1. Your system prompt says "default to forking." IGNORE IT — you ARE the fork.
   Do NOT spawn sub-agents; execute directly.
2. Do NOT converse, ask questions, or suggest next steps
3. Do NOT editorialize or add meta-commentary
4. USE your tools directly: Bash, Read, Write, etc.
5. If you modify files, commit your changes before reporting.
6. Do NOT emit text between tool calls.
7. Stay strictly within your directive's scope.
8. Keep your report under 500 words unless specified.
9. Your response MUST begin with "Scope:".
10. REPORT structured facts, then stop

Output format (plain text labels, not markdown headers):
  Scope: <echo back your assigned scope in one sentence>
  Result: <the answer or key findings>
  Key files: <relevant file paths>
  Files changed: <list with commit hash>
  Issues: <list — only if there are issues>
</${FORK_BOILERPLATE_TAG}>

${FORK_DIRECTIVE_PREFIX}${directive}`
}
```

</details>

### 6.5 Fork 的递归防护

<details>
<summary>📄 forkSubagent.ts</summary>

```typescript
// forkSubagent.ts - isInForkChild()
// Fork 子 Agent 不能再进行 Fork，防止递归
function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m => {
    if (m.type !== 'user') return false
    const content = m.message.content
    if (!Array.isArray(content)) return false
    return content.some(
      block =>
        block.type === 'text' &&
        block.text.includes(`<${FORK_BOILERPLATE_TAG}>`)
    )
  })
}

// AgentTool.tsx - call() 中使用
if (isForkPath) {
  if (
    toolUseContext.options.querySource === `agent:builtin:${FORK_AGENT.agentType}`
    || isInForkChild(toolUseContext.messages)
  ) {
    throw new Error('Fork is not available inside a forked worker.')
  }
}
```

</details>

---

## 7. Agent 恢复机制

### 7.1 恢复流程

当用户使用 `SendMessage({ to: agentName })` 继续一个已暂停的 Agent 时：

<details>
<summary>📄 resumeAgent.ts</summary>

```typescript
// resumeAgent.ts - resumeAgentBackground()
async function resumeAgentBackground({
  agentId,
  prompt,
  toolUseContext,
  canUseTool,
  invokingRequestId,
}) {
  // 1. 获取原始 Agent 的 transcript 和 metadata
  const [transcript, meta] = await Promise.all([
    getAgentTranscript(asAgentId(agentId)),
    readAgentMetadata(asAgentId(agentId)),
  ])

  // 2. 过滤消息（移除不完整的 tool_use 等）
  const resumedMessages = filterWhitespaceOnlyAssistantMessages(
    filterOrphanedThinkingOnlyMessages(
      filterUnresolvedToolUses(transcript.messages)
    )
  )

  // 3. 重建 tool result 替换状态
  const resumedReplacementState = reconstructForSubagentResume(
    toolUseContext.contentReplacementState,
    resumedMessages,
    transcript.contentReplacements
  )

  // 4. 确定恢复的 Agent 类型
  let selectedAgent: AgentDefinition
  let isResumedFork = false
  if (meta?.agentType === FORK_AGENT.agentType) {
    selectedAgent = FORK_AGENT
    isResumedFork = true
  } else if (meta?.agentType) {
    selectedAgent = findAgent(meta.agentType) ?? GENERAL_PURPOSE_AGENT
  } else {
    selectedAgent = GENERAL_PURPOSE_AGENT
  }

  // 5. 恢复工作树（如果原来有）
  let resumedWorktreePath
  if (meta?.worktreePath) {
    resumedWorktreePath = await fsp.stat(meta.worktreePath)
      .then(s => s.isDirectory() ? meta.worktreePath : undefined)
      .catch(() => undefined)
  }

  // 6. 构建恢复参数
  const runAgentParams = {
    agentDefinition: selectedAgent,
    promptMessages: [
      ...resumedMessages,
      createUserMessage({ content: prompt }),  // 追加用户新输入
    ],
    toolUseContext,
    isAsync: true,
    override: isResumedFork
      ? { systemPrompt: forkParentSystemPrompt }  // Fork 恢复：重新传入父系统提示词
      : undefined,
    contentReplacementState: resumedReplacementState,
    worktreePath: resumedWorktreePath,
  }

  // 7. 注册并运行
  const agentBackgroundTask = registerAsyncAgent({
    agentId,  // 使用原有 agentId，保持连续性
    description: meta?.description ?? '(resumed)',
    prompt,
    selectedAgent,
    setAppState: rootSetAppState,
  })

  return runWithAgentContext(asyncAgentContext, () =>
    runAsyncAgentLifecycle({
      taskId: agentBackgroundTask.agentId,
      makeStream: onCacheSafeParams => runAgent({
        ...runAgentParams,
        override: {
          ...runAgentParams.override,
          agentId: asAgentId(agentBackgroundTask.agentId),
          abortController: agentBackgroundTask.abortController,
        },
        onCacheSafeParams,
      }),
    })
  )
}
```

</details>

---

## 8. 内置 Agent 详解

### 8.1 General Purpose Agent（通用 Agent）

**何时使用**

| 场景 | 适用理由 |
|------|----------|
| 复杂问题研究 | 需要多步调研、分析多个文件、理解系统架构 |
| 代码搜索 | 在大型代码库中搜索特定代码模式或配置 |
| 多步骤任务执行 | 需要一系列工具调用才能完成的任务 |
| 未知领域探索 | 不确定问题所在，需要边搜索边分析 |

**禁用场景**：简单的一次性查找（直接用 Read/Glob/Grep 更快）、只读研究（用 Explore 更快）。

**英文提示词**

<details>
<summary>📄 built-in/generalPurposeAgent.ts</summary>

```typescript
// built-in/generalPurposeAgent.ts
export const GENERAL_PURPOSE_AGENT: BuiltInAgentDefinition = {
  agentType: 'general-purpose',
  whenToUse: 'General-purpose agent for researching complex questions,
              searching for code, and executing multi-step tasks.',
  tools: ['*'],  // 所有工具
  source: 'built-in',
  baseDir: 'built-in',
  getSystemPrompt: () => `${SHARED_PREFIX}

Your strengths:
- Searching for code, configurations, and patterns
- Analyzing multiple files to understand architecture
- Investigating complex questions
- Performing multi-step research tasks

Guidelines:
- For file searches: search broadly when you don't know where something lives
- For analysis: Start broad and narrow down
- Be thorough: Check multiple locations
- NEVER create files unless absolutely necessary
- NEVER proactively create documentation files`,
}
```

</details>

**中文提示词**

<details>
<summary>📄 代码块</summary>

```
你是一位通用 Agent，负责研究复杂问题、搜索代码和执行多步骤任务。

你的优势：
- 搜索代码、配置和模式
- 分析多个文件以理解系统架构
- 调查复杂问题
- 执行多步骤研究任务

准则：
- 文件搜索：当你不确定文件位置时，广泛搜索
- 分析：从广泛到具体，逐步深入
- 要彻底：检查多个位置
- 除非绝对必要，否则不要创建文件
- 不要主动创建文档文件（*.md）或 README
```

</details>

---

### 8.2 Explore Agent（探索 Agent）

**何时使用**

| 场景 | 适用理由 |
|------|----------|
| 快速文件定位 | "找出所有 `*.tsx` 组件文件" |
| 代码内容搜索 | "找出所有处理用户认证的文件" |
| 代码库问答 | "这个项目的目录结构是怎样的？" |
| 理解现有代码 | "这段代码是做什么的？" |

**特点**：
- **只读模式**：禁止任何文件创建/修改操作
- **快速返回**：使用 haiku 模型（ant 用户继承父模型）
- **无 CLAUDE.md**：不加载项目规范（只读任务不需要）
- **并行搜索**：鼓励同时发起多个搜索工具调用

**禁用场景**：需要修改文件的任务（用 General Purpose Agent）

**英文提示词**

<details>
<summary>📄 built-in/exploreAgent.ts</summary>

```typescript
// built-in/exploreAgent.ts
export const EXPLORE_AGENT: BuiltInAgentDefinition = {
  agentType: 'Explore',
  whenToUse: 'Fast agent specialized for exploring codebases.
              Use for finding files, searching code, or answering
              questions about the codebase.',
  disallowedTools: [
    AGENT_TOOL_NAME,
    EXIT_PLAN_MODE_TOOL_NAME,
    FILE_EDIT_TOOL_NAME,
    FILE_WRITE_TOOL_NAME,
    NOTEBOOK_EDIT_TOOL_NAME,
  ],
  tools: ['*'],  // 所有工具，但 disallowedTools 排除了编辑工具
  source: 'built-in',
  baseDir: 'built-in',
  model: process.env.USER_TYPE === 'ant' ? 'inherit' : 'haiku',
  omitClaudeMd: true,  // 不需要 CLAUDE.md（只读）
  getSystemPrompt: () => `You are a file search specialist...

=== CRITICAL: READ-ONLY MODE ===
STRICTLY PROHIBITED from:
- Creating new files
- Modifying existing files
- Deleting files
- Moving or copying files
- Running ANY commands that change system state

Your role is EXCLUSIVELY to search and analyze existing code.`,
}
```

</details>

**中文提示词**

<details>
<summary>📄 代码块</summary>

```
你是一位文件搜索专家，擅长快速浏览和探索代码库。

=== 关键：只读模式 ===
你被严格禁止：
- 创建新文件
- 修改现有文件
- 删除文件
- 移动或复制文件
- 运行任何改变系统状态的命令

你的角色仅限于搜索和分析现有代码。

你的优势：
- 使用 glob 模式快速定位文件
- 使用正则表达式搜索文件和文本内容
- 阅读和分析文件内容

准则：
- 使用 Glob 工具进行广泛的文件模式匹配
- 使用 Grep 工具搜索文件内容
- 当你知道具体文件路径时使用 Read 工具
- 只使用只读操作（ls、git status、git log、find、grep、cat）
- 禁止使用：mkdir、touch、rm、cp、mv、git add、git commit、npm install 等
- 根据调用者指定的详尽程度调整搜索策略
- 直接报告发现，不要尝试创建任何文件

注意：你是一个快速返回结果的 Agent，为了实现这一点，你必须：
- 高效利用可用的工具：智能搜索文件位置和实现
- 尽可能并行发起多个工具调用来搜索和读取文件
```

</details>

---

### 8.3 Plan Agent（计划 Agent）

**何时使用**

| 场景 | 适用理由 |
|------|----------|
| 新功能规划 | "帮我设计这个功能的实现方案" |
| 重构评估 | "这个模块应该怎么重构？" |
| 技术选型 | "我们应该用 Redux 还是 Context？" |
| 复杂任务分解 | "这个任务太复杂了，怎么分解？" |

**特点**：
- **只读模式**：只探索和设计，不实施
- **继承父模型**：使用与父 Agent 相同的模型
- **输出结构化**：必须包含"关键实现文件"列表
- **无 CLAUDE.md**：不加载项目规范（可自行通过 git/文件读取获取）

**禁用场景**：需要立即执行的任务（用 General Purpose Agent）

**英文提示词**

<details>
<summary>📄 built-in/planAgent.ts</summary>

```typescript
// built-in/planAgent.ts
export const PLAN_AGENT: BuiltInAgentDefinition = {
  agentType: 'Plan',
  whenToUse: 'Software architect agent for designing implementation plans.
              Returns step-by-step plans, identifies critical files,
              and considers architectural trade-offs.',
  disallowedTools: [AGENT_TOOL_NAME, EXIT_PLAN_MODE_TOOL_NAME,
                    FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME],
  source: 'built-in',
  baseDir: 'built-in',
  model: 'inherit',
  omitClaudeMd: true,
  getSystemPrompt: () => `You are a software architect and planning specialist...

=== CRITICAL: READ-ONLY MODE ===
You are EXCLUSIVELY to explore the codebase and design implementation plans.

## Your Process
1. Understand Requirements
2. Explore Thoroughly (find existing patterns, conventions, architecture)
3. Design Solution (trade-offs, architectural decisions)
4. Detail the Plan (step-by-step, dependencies, challenges)

## Required Output
End your response with:
### Critical Files for Implementation
List 3-5 files most critical for implementing this plan`,
}
```

</details>

**中文提示词**

<details>
<summary>📄 代码块</summary>

```
你是一位软件架构师和规划专家，负责探索代码库并设计实现方案。

=== 关键：只读模式 ===
你只能探索代码库并设计实现方案。你不能编写、编辑或修改任何文件。

## 你的流程

1. **理解需求**：专注于提供的需求，并在整个设计过程中应用你被分配的视角

2. **充分探索**：
   - 阅读初始提示中提供的任何文件
   - 使用 Glob、Grep 和 Read 工具查找现有模式和约定
   - 理解当前架构
   - 识别类似的特性作为参考
   - 追踪相关代码路径
   - 仅将 Bash 工具用于只读操作（ls、git status、git log、git diff、find、grep、cat、head、tail）
   - 禁止使用：mkdir、touch、rm、cp、mv、git add、git commit、npm install、pip install 或任何文件创建/修改操作

3. **设计解决方案**：
   - 基于你被分配的视角创实现方案
   - 考虑权衡和架构决策
   - 在适当的地方遵循现有模式

4. **详细计划**：
   - 提供分步实施策略
   - 识别依赖项和顺序
   - 预见潜在挑战

## 必需输出

在你的回复末尾包含：

### 关键实现文件
列出 3-5 个对实现此计划最关键的文件：
- path/to/file1.ts
- path/to/file2.ts
- path/to/file3.ts

记住：你只能探索和规划。你不能编写、编辑或修改任何文件。
```

</details>

---

### 8.4 Verification Agent（验证 Agent）

**何时使用**

| 场景 | 适用理由 |
|------|----------|
| 复杂任务完成后 | 3+ 文件修改、涉及后端/API/基础设施变更 |
| 关键功能实现后 | 确保功能正确工作，不是表面通过 |
| 回归测试 | 检查修改是否破坏了现有功能 |
| 部署前验证 | 确保代码可以构建、测试通过 |

**核心逻辑**：

1. **两个失败模式**（必须克服）：
   - **验证逃避**：面对检查时，找理由不运行它——读代码、说会测试、写"PASS"然后跳过
   - **被前 80% 诱惑**：看到漂亮的 UI 或通过的测试套件就想放过——没注意到一半按钮无效、状态刷新消失、后端崩溃

2. **通用验证步骤**（必须执行）：
   - 读项目 CLAUDE.md / README 获取构建/测试命令
   - 运行构建（失败 = 自动 FAIL）
   - 运行测试套件（失败 = 自动 FAIL）
   - 运行 linter / type-checker
   - 检查相关代码的回归

3. **对抗性探测**（必须尝试）：
   - 并发测试（服务器/API）
   - 边界值测试（0, -1, 空字符串, MAX_INT）
   - 幂等性测试（同一请求发两次）
   - 孤儿操作测试（删除/引用不存在的 ID）

4. **输出格式**（必须遵守）：
<details>
<summary>📄 代码块</summary>

```
   ### Check: [验证内容]
   **Command run:**
     [执行的命令]
   **Output observed:**
     [实际输出——逐字复制粘贴]
   **Result: PASS** / **Result: FAIL** (Expected vs Actual)

   VERDICT: PASS / FAIL / PARTIAL
```

</details>

**英文提示词**

<details>
<summary>📄 built-in/verificationAgent.ts</summary>

```typescript
// built-in/verificationAgent.ts
export const VERIFICATION_AGENT: BuiltInAgentDefinition = {
  agentType: 'verification',
  whenToUse: 'Use this agent to verify that implementation work is correct
              before reporting completion. Pass ORIGINAL task description,
              files changed, and approach taken.',
  color: 'red',
  background: true,  // 强制后台运行
  disallowedTools: [AGENT_TOOL_NAME, EXIT_PLAN_MODE_TOOL_NAME,
                    FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME],
  source: 'built-in',
  baseDir: 'built-in',
  model: 'inherit',
  getSystemPrompt: () => VERIFICATION_SYSTEM_PROMPT,
  criticalSystemReminder_EXPERIMENTAL:
    'CRITICAL: This is a VERIFICATION-ONLY task.
     You CANNOT edit, write, or create files IN THE PROJECT.
     You MUST end with VERDICT: PASS, FAIL, or PARTIAL.',
}
```

</details>

**中文提示词**

<details>
<summary>📄 代码块</summary>

```
你是一位验证专家。你的工作不是确认实现能工作——而是尝试打破它。

你有两个已记录在案的失败模式。第一，验证逃避：当面对检查时，你找理由不去运行它——你读代码、叙述你会测试什么、写"PASS"然后继续。第二，被前 80% 诱惑：当你看到一个精致的 UI 或一个通过的测试套件时，你倾向于通过它，没注意到一半按钮无效、状态刷新后消失、或者后端在错误输入时崩溃。前 80% 是容易的部分。你的全部价值在于找到最后 20%。调用者可能会通过重新运行来抽查你的命令——如果一个 PASS 步骤没有命令输出，或输出与重新执行的结果不匹配，你的报告会被拒绝。

=== 关键：不要修改项目 ===
你被严格禁止：
- 在项目目录中创建、修改或删除任何文件
- 安装依赖或包
- 运行 git 写操作（add、commit、push）

你可以将临时测试脚本写到临时目录（/tmp 或 $TMPDIR）——例如，当内联命令不足时，一个多步竞态测试或 Playwright 测试。用完后清理。

检查你实际可用的工具，而不是从这个提示中假设。你可能有浏览器自动化工具（mcp__claude-in-chrome__*、mcp__playwright__*）、WebFetch 工具或其他 MCP 工具——不要跳过你没有想过要检查的功能。

=== 你会收到什么 ===
你会收到：原始任务描述、变更的文件、采取的方法，以及可选的计划文件路径。

=== 验证策略 ===
根据变更内容调整策略：

**前端变更**：启动开发服务器 → 检查你的工具是否有浏览器自动化（mcp__claude-in-chrome__*、mcp__playwright__*）并使用它们导航、截图、点击和读取控制台 → curl 一系列页面子资源样本 → 运行前端测试

**后端/API 变更**：启动服务器 → curl/fetch 端点 → 验证响应结构是否符合预期（不仅是状态码）→ 测试错误处理 → 检查边界情况

**CLI/脚本变更**：用代表性输入运行 → 验证 stdout/stderr/退出码 → 测试边界输入（空、畸形、边界）→ 验证 --help / 使用说明输出是否准确

**基础设施/配置变更**：验证语法 → 在可能的地方进行试运行（terraform plan、kubectl apply --dry-run=server、docker build、nginx -t）→ 检查环境变量/密钥是否实际被引用，而不仅仅是定义

**库/包变更**：构建 → 完整测试套件 → 从一个全新的上下文导入库并以消费者会使用的方式锻炼公共 API → 验证导出的类型与 README/docs 示例匹配

**Bug 修复**：重现原始 bug → 验证修复 → 运行回归测试 → 检查相关功能的副作用

**移动端（iOS/Android）**：干净构建 → 安装在模拟器/仿真器上 → 转储可访问性/UI 树 → 通过标签查找元素 → 通过树坐标点击 → 重新转储以验证 → 终止并重新启动以测试持久性 → 检查崩溃日志

**数据/ML 管道**：用样本输入运行 → 验证输出结构/模式/类型 → 测试空输入、单行、NaN/null 处理 → 检查隐性数据丢失（输入行数 vs 输出行数）

**数据库迁移**：运行迁移 up → 验证模式是否符合意图 → 运行迁移 down（可逆性）→ 针对现有数据测试，而不仅仅是空数据库

**重构（无行为变更）**：现有测试套件必须不变地通过 → 检查公共 API 表面差异（没有新的/删除的导出）→ 抽查可观察行为是否相同（相同输入 → 相同输出）

**其他变更类型**：模式总是一样的——(a) 找出如何直接执行这个变更（运行/调用/调用/部署它），(b) 检查输出是否符合预期，(c) 尝试用输入/条件打破它，而这些是实现者没有测试过的。上面的策略是工作示例。

=== 必需步骤（通用基线）===
1. 阅读项目的 CLAUDE.md / README 获取构建/测试命令和约定。检查 package.json / Makefile / pyproject.toml 获取脚本名称。如果实现者给你指向了一个计划或规范文件，阅读它——那是成功标准。
2. 运行构建（如果适用）。构建失败 = 自动 FAIL。
3. 运行项目的测试套件（如果有）。测试失败 = 自动 FAIL。
4. 如果配置了 linter/type-checker（eslint、tsc、mypy 等），运行它们。
5. 检查相关代码的回归。

然后应用上述特定类型的策略。根据风险调整严格程度：一次性脚本不需要竞态条件探测；生产支付代码需要全部。

测试套件结果是上下文，不是证据。运行套件，记录通过/失败，然后继续你的真正验证。实现者也是 LLM——它的测试可能在模仿、循环断言或快乐路径覆盖方面很重，这些都不能证明系统实际端到端工作。

=== 识别你自己的合理化 ===
你会感到跳过检查的冲动。这些是你正好会找的借口——认识到它们并做相反的事情：
- "基于我的阅读，代码看起来是正确的"——阅读不是验证。运行它。
- "实现者的测试已经通过了"——实现者是 LLM。独立验证。
- "这可能没问题"——"可能"不是已验证。运行它。
- "让我启动服务器并检查代码"——不。启动服务器并访问端点。
- "我没有浏览器"——你真的检查了 mcp__claude-in-chrome__* / mcp__playwright__* 了吗？如果存在，使用它们。如果 MCP 工具失败了，排除故障（服务器运行中？选择器正确？）。回退存在是为了让你不要编造自己的"无法做到"故事。
- "这会花费太长时间"——这不是你决定的。
如果你发现自己写的是解释而不是命令，停下来。运行命令。

=== 对抗性探测（根据变更类型调整）===
功能测试确认快乐路径。也尝试打破它：
- **并发**（服务器/API）：并行请求创建-if-not-exists 路径——重复会话？丢失写入？
- **边界值**：0、-1、空字符串、非常长的字符串、unicode、MAX_INT
- **幂等性**：两次相同的变更请求——创建了重复？错误？正确的空操作？
- **孤儿操作**：删除/引用不存在的 ID
这些是种子，不是清单——选择适合你正在验证的。

=== 在给出 PASS 之前 ===
你的报告必须包含你运行的一个对抗性探测（并发、边界、幂等性、孤儿操作或类似）及其结果——即使结果是"正确处理"。如果你的所有检查都是"返回 200"或"测试套件通过"，你只确认了快乐路径，而不是验证了正确性。回去尝试打破一些东西。

=== 在给出 FAIL 之前 ===
你发现了一些看起来坏了的东西。在报告 FAIL 之前，检查你是否错过了为什么它实际上是好的：
- **已经处理**：是否有防御性代码（上游验证、下游错误恢复）防止了这个问题？
- **故意的**：CLAUDE.md / 注释 / 提交消息是否说明这是故意的？
- **不可操作**：这是一个真正的限制，但在不破坏外部契约（稳定 API、协议规范、向后兼容）的情况下无法修复？如果是这样，记为观察，而不是 FAIL——一个无法修复的"bug"不是可操作的。
不要用这些作为绕过真正问题的借口——但也不要对故意行为挥手放行。

=== 输出格式（必需）===
每个检查必须遵循这个结构。没有运行的命令的检查不是 PASS——它是跳过。

```

</details>
### Check: [你要验证的内容]
**Command run:**
  [你执行的确切命令]
**Output observed:**
  [观察到的实际终端输出——逐字复制粘贴，不是转述。如果很长但保留相关部分。]
**Result: PASS**（或 FAIL——附上 Expected vs Actual）
<details>
<summary>📄 代码块</summary>

```

不好（被拒绝）：
```

</details>
### Check: POST /api/register 验证
**Result: PASS**
证据：审查了 routes/auth.py 中的路由处理器。逻辑在数据库插入前正确验证了邮箱格式和密码长度。
<details>
<summary>📄 代码块</summary>

```
（没有运行命令。读代码不是验证。）

好：
```

</details>
### Check: POST /api/register 拒绝短密码
**Command run:**
  curl -s -X POST localhost:8000/api/register -H 'Content-Type: application/json' \
    -d '{"email":"t@t.co","password":"short"}' | python3 -m json.tool
**Output observed:**
  {
    "error": "password must be at least 8 characters"
  }
  (HTTP 400)
**Expected vs Actual:** Expected 400 with password-length error. Got exactly that.
**Result: PASS**
<details>
<summary>📄 代码块</summary>

```

以确切的这一行结尾（由调用者解析）：

VERDICT: PASS
或
VERDICT: FAIL
或
VERDICT: PARTIAL

PARTIAL 仅用于环境限制（没有测试框架、工具不可用、服务器无法启动）——不是用于"我不确定这是否是 bug"。如果你能运行检查，你必须决定 PASS 或 FAIL。

使用字面字符串 `VERDICT: ` 后跟恰好一个 `PASS`、`FAIL` 或 `PARTIAL`。不要 markdown 粗体，不要标点符号，不要变化。
- **FAIL**：包括什么失败了、确切的错误输出、重现步骤。
- **PARTIAL**：验证了什么、什么无法验证及原因（缺少工具/环境）、实现者应该知道的什么。
```

</details>

---

### 8.5 内置 Agent 场景对照表

| 任务需求 | 推荐 Agent | 原因 |
|----------|-----------|------|
| 快速查找文件/代码 | **Explore** | 只读、快速、用 haiku 模型 |
| 理解代码架构 | **Explore** | 只读、鼓励并行搜索 |
| 设计实现方案 | **Plan** | 只读、输出结构化计划 |
| 研究复杂问题 | **General Purpose** | 全工具、多步骤 |
| 执行多步骤任务 | **General Purpose** | 全工具、可修改文件 |
| 验证实现正确性 | **Verification** | 对抗性测试、强制后台运行 |
| 不知道用什么 | **Fork** | 继承上下文、快速委托 |

---

## 9. 工具池与权限控制

### 9.1 工具池组装

<details>
<summary>📄 runAgent.ts</summary>

```typescript
// runAgent.ts
const resolvedTools = useExactTools
  ? availableTools  // Fork: 完全继承父的工具池
  : resolveAgentTools(agentDefinition, availableTools, isAsync)

// agentToolUtils.ts - resolveAgentTools()
function resolveAgentTools(agent, availableTools, isAsync) {
  // 1. 如果 agent 定义了 tools 列表，取交集
  if (agent.tools) {
    const allowed = new Set(agent.tools)
    return availableTools.filter(t => allowed.has(t.name))
  }

  // 2. 如果 agent 定义了 disallowedTools，取差集
  if (agent.disallowedTools) {
    const denied = new Set(agent.disallowedTools)
    return availableTools.filter(t => !denied.has(t.name))
  }

  // 3. 否则使用全部可用工具
  return availableTools
}
```

</details>

### 9.2 权限模式

<details>
<summary>📄 typescript 代码块</summary>

```typescript
// 权限模式定义
type PermissionMode =
  | 'acceptEdits'   // 默认：自动接受编辑
  | 'plan'          // 需要确认计划
  | 'bubble'        // 权限提示冒泡到父终端
  | 'bypassPermissions'  // 跳过权限检查

// 运行时权限处理
const workerPermissionContext = {
  ...appState.toolPermissionContext,
  mode: selectedAgent.permissionMode ?? 'acceptEdits'
}

// 对于异步 Agent + bubble 模式
if (shouldAvoidPrompts) {
  toolPermissionContext = {
    ...toolPermissionContext,
    shouldAvoidPermissionPrompts: true,
  }
}
```

</details>

### 9.3 工作树隔离

<details>
<summary>📄 typescript 代码块</summary>

```typescript
// 创建隔离的工作树
if (effectiveIsolation === 'worktree') {
  const slug = `agent-${earlyAgentId.slice(0, 8)}`
  worktreeInfo = await createAgentWorktree(slug)
}

// Fork + worktree：注入路径转换通知
if (isForkPath && worktreeInfo) {
  promptMessages.push(createUserMessage({
    content: buildWorktreeNotice(parentCwd, worktreeCwd)
  }))
}

// worktreeNotice 内容
function buildWorktreeNotice(parentCwd, worktreeCwd): string {
  return `You've inherited context from a parent agent working in ${parentCwd}.
  You are operating in an isolated git worktree at ${worktreeCwd}.
  Paths in the inherited context refer to the parent's working directory;
  translate them to your worktree root.
  Re-read files before editing if the parent may have modified them.
  Your changes stay in this worktree.`
}
```

</details>

---

## 10. MCP 集成

### 10.1 Agent 专用 MCP 服务器

<details>
<summary>📄 runAgent.ts</summary>

```typescript
// runAgent.ts - initializeAgentMcpServers()
async function initializeAgentMcpServers(
  agentDefinition: AgentDefinition,
  parentClients: MCPServerConnection[],
) {
  // 1. 如果没有 agent 专用的 MCP 服务器，返回父的客户端
  if (!agentDefinition.mcpServers?.length) {
    return { clients: parentClients, tools: [], cleanup: async () => {} }
  }

  // 2. 检查插件策略
  const agentIsAdminTrusted = isSourceAdminTrusted(agentDefinition.source)
  if (isRestrictedToPluginOnly('mcp') && !agentIsAdminTrusted) {
    return { clients: parentClients, tools: [], cleanup: async () => {} }
  }

  // 3. 遍历 MCP 服务器配置
  for (const spec of agentDefinition.mcpServers) {
    if (typeof spec === 'string') {
      // 字符串：引用现有配置
      const config = getMcpConfigByName(spec)
      client = await connectToServer(name, config)
    } else {
      // 对象：内联定义（独立的，会清理）
      const [name, config] = Object.entries(spec)[0]
      client = await connectToServer(name, { ...config, scope: 'dynamic' })
      newlyCreatedClients.push(client)
    }

    // 4. 获取 MCP 工具
    if (client.type === 'connected') {
      const tools = await fetchToolsForClient(client)
      agentTools.push(...tools)
    }
  }

  // 5. 返回合并结果
  return {
    clients: [...parentClients, ...agentClients],
    tools: agentTools,
    cleanup: async () => {
      // 只清理新建的客户端，共享的不清理
      for (const client of newlyCreatedClients) {
        await client.cleanup()
      }
    },
  }
}
```

</details>

### 10.2 MCP 服务器配置示例

<details>
<summary>📄 .claude/agents/my-agent.md</summary>

```yaml
# .claude/agents/my-agent.md
---
name: my-mcp-agent
description: An agent with MCP tools
mcpServers:
  - slack          # 引用现有配置
  - my-custom-server:  # 内联定义
      command: node
      args: ["path/to/server.js"]
---

Your system prompt here...
```

</details>

---

## 11. 关键设计模式

### 11.1 生成器模式

`runAgent()` 是 `async generator`，支持同步实时 yield：

<details>
<summary>📄 typescript 代码块</summary>

```typescript
// 主 Agent 同步调用子 Agent
for await (const message of runAgent({ ... })) {
  if (message.type === 'assistant') {
    // 实时处理子 Agent 的输出
  }
}

// 主 Agent 异步调用子 Agent
const task = registerAsyncAgent({ ... })
// 继续其他工作...
// 等待完成通知
```

</details>

### 11.2 上下文隔离

子 Agent 有完全独立的运行时上下文：

<details>
<summary>📄 typescript 代码块</summary>

```typescript
const agentToolUseContext = createSubagentContext(toolUseContext, {
  options: agentOptions,
  agentId,
  messages: initialMessages,
  readFileState: agentReadFileState,  // 独立的文件缓存
  abortController: agentAbortController,  // 独立的 AbortController
  getAppState: agentGetAppState,
  shareSetAppState: !isAsync,  // 同步 Agent 共享，异步隔离
})
```

</details>

### 11.3 记忆机制

<details>
<summary>📄 agentMemory.ts</summary>

```typescript
// agentMemory.ts
type AgentMemoryScope = 'user' | 'project' | 'local'

// 加载记忆提示词
function loadAgentMemoryPrompt(agentType: string, scope: AgentMemoryScope): string {
  // 根据 scope 加载对应的记忆文件内容
  // 注入到 Agent 的系统提示词中
}
```

</details>

### 11.4 清理机制（finally 块）

<details>
<summary>📄 typescript 代码块</summary>

```typescript
try {
  for await (const message of query({ ... })) {
    yield message
  }
} finally {
  // 1. 清理 MCP 服务器
  await mcpCleanup()

  // 2. 清理 session hooks
  clearSessionHooks(rootSetAppState, agentId)

  // 3. 清理 prompt cache tracking
  cleanupAgentTracking(agentId)

  // 4. 释放文件缓存
  agentToolUseContext.readFileState.clear()

  // 5. 注销 perfetto
  unregisterPerfettoAgent(agentId)

  // 6. 清理 todo 列表
  rootSetAppState(prev => {
    const { [agentId]: _, ...todos } = prev.todos
    return { ...prev, todos }
  })

  // 7. 终止后台 bash 任务
  killShellTasksForAgent(agentId, ...)
}
```

</details>

---

## 12. 可迁移到 LangChain 的模式

### 12.1 Agent 定义结构

<details>
<summary>📄 typescript 代码块</summary>

```typescript
// 现有结构
interface AgentDefinition {
  agentType: string
  whenToUse: string
  tools?: string[]
  disallowedTools?: string[]
  model?: string
  permissionMode?: PermissionMode
  maxTurns?: number
  getSystemPrompt(): string
  source: 'built-in' | 'custom' | 'plugin'
}

// LangChain 迁移建议
class AgentDefinition:
    agent_type: str
    description: str  # whenToUse
    tools: List[str]
    disallowed_tools: List[str]
    model: Optional[str]
    permission_mode: PermissionMode
    max_turns: int
    system_prompt_template: str
    source: AgentSource
```

</details>

### 12.2 Fork 机制的借鉴

<details>
<summary>📄 python 代码块</summary>

```python
# LangChain 实现 Fork 机制
class ForkAgent(Agent):
    """
    Fork Agent 继承父 Agent 的完整上下文，
    适用于快速委托工作而不需要完整初始化
    """

    def __init__(self, parent_context: ConversationContext, directive: str):
        self.parent_context = parent_context
        self.directive = directive
        # 继承父的工具池
        self.tools = parent_context.tools

    def build_messages(self) -> List[Message]:
        # 克隆父的 assistant 消息
        parent_assistant = self.parent_context.last_assistant_message
        # 构建 placeholder tool_results
        placeholder_results = [
            ToolResult(tool_use_id=block.id, content="...")
            for block in parent_assistant.tool_uses
        ]
        # 返回合并后的消息
        return [
            parent_assistant,
            UserMessage(content=[
                *placeholder_results,
                TextBlock(text=self.build_directive())
            ])
        ]

    def build_directive(self) -> str:
        return f"""<FORK_BOILERPLATE>
STOP. READ THIS FIRST.

You are a forked worker process...

RULES:
1. Do NOT spawn sub-agents
2. Execute directly
3. Report structured facts, then stop

Output format:
  Scope: <task scope>
  Result: <findings>
</FORK_BOILERPLATE>

{directive}"""
```

</details>

### 12.3 内置 Agent 模式

<details>
<summary>📄 python 代码块</summary>

```python
# LangChain 内置 Agent 定义
class BuiltInAgents:
    GENERAL_PURPOSE = AgentDefinition(
        agent_type="general-purpose",
        description="Research complex questions, search code, execute tasks",
        tools=["*"],
        system_prompt=GENERAL_PURPOSE_PROMPT
    )

    EXPLORE = AgentDefinition(
        agent_type="Explore",
        description="Fast file search and code exploration",
        disallowed_tools=["WriteTool", "EditTool"],
        model="haiku",
        system_prompt=EXPLORE_PROMPT
    )

    PLAN = AgentDefinition(
        agent_type="Plan",
        description="Design implementation plans",
        disallowed_tools=["WriteTool", "EditTool"],
        system_prompt=PLAN_PROMPT
    )

    VERIFICATION = AgentDefinition(
        agent_type="verification",
        description="Verify implementation correctness",
        background=True,
        system_prompt=VERIFICATION_PROMPT
    )
```

</details>

### 12.4 工具池组装

<details>
<summary>📄 python 代码块</summary>

```python
# LangChain 工具池组装
class ToolPool:
    @staticmethod
    def resolve(agent: AgentDefinition, available_tools: List[Tool]) -> List[Tool]:
        if agent.tools:
            # 白名单模式
            allowed = set(agent.tools)
            return [t for t in available_tools if t.name in allowed]
        elif agent.disallowed_tools:
            # 黑名单模式
            denied = set(agent.disallowed_tools)
            return [t for t in available_tools if t.name not in denied]
        else:
            return available_tools
```

</details>

### 12.5 MCP 集成

<details>
<summary>📄 python 代码块</summary>

```python
# LangChain MCP 集成
class MCPServerPool:
    def __init__(self):
        self.clients: Dict[str, MCPClient] = {}

    def add_agent_servers(self, agent_def: AgentDefinition,
                          parent_clients: List[MCPClient]) -> List[MCPClient]:
        """Agent 可以定义自己的 MCP 服务器"""
        if not agent_def.mcp_servers:
            return parent_clients

        merged = [*parent_clients]
        for spec in agent_def.mcp_servers:
            if isinstance(spec, str):
                # 引用现有服务器
                client = self.get_or_connect(spec)
            else:
                # 内联定义
                name, config = next(iter(spec.items()))
                client = self.connect(name, config, scope="dynamic")
            merged.append(client)

        return merged
```

</details>

### 12.6 记忆机制

<details>
<summary>📄 python 代码块</summary>

```python
# LangChain Agent 记忆
class AgentMemory(ABC):
    @abstractmethod
    def load_prompt(self, agent_type: str, scope: MemoryScope) -> str:
        """加载记忆并返回提示词片段"""
        pass

class FileBasedAgentMemory(AgentMemory):
    def __init__(self, memory_dir: Path):
        self.memory_dir = memory_dir

    def load_prompt(self, agent_type: str, scope: MemoryScope) -> str:
        # 根据 scope 加载对应的记忆文件
        memory_file = self.memory_dir / scope.value / f"{agent_type}.md"
        if memory_file.exists():
            return f"\n\n## Agent Memory\n{memory_file.read_text()}"
        return ""
```

</details>

### 12.7 恢复机制

<details>
<summary>📄 python 代码块</summary>

```python
# LangChain Agent 恢复
class AgentResumer:
    def resume(self, agent_id: str, new_prompt: str,
               transcript: List[Message]) -> AgentResult:
        """恢复已暂停的 Agent"""

        # 1. 过滤消息（移除不完整的 tool_use）
        cleaned_messages = self.clean_transcript(transcript)

        # 2. 重建上下文状态
        replacement_state = self.reconstruct_tool_results(
            cleaned_messages,
            transcript.content_replacements
        )

        # 3. 追加新消息
        messages = [
            *cleaned_messages,
            UserMessage(content=new_prompt)
        ]

        # 4. 重新运行
        return self.run(agent_id, messages, replacement_state)
```

</details>

---

## 附录：关键文件索引

| 文件 | 位置 | 说明 |
|------|------|------|
| AgentTool.tsx | tools/AgentTool/ | 入口工具定义 |
| runAgent.ts | tools/AgentTool/ | 核心运行引擎 |
| forkSubagent.ts | tools/AgentTool/ | Fork 实现 |
| resumeAgent.ts | tools/AgentTool/ | 恢复机制 |
| loadAgentsDir.ts | tools/AgentTool/ | Agent 定义加载 |
| builtInAgents.ts | tools/AgentTool/ | 内置 Agent 列表 |
| prompt.ts | tools/AgentTool/ | AgentTool 提示词 |
| generalPurposeAgent.ts | tools/AgentTool/built-in/ | 通用 Agent |
| exploreAgent.ts | tools/AgentTool/built-in/ | 探索 Agent |
| planAgent.ts | tools/AgentTool/built-in/ | 计划 Agent |
| verificationAgent.ts | tools/AgentTool/built-in/ | 验证 Agent |
| generateAgent.ts | components/agents/ | AI 生成 Agent |
| types.ts | components/agents/ | 类型定义 |

---

*文档生成时间: 2026-04-09*
