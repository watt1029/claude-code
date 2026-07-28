# Claude Code 整体架构总览

## 一、项目定位

Claude Code 是一个 **基于终端的 AI 编程助手 (Code Agent)**。用户通过自然语言对话的方式与 AI 交互，AI 可以执行文件操作、运行命令、搜索代码、管理项目等软件开发任务。它既可以运行在交互式终端中（REPL 模式），也可以通过 SDK 被外部程序调用（非交互模式）。

## 二、整体分层架构

```
┌─────────────────────────────────────────────────────────┐
│                   用户交互层 (UI)                        │
│   Ink 终端渲染  ·  React 组件  ·  REPL 屏幕 ·  CLI    │
├─────────────────────────────────────────────────────────┤
│                   应用逻辑层                             │
│   QueryEngine (查询/响应循环)  ·  Tools (40+ 工具)      │
│   Tasks (任务系统)            ·  Commands (80+ 命令)   │
│   Services (MCP/API/分析)     ·  Plugins (插件)         │
│   Skills                     ·  Swarms (Agent 团队)    │
├─────────────────────────────────────────────────────────┤
│                   基础服务层                             │
│   权限系统   ·  沙箱   ·  配置/设置  ·  遥测/分析       │
│   记忆系统   ·  Hooks  ·  定时任务  ·  文件监听         │
├─────────────────────────────────────────────────────────┤
│                   桥接与远程层                            │
│   Bridge (远程控制)  ·  Remote (CCR WebSocket)          │
│   Server (Direct Connect)  ·  UpstreamProxy             │
├─────────────────────────────────────────────────────────┤
│                   基础设施层                              │
│   Ink (终端渲染引擎)  ·  Yoga (Flexbox 布局)             │
│   API Client (Anthropic SDK)  ·  Zod (验证)              │
│   OpenTelemetry  ·  GrowthBook                          │
└─────────────────────────────────────────────────────────┘
```

## 三、核心模块总览

| 模块 | 目录 | 职责 | 核心文件 |
|------|------|------|---------|
| **入口与启动** | `src/main.tsx` + `entrypoints/` | 进程入口、初始化 | `main.tsx`, `init()`, `setup()` |
| **CLI 子命令** | `src/cli/` | 命令行非交互模式、结构化 I/O | `print.ts`, `structuredIO.ts` |
| **查询引擎** | `src/` | LLM 查询/响应循环的核心 | `QueryEngine.ts`, `query.ts` |
| **工具系统** | `src/tools/` | 40+ 个 AI 可用工具的定义 | `Tool.ts`, 各工具目录 |
| **命令系统** | `src/commands/` | 80+ 个用户斜杠命令 | `commands.ts`, 各命令目录 |
| **任务系统** | `src/tasks/` | 后台任务 (Agent/Bash/远程) | `Task.ts`, 各任务实现 |
| **组件系统** | `src/components/` | Ink (React) 终端 UI 组件 | 各组件目录 |
| **渲染引擎** | `src/ink/` | 自定义终端渲染引擎 | `ink.tsx`, `reconciler.ts` |
| **服务层** | `src/services/` | API客户端、MCP、分析、LSP等 | 各服务目录 |
| **权限系统** | `src/utils/permissions/` | 多层权限门控机制 | `permissions.ts` |
| **桥接系统** | `src/bridge/` | 远程控制 (CCR) 集成 | `bridgeMain.ts`, `replBridge.ts` |
| **插件系统** | `src/utils/plugins/` | 第三方插件生命周期 | `pluginLoader.ts` |
| **状态管理** | `src/state/` | 全局应用状态 | `AppState.tsx`, `AppStateStore.ts` |
| **钩子系统** | `src/utils/hooks/` | 事件驱动的 hook 系统 | 各 hook 执行器 |
| **配置系统** | `src/utils/` | 全局/项目配置 | `config.ts` |
| **设置系统** | `src/utils/settings/` | 多源合并设置 | `settings.ts` |

## 四、关键设计模式

### 1. 分层初始化 (Layered Initialization)

启动流程分为三步：
1. **`init()`** — 环境初始化（网络、证书、代理、OAuth 缓存）
2. **`setup()`** — 会话初始化（工作目录、worktree、后台服务）
3. **REPL / CLI** — 用户交互层启动

