# 查询引擎与工具系统

## 一、查询引擎 QueryEngine

**文件**: `src/QueryEngine.ts` (~1295 行)

### 设计定位

`QueryEngine` 封装了 AI 对话的核心生命周期，是一个**可复用类**，同时被 REPL 和 SDK 路径使用。每个对话对应一个实例，状态在多次轮次间持久保持。

### 核心接口

```typescript
class QueryEngine {
  constructor(config: QueryEngineConfig)
  
  // 提交用户消息，返回异步生成器
  submitMessage(prompt: string, options?): AsyncGenerator<SDKMessage>
  
  // 中断当前查询
  interrupt(): void
  
  // 读取状态
  getMessages(): SDKMessage[]
  getReadFileState(): ReadFileState
  
  // 切换模型
  setModel(model: string): void
}
```

### 请求生命周期

```
submitMessage(prompt)
  │
  ├── 包装 canUseTool → 追踪权限拒绝
  ├── 获取系统提示 + 记忆 + 用户上下文
  ├── 处理用户输入 (斜杠命令、附件)
  ├── 保存消息到记录 (transcript)
  ├── yield system_init 事件
  │
  └── → queryLoop() [详见 query.ts]
```

**便利函数 `ask()`**: 创建 QueryEngine → 调用 submitMessage → 清理资源，用于一次性非交互场景。

### 关键特性

- **特征标记死代码消除**: 使用 `feature()` 控制实验性模块的加载
- **惰性 React 导入**: React 重型模块使用 `require()` 延迟加载，避免启动时引入 Ink
- **消息分片**: 将可变消息分为 `this.mutableMessages` 和局部 `messages` 两份，安全处理 compact 边界 GC

## 二、查询循环 query()

**文件**: `src/query.ts` (~1730 行)

### 核心循环 queryLoop

```
queryLoop(state):
  while (true) {
    1. 预处理:
       ├── Skill 发现预取
       ├── Microcompact (微压缩)
       ├── Snip compact (裁剪压缩)
       ├── Context collapse (上下文折叠)
       └── Autocompact (自动压缩)

    2. API 调用: callModel(messages, systemPrompt)
       ├── 流式收集助手消息 + 工具调用块
       └── 错误恢复:
           ├── 回退模型
           ├── 图片大小错误
           ├── prompt-too-long → 反应式 compact
           ├── max_output_tokens → 递增重试
           └── abort 信号

    3. 终止检查:
       ├── 无工具调用 → 停止 hooks
       └── token 预算检查

    4. 工具执行:
       ├── StreamingToolExecutor (流式)
       └── runTools() (同步)

    5. 后处理:
       ├── Memory prefetch consume
       ├── Skill 发现附件注入
       └── 工具使用摘要

    6. 继续检查:
       ├── max_turns 限制
       └── → 循环或终止
  }
```

### 关键设计

- **`State` 对象**: 所有可变跨迭代状态集中在一个类型中，循环顶部解构，继续点通过 `state = { ... }` 更新
- **AsyncGenerator**: 整个循环作为 AsyncGenerator 产出事件，支持流式消费
- **`withheld` 模式**: 某些 API 错误（prompt-too-long, max_output_tokens）在流式输出期间对 SDK 调用方保留，直到恢复循环确定不可恢复才暴露
- **错误恢复级联**: 多个错误恢复策略按严重程度级联尝试

## 三、工具系统 Tool

**文件**: `src/Tool.ts` (~793 行)

### Tool 接口

```typescript
interface Tool<Input, Output, P> {
  name: string
  aliases?: string[]
  inputSchema / inputJSONSchema   // Zod 或 JSON Schema
  outputSchema                    // 可选
  description(input, options)     // 给模型的描述
  prompt(options)                 // 系统提示
  call(args, context, canUseTool, parentMessage, onProgress?)
  isEnabled()                     // 是否启用
  isReadOnly(input)               // 只读?
  isConcurrencySafe(input)        // 可并发?
  isDestructive?(input)           // 破坏性?
  validateInput?(input, context)  // 输入验证
  checkPermissions(input, context) // 权限检查
  renderToolUseMessage()          // React UI 渲染
  renderToolResultMessage()
  // ... 更多渲染和元数据方法
}
```

### ToolDef 与 buildTool

```typescript
// 工具作者定义的简化接口
interface ToolDef<Input, Output, P> = Partial<Tool> 中非必需字段可选

// 工厂函数：补充默认值
function buildTool(def: ToolDef): Tool
```

默认值：`isEnabled=true`, `isConcurrencySafe=false`, `isReadOnly=false`, `isDestructive=false`

### 工具集合

```typescript
type Tools = readonly Tool[]
function findToolByName(tools: Tools, name: string): Tool | undefined
```

### ToolUseContext

传递给每个工具 `call()` 方法的上下文对象，包含：
- Options (命令、工具、MCP 客户端、模型、thinking 配置、预算)
- App state 访问器
- Abort controllers
- 读文件状态缓存
- Hooks 回调
- 等 50+ 属性

