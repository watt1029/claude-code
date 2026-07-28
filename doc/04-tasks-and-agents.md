# 任务系统与 Agent 执行

## 一、任务抽象 Task

**文件**: `src/Task.ts`

### TaskType

```typescript
type TaskType = 
  | 'local_bash'           // 本地 Shell 命令
  | 'local_agent'          // 本地子 Agent (Agent 工具触发)
  | 'remote_agent'         // 远程 CCR Agent
  | 'in_process_teammate'  // 进程内团队队友
  | 'local_workflow'       // 本地工作流
  | 'monitor_mcp'          // MCP 监控
  | 'dream'                // 自动记忆处理
```

### 生命周期

```
pending → running → completed | failed | killed
```

### Task 接口

```typescript
interface Task {
  name: string
  type: TaskType
  kill(taskId, setAppState): void  // 终止任务
}

interface TaskStateBase {
  id: string
  type: TaskType
  status: TaskStatus
  description: string
  toolUseId: string
  startTime: number
  endTime?: number
  totalPausedMs?: number
  outputFile: string
  outputOffset: number
  notified: boolean
}
```

### 辅助函数

- `generateTaskId(type)`: 生成 9 字符任务 ID (前缀: b=local_bash, a=local_agent, t=in_process_teammate, d=dream)
- `createTaskStateBase()`: 工厂函数
- `isTerminalTaskStatus()`: 终止状态守卫

## 二、任务类型实现

**目录**: `src/tasks/`

### 1. LocalAgentTask (异步 Agent 执行)

**文件**: `LocalAgentTask/LocalAgentTask.tsx` (~83KB)

主要管理通过 Agent 工具启动的后台异步 Agent 的执行。这是后台 Agent 的核心实现。

**状态追踪**:
- `AgentProgress` / `ProgressTracker`: tool 使用计数、token 计数 (输入/输出分开)、最近的 ToolActivity
- `ToolActivity`: 记录工具名、输入、预计算的活动描述

**特性**:
- 后台化执行
- Speculation abort (猜测性中止)
- Task 通知 XML
- SDK 进度发射

### 2. LocalShellTask (本地 Shell 执行)

**文件**: `LocalShellTask/LocalShellTask.tsx` (~66KB)

管理本地 bash 命令作为后台任务的执行。

**关键特性**:
- **Stall watchdog**: 每 5 秒监控输出文件。如果输出停滞 45 秒且最后一行匹配提示模式，生成通知
- `looksLikePrompt()`: 正则匹配常见交互提示模式 (y/n、Continue?)
- 支持 `kind: 'bash' | 'monitor'` 两种显示变体

### 3. RemoteAgentTask (远程 Agent)

**文件**: `RemoteAgentTask/RemoteAgentTask.tsx` (~126KB)

管理远程 CCR 会话中的 Agent 执行。

**特性**:
- 支持 `RemoteTaskType`: `remote-agent | ultraplan | ultrareview | autofix-pr | background-pr`
- `RemoteTaskCompletionChecker`: 可插拔完成检查器
- 远程审查进度追踪 (bug found/verified/refuted)
- Ultraplan 阶段追踪 (`needs_input`, `plan_ready`)

### 4. InProcessTeammateTask (进程内团队队友)

**文件**: `InProcessTeammateTask/InProcessTeammateTask.tsx` (~16KB)

管理进程内 swarm 团队中的队友。

**关键区别**: 与 LocalAgentTask 不同：
- 在同一个 Node.js 进程内运行 (AsyncLocalStorage 隔离)
- 有团队感知身份 (`agentName@teamName`)
- 支持计划模式审批流 (`awaitingPlanApproval`)
- 可以处于空闲 (等待工作) 或活跃 (处理中) 状态

### 5. DreamTask (自动记忆)

**文件**: `DreamTask/DreamTask.tsx`

轻量级 UI 任务，用于在界面上显示自动记忆 (auto-dream) 子 Agent 的处理状态。纯粹是 UI 可见性用途。

## 三、Agent 上下文隔离

**文件**: `src/utils/agentContext.ts`

使用 `AsyncLocalStorage` 实现分析归属的上下文传递，无需参数透传。

```typescript
type AgentContext = 
  | { agentType: 'subagent'; agentId; parentSessionId; subagentName; ... }
  | { agentType: 'teammate'; agentName; teamName; agentColor; ... }
```

**关键**: 使用 `AsyncLocalStorage` 而非 `AppState`，因为多个 Agent 可以在同进程中并发运行，AsyncLocalStorage 隔离每个异步执行链。

## 四、Swarm (Agent 团队)

**目录**: `src/utils/swarm/`

### 架构

在 swarm 模式下，一个领导 Agent 协调多个团队队友 Agent。

### 队友可视化后端

| 后端 | 文件 | 平台 |
|------|------|------|
| **TmuxBackend** | `backends/TmuxBackend.ts` | Linux/macOS |
| **ITermBackend** | `backends/ITermBackend.ts` | macOS (iTerm2) |
| **InProcessBackend** | `backends/InProcessBackend.ts` | 所有平台 (无终端面板) |
| **PaneBackendExecutor** | `backends/PaneBackendExecutor.ts` | 编排器 |

### 核心文件

| 文件 | 作用 |
|------|------|
| `inProcessRunner.ts` (~53KB) | 进程内队友执行包装器 |
| `spawnInProcess.ts` (~10KB) | 创建/注册进程内队友任务 |
| `teamHelpers.ts` (~21KB) | 团队生命周期操作 |
| `permissionSync.ts` (~26KB) | 领导/队友间权限同步 |
| `teammateInit.ts` (~4.3KB) | 队友初始化逻辑 |
| `reconnection.ts` (~3.4KB) | Swarm 会话重连 |

## 五、协调器模式 Coordinator Mode

**文件**: `src/coordinator/coordinatorMode.ts` (~19KB)

一种特殊的编排模式，Claude Code 作为协调器指导多个 Worker Agent。

**流程**: 
```
Research (并行 Worker) → Synthesis (协调器) 
→ Implementation (Worker) → Verification (Worker)
```

通过 `CLAUDE_CODE_COORDINATOR_MODE` 环境变量启用。

## 六、生成任务 ID

```
local_bash:             bxxxxxxx
local_agent:            axxxxxxx
in_process_teammate:    txxxxxxx
dream:                  dxxxxxxx
remote_agent / monitor / workflow: 其他前缀
```
