# 入口点与 CLI 系统

## 一、进程入口 main.tsx

**文件**: `src/main.tsx`

这是整个应用的进程入口，通过 Commander.js 定义 CLI 接口。

### 启动流程

```
main.tsx
    │
    ├── 1. 副作用导入 (profile, MDM, keychain)
    │       这些在模块加载时立即执行，利用导入时间并行化
    │
    ├── 2. Commander.js 命令行定义
    │       --print / -p       非交互模式
    │       --model             指定模型
    │       --worktree          创建 git worktree
    │       --dangerously-skip-permissions  跳过权限
    │       --remote            远程/SSH 会话
    │       --sdk               SDK 模式
    │       等 40+ 参数
    │
    ├── 3. init() 环境初始化
    │       → 配置系统、环境变量、CA证书、OAuth缓存
    │       → 远程托管设置、mTLS/代理、API预连接
    │
    ├── 4. 分发子命令
    │       claude login        OAuth 登录
    │       claude mcp *        MCP 管理
    │       claude doctor       诊断
    │       claude plugin *     插件管理
    │       claude remote-setup 远程设置
    │       claude config       配置
    │       等 20+ 子命令
    │
    ├── 5. --print / --sdk 模式
    │       → CLI.print() 或 Structured I/O
    │
    └── 6. 交互式模式 (默认)
            → setup() 会话初始化
            → launchRepl() 启动 REPL
```

### 关键特性

- **启动性能优化**: 使用 `profileCheckpoint()` 追踪启动耗时，命令处理函数懒加载以避免影响启动时间
- **特征标记**: 通过 `feature()` 实现编译时死代码消除
- **会话 ID 管理**: 支持 `--session-id` 参数恢复历史会话

## 二、环境初始化 init()

**文件**: `src/entrypoints/init.ts`

```
init()
 ├── enableConfigs()          配置系统启用
 ├── 设置安全环境变量          env, CA 证书
 ├── 注册优雅关闭处理器        cleanup handlers
 ├── 1P 事件日志初始化        懒加载 OpenTelemetry
 ├── OAuth 账户信息缓存        populateCachedAccountInfo
 ├── IDE 检测 + GitHub 检测   JetBrains, git remote
 ├── 远程托管设置              remote managed settings
 ├── 策略限制                  policy limits
 ├── mTLS + 代理配置          configureGlobalMTLS/Agents
 ├── API 预连接                warm TLS to API
 ├── CCR 上游代理              upstream proxy (远程模式)
 └── LSP 清理 / 团队清理 / scratchpad
```

**额外的 `initializeTelemetryAfterTrust()`**: 在信任对话框接受后调用，初始化完整的 OTLP 遥测。

## 三、会话初始化 setup()

**文件**: `src/setup.ts`

```
setup(cwd, permissionMode, ...)
 ├── Node.js 版本检查          < 18 则退出
 ├── 自定义 session ID
 ├── UDS 消息服务器            Unix socket (特性标记)
 ├── 终端备份恢复              iTerm2/Terminal.app
 ├── 设置 CWD                  setCwd(cwd)
 ├── Hooks 配置快照
 ├── FileChanged hook 监听器
 ├── Worktree 创建             可选的 git worktree
 ├── 后台服务启动
 │   ├── 会话记忆 (SessionMemory)
 │   ├── 上下文折叠 (Context Collapse)
 │   ├── 插件 hook 加载
 │   ├── 归属信息 hooks
 │   └── 团队记忆监听器
 ├── 错误日志 + 分析 sink
 ├── API key 预获取
 └── 权限安全检查              --dangerously-skip-permissions validation
```

## 四、CLI 子命令系统

**目录**: `src/cli/` + `src/cli/handlers/` + `src/cli/transports/`

### CLI 核心文件

| 文件 | 作用 |
|------|------|
| `print.ts` (213KB) | `--print` 非交互模式主驱动：查询循环、工具执行、流式输出 |
| `structuredIO.ts` (29KB) | SDK 结构化 JSON I/O 协议 (Python SDK 等) |
| `remoteIO.ts` (10KB) | 远程控制 I/O (移动端/Web 客户端) |
| `update.ts` (14KB) | 发布更新检查和安装进度 |
| `exit.ts` | 子命令退出辅助函数 |

### CLI 子命令处理器

| 处理器 | 子命令 |
|--------|--------|
| `handlers/util.tsx` | `setup-token`, `doctor`, `install` |
| `handlers/auth.ts` | `login`, `logout`, `whoami`, `--force-login` |
| `handlers/mcp.tsx` (56KB) | `mcp serve/add/remove/list/get/auth/import/export` |
| `handlers/plugins.ts` (31KB) | `plugin install/uninstall/list/search/marketplace` |
| `handlers/agents.ts` | `agents list` |
| `handlers/autoMode.ts` | `auto-mode defaults/config` |

### 传输层 (transports)

| 文件 | 用途 |
|------|------|
| `HybridTransport.ts` | 混合传输协议 |
| `SSETransport.ts` | Server-Sent Events |
| `WebSocketTransport.ts` | WebSocket |
| `ccrClient.ts` | CCR 远程客户端 |
| `SerialBatchEventUploader.ts` | 远程事件批量上传 |
| `WorkerStateUploader.ts` | 工作进程状态上传 |

## 五、命令系统 (Slash Commands)

**文件**: `src/commands.ts` | **命令实现**: `src/commands/` (80+ 命令)

### 命令类型

```typescript
type Command = 
  | { type: 'local'; name: string; load: () => Promise<...> }       // 进程内执行
  | { type: 'local-jsx'; name: string; load: () => Promise<...> }  // 渲染 React UI
  | { type: 'prompt'; name: string; load: () => Promise<...> }     // 扩展为模型内容
```

### 命令来源 (合并顺序)

1. 内置捆绑 skills
2. 内置插件 skills
3. Skills 目录命令
4. 工作流命令
5. 插件命令
6. 插件 skills
7. 内置命令 (`COMMANDS()`)

### 命令结构模式 (每个命令)

```
commands/<name>/
 ├── index.ts      # 元数据 + lazy load 引用
 └── impl.ts       # 实际实现 (按需加载)
```

### 安全过滤

- `REMOTE_SAFE_COMMANDS`: 远程模式下安全的命令集合
- `BRIDGE_SAFE_COMMANDS`: 桥接模式下安全的命令集合
- `isBridgeSafeCommand()`: 运行时安全检查

## 六、SDK 入口

**目录**: `src/entrypoints/sdk/`

| 文件 | 用途 |
|------|------|
| `coreSchemas.ts` (56KB) | Zod v4 核心 schema (所有可序列化数据类型) |
| `coreTypes.ts` | 公开 SDK 类型入口 |
| `controlSchemas.ts` (20KB) | SDK 控制协议 schema (权限请求、hook 回调等) |
| `agentSdkTypes.ts` | SDK 消息类型定义 |
| `sandboxTypes.ts` | 沙箱相关类型 |