## 四、工具执行服务

**目录**: `src/services/tools/`

### 工具编排

```
toolExecution.ts  (核心执行逻辑)
    ↓
StreamingToolExecutor.ts  (流式并发执行器)
    ↓
toolHooks.ts  (pre/post hooks 包装)
    ↓
toolOrchestration.ts  (批处理编排)
```

### StreamingToolExecutor

- **并发安全工具** 可以并行执行
- **非并发安全工具** 获得独占访问
- 缓冲结果并按工具接收顺序发射
- 维护跟踪工具队列（状态：queued / executing / completed / yielded）

### 批处理编排

```typescript
// 将工具调用分为并发安全批次和顺序批次
// 并发批次: runToolsConcurrently() 默认最多 10 个并发
// 顺序批次: 一次执行一个
```

### 工具 Hooks

- `executePreToolHooks` — 执行前（阻塞）
- `executePostToolHooks` — 执行后
- `executePostToolUseFailureHooks` — 失败后

## 五、40+ 工具目录一览

| 工具 | 目录 | 作用 |
|------|------|------|
| **AgentTool** | `AgentTool/` | 启动子 Agent (后台任务) |
| **AskUserQuestionTool** | `AskUserQuestionTool/` | 向用户提问 |
| **BashTool** | `BashTool/` | 执行 Shell 命令 |
| **BriefTool** | `BriefTool/` | 发送格式化消息给用户 |
| **ConfigTool** | `ConfigTool/` | 读写配置 |
| **EnterPlanModeTool** | `EnterPlanModeTool/` | 进入计划模式 |
| **EnterWorktreeTool** | `EnterWorktreeTool/` | 创建 git worktree |
| **ExitPlanModeTool** | `ExitPlanModeTool/` | 退出计划模式 |
| **ExitWorktreeTool** | `ExitWorktreeTool/` | 退出 worktree |
| **FileEditTool** | `FileEditTool/` | 文件内容编辑 |
| **FileReadTool** | `FileReadTool/` | 文件读取 |
| **FileWriteTool** | `FileWriteTool/` | 文件创建/覆写 |
| **GlobTool** | `GlobTool/` | 文件通配搜索 |
| **GrepTool** | `GrepTool/` | 文件内容搜索 |
| **LSPTool** | `LSPTool/` | LSP 代码导航 |
| **MCPTool** | `MCPTool/` | MCP 工具通用包装 |
| **McpAuthTool** | `McpAuthTool/` | MCP OAuth 认证 |
| **NotebookEditTool** | `NotebookEditTool/` | Jupyter 笔记本编辑 |
| **PowerShellTool** | `PowerShellTool/` | Windows PowerShell |
| **REPLTool** | `REPLTool/` | 透明包装层 (隐藏基础工具) |
| **ReadMcpResourceTool** | `ReadMcpResourceTool/` | 读取 MCP 资源 |
| **RemoteTriggerTool** | `RemoteTriggerTool/` | 远程触发器管理 |
| **ScheduleCronTool** | `ScheduleCronTool/` | 定时任务 |
| **SendMessageTool** | `SendMessageTool/` | Agent 间消息 |
| **SkillTool** | `SkillTool/` | 调用 Skills |
| **SleepTool** | `SleepTool/` | 等待/休眠 |
| **SyntheticOutputTool** | `SyntheticOutputTool/` | 结构化输出 |
| **TaskCreateTool** | `TaskCreateTool/` | 创建任务 |
| **TaskGetTool** | `TaskGetTool/` | 获取任务 |
| **TaskListTool** | `TaskListTool/` | 列出任务 |
| **TaskOutputTool** | `TaskOutputTool/` | 获取任务输出 |
| **TaskStopTool** | `TaskStopTool/` | 停止任务 |
| **TaskUpdateTool** | `TaskUpdateTool/` | 更新任务 |
| **TeamCreateTool** | `TeamCreateTool/` | 创建 Agent 团队 |
| **TeamDeleteTool** | `TeamDeleteTool/` | 删除 Agent 团队 |
| **TodoWriteTool** | `TodoWriteTool/` | 写入待办列表 |
| **ToolSearchTool** | `ToolSearchTool/` | 搜索延迟加载工具 |
| **WebFetchTool** | `WebFetchTool/` | 获取网页内容 |
| **WebSearchTool** | `WebSearchTool/` | 网页搜索 |
| **UsageTool** | `UsageTool/` | 使用量查询 (从 commands 迁移) |

## 六、共享工具功能

**目录**: `src/tools/shared/`

| 文件 | 用途 |
|------|------|
| `gitOperationTracking.ts` | Git 操作检测和遥测 |
| `spawnMultiAgent.ts` (~1094行) | 多 Agent 启动逻辑 (tmux/iTerm2/进程内) |

## 七、测试工具

**目录**: `src/tools/testing/`

`TestingPermissionTool.tsx` — 仅测试环境启用的工具，始终触发权限对话框，用于权限系统的端到端测试。
