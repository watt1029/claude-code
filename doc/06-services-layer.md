# 服务层

## 一、Analytics 分析服务

**目录**: `src/services/analytics/`

| 文件 | 作用 |
|------|------|
| `index.ts` | 分析入口 - `logEvent()` 统一事件记录 |
| `metadata.ts` | 事件元数据清洗 (防止代码/文件路径泄露) |
| `growthbook.ts` | GrowthBook 特征标记集成 |
| `config.ts` | 分析配置 |
| `datadog.ts` | DataDog 指标 |
| `firstPartyEventLogger.ts` | 1P 事件日志 |
| `firstPartyEventLoggingExporter.ts` | 1P 导出器 |
| `sink.ts` | 分析 sink |
| `sinkKillswitch.ts` | Sink 终止开关 |

### 数据流

```
工具调用/事件 → logEvent() → Statsig + OTel + (DataDog)
                            → 事件清洗 (metadata.ts)
                            → GrowthBook 特征评估
```

## 二、API 客户端层

**目录**: `src/services/api/`

| 文件 | 作用 |
|------|------|
| `client.ts` | HTTP 客户端配置 (axios) |
| `claude.ts` | Claude API 请求封装 (queryHaiku 等) |
| `bootstrap.ts` | API 启动信息获取 |
| `adminRequests.ts` | 管理 API 请求 |
| `filesApi.ts` | 文件 API |
| `errorUtils.ts` / `errors.ts` | API 错误处理/分类 |
| `logging.ts` | API 请求日志 |
| `metricsOptOut.ts` | 指标退出机制 |
| `firstTokenDate.ts` | 首次 token 延迟追踪 |
| `promptCacheBreakDetection.ts` | 提示缓存中断检测 |
| `overageCreditGrant.ts` | 超量信用授予 |
| `grove.ts` | Grove 集成 |
| `dumpPrompts.ts` | 提示转储调试 |
| `emptyUsage.ts` | 空使用量处理 |

### 工具 Schema 转换 (utils/api.ts)

内部 `Tool` 对象到 Anthropic API 兼容格式的转换：

```typescript
toolToAPISchema(tool, options): BetaToolUnion
  // 缓存基础 schema (会话级)
  // 添加 per-request overlay (defer_loading, cache_control, strict)
  // 无效开关: CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS
```

## 三、MCP 服务

**目录**: `src/services/mcp/` (24 个文件)

MCP (Model Context Protocol) 是 Claude Code 与外部服务交互的标准协议。

### 核心模块

| 模块 | 文件 | 作用 |
|------|------|------|
| **客户端** | `client.ts` (~119KB) | MCP 客户端核心：连接管理、工具/资源/提示列表、请求处理 |
| **配置** | `config.ts` (~51KB) | MCP 服务器配置解析、环境变量扩展、服务器类型检测 |
| **认证** | `auth.ts` (~89KB) | OAuth 认证流程、token 管理、凭据存储 |
| **UI 管理** | `useManageMCPConnections.ts` (~45KB) | React Hook：连接状态、重连逻辑 |
| **Channel** | `channelPermissions.ts` | 通信频道权限 |
| | `channelNotification.ts` | 频道通知 |
| | `channelAllowlist.ts` | 频道白名单 |
| **传输层** | `SdkControlTransport.ts` | SDK 控制传输 |
| | `InProcessTransport.ts` | 进程内传输 (函数直接调用) |
| **集成** | `claudeai.ts` | Claude.ai 集成 |
| | `vscodeSdkMcp.ts` | VS Code SDK 集成 |
| | `xaa.ts` | 跨账户访问 |
| | `elicitationHandler.ts` | 交互式提示处理 |
| **其他** | `headersHelper.ts`, `mcpStringUtils.ts`, `normalization.ts`, `envExpansion.ts` |

## 四、LSP 服务

**目录**: `src/services/lsp/`

| 文件 | 作用 |
|------|------|
| `LSPServerManager.ts` | LSP 服务器管理器 (全局单例) |
| `LSPServerInstance.ts` | 单个 LSP 服务器实例 |
| `LSPClient.ts` | LSP 客户端 (语言协议交互) |
| `LSPDiagnosticRegistry.ts` | 诊断注册表 |
| `config.ts` | LSP 配置 |
| `manager.ts` | LSP 管理逻辑 |
| `passiveFeedback.ts` | 被动反馈生成 |

## 五、OAuth 服务

**目录**: `src/services/oauth/`

| 文件 | 作用 |
|------|------|
| `client.ts` | OAuth 客户端 (token 管理、组织 UUID) |
| `auth-code-listener.ts` | 认证码监听服务器 |
| `crypto.ts` | OAuth 加密工具 |
| `getOauthProfile.ts` | 获取 OAuth 资料 |
| `index.ts` | 统一入口 |

## 六、Context Compact (上下文压缩)

**目录**: `src/services/compact/`

当对话上下文过长时，自动压缩以保持 token 预算。

| 文件 | 作用 |
|------|------|
| `compact.ts` | 核心压缩逻辑 |
| `autoCompact.ts` | 自动压缩决策 |
| `microCompact.ts` | 微压缩 (快速剪枝) |
| `apiMicrocompact.ts` | API 辅助微压缩 |
| `sessionMemoryCompact.ts` | 会话记忆压缩 |
| `prompt.ts` | 压缩提示模板 |
| `grouping.ts` | 消息分组 |
| `postCompactCleanup.ts` | 压缩后清理 |
| `compactWarningHook.ts` | 压缩警告 hook |
| `timeBasedMCConfig.ts` | 基于时间的压缩配置 |

## 七、AutoDream (自动记忆处理)

**目录**: `src/services/autoDream/`

| 文件 | 作用 |
|------|------|
| `autoDream.ts` | 自动梦境任务处理 |
| `config.ts` | 梦境配置 |
| `consolidationLock.ts` | 整合锁 (防止并发) |
| `consolidationPrompt.ts` | 整合提示 |

## 八、其他服务

| 服务目录 | 文件数 | 作用 |
|----------|--------|------|
| `services/plugins/` | 3 | 插件安装管理、CLI 命令、操作 |
| `services/tools/` | 4 | 工具执行管线 [见 03-query-engine] |
| `services/SessionMemory/` | 3 | 会话记忆管理 |
| `services/extractMemories/` | 2 | 记忆提取到 CLAUDE.md |
| `services/PromptSuggestion/` | 2 | 提示建议与推测 |
| `services/AgentSummary/` | 1 | Agent 活动摘要 |
| `services/MagicDocs/` | 2 | 魔术文档生成 |
| `services/policyLimits/` | 2 | 基于策略的资源限制 |
| `services/remoteManagedSettings/` | 5 | 远程托管设置同步 |
| `services/settingsSync/` | 2 | 跨设备设置同步 |
| `services/teamMemorySync/` | 5 | 团队记忆同步 (含秘密扫描) |
| `services/tips/` | 3 | 每日提示系统 |
| `services/toolUseSummary/` | 1 | 工具使用摘要生成 |

## 九、服务层数据流示例

```
用户输入
  → QueryEngine
    → api/client.ts (HTTP 请求)
      → claude.ts (API 调用)
        → Anthropic API
          → 流式响应
            → StreamingToolExecutor
              → MCPTool.call() → services/mcp/client.ts
              → BashTool.call() → utils/bash/parser.ts
              → FileEditTool.call() → services/lsp/diagnostics
```

```
工具执行后
  → services/tools/toolHooks.ts
  → services/analytics/logEvent()
  → utils/telemetry/
```