每个阶段可以独立失败，且异步初始化尽早触发以隐藏延迟。

### 2. 编译时特征标记 (Feature Flags at Compile Time)

使用 Bun 的 `bun:bundle` 模块提供的 `feature()` 函数实现编译时死代码消除：

```typescript
import { feature } from 'bun:bundle'
if (feature('VOICE_MODE')) { /* voice code */ }
```

未启用的特征对应模块在构建时被完全 tree-shake 掉，不产生运行时开销。

### 3. 惰性加载工具与命令 (Lazy Loading)

所有工具和命令都采用 `load: () => import('./impl.js')` 模式：

```typescript
const myCommand = {
  type: 'local',
  name: 'my-command',
  load: () => import('./my-command.js'),
} satisfies Command
```

这样启动时只加载元数据，实际实现按需动态导入。

### 4. 构建工具模式 (buildTool Factory)

所有工具通过 `buildTool()` 工厂函数创建，自动填充默认值：

```typescript
const MyTool = buildTool({
  name: 'my_tool',
  call: async (args, ctx) => { ... },
  // isEnabled, isConcurrencySafe 等自动获得默认值
})
```

### 5. 异步生成器数据流 (AsyncGenerator)

查询引擎的 `submitMessage()` 返回 `AsyncGenerator<SDKMessage>`，消费方通过 `for await...of` 消费。这允许流式处理中间事件（stream events、tool progress 等）。

### 6. 多源合并设置 (Layered Settings)

设置从多个来源合并，优先级递增：
1. User settings (`~/.claude/settings.json`)
2. Project settings (`.claude/settings.json`)
3. Local settings (`.claude/settings.local.json`)
4. CLI flag settings (`--settings`)
5. Managed/policy settings

### 7. 任务抽象 (Task Polymorphism)

通过 `TaskType` 区分任务类型，使用 `Task` 接口进行多态调度：

```typescript
type TaskType = 'local_bash' | 'local_agent' | 'remote_agent' | 'in_process_teammate' | 'local_workflow' | 'monitor_mcp' | 'dream'
```

### 8. 双层 React 渲染 (Ink + Components)

Ink 层提供**基础渲染能力**（Box/Text、Flexbox 布局、事件处理），`components/` 层构建**应用级 UI**（消息、权限对话框、设置界面）。

### 9. AsyncLocalStorage 隔离

在多 Agent 场景中（swarm 模式），使用 Node.js 的 `AsyncLocalStorage` 实现并发 Agent 的上下文隔离，而非全局状态。

### 10. 记忆/状态持久化 (Memory/Memdir)

自动记忆系统使用文件系统中的 Markdown 文件存储持久化记忆（用户偏好、项目知识、反馈等），通过前文元数据检索相关记忆。

## 五、请求-响应数据流

```
用户输入 → processUserInput() → QueryEngine.submitMessage()
    → queryLoop() [主循环]
        → [预处理: microcompact / snip / context collapse / autocompact]
        → callModel() [API 调用]
            → 流式响应 + 工具调用块
            → [需要回复?] → StreamingToolExecutor (并发执行工具)
                → 各 Tool.call()
                → 工具结果收集
            → [继续?] → 循环回到 callModel()
        → [终止] → 返回结果
```

## 六、核心数据流路径

```
┌──────────┐   system prompt    ┌─────────────┐
│  用户输入  │ ──────────────→   │ QueryEngine  │
│          │   + context        │             │
│  /命令   │                    │ queryLoop() │
└──────────┘                    └──────┬──────┘
                                       │
                          ┌────────────┴────────────┐
                          │     callModel() / API    │
                          │  (Anthropic Messages API)│
                          └────────────┬────────────┘
                                       │
                          ┌────────────┴────────────┐
                          │   StreamingToolExecutor  │
                          │    并发/顺序工具执行      │
                          └──┬─────┬─────┬─────┬───┘
                             │     │     │     │
                   ┌─────────┘ ┌───┘ ┌───┘ ┌──┘
                   ▼           ▼     ▼     ▼
               BashTool   FileEdit   Read   WebFetch
                   │           │     │        │
                   ▼           ▼     ▼        ▼
               结果收集 → 回到 queryLoop
```
