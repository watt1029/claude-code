# 权限系统与安全机制

## 一、权限系统概述

Claude Code 拥有一个**多层门控机制**来控制工具执行。每个工具调用在真正执行前都要经过权限检查。

## 二、权限模式 (PermissionMode)

**定义**: `src/utils/permissions/PermissionMode.ts`

```typescript
type PermissionMode =
  | 'default'       // 默认：每次都询问
  | 'acceptEdits'   // 接受编辑：自动批准文件修改
  | 'bypass'        // 跳过：自动批准所有 (dangerously-skip-permissions)
  | 'plan'          // 计划模式：只读，需审批
  | 'auto'          // 自动模式：AI 决策是否批准
  | 'dontAsk'       // 不问：拒绝所有
  | 'bubble'        // 冒泡：传递给上层决策者
```

## 三、权限检查流程

```
工具执行前 → checkPermissions() 
    │
    ├── 1. 权限规则检查 (permissionsLoader)
    │       从设置加载 allow/deny/ask 规则
    │       规则来自所有设置源 (合并后)
    │
    ├── 2. Shell 命令分类 (shellRuleMatching)
    │       匹配 shell 命令模式
    │
    ├── 3. Bash 解析器安全检查 (bash/parser + bash/ast)
    │       基于 tree-sitter 的 AST 分析
    │       (门控: TREE_SITTER_BASH)
    │
    ├── 4. YOLO 分类器 (yoloClassifier)
    │       Auto模式下，用 AI 判断操作安全性
    │       (门控: BASH_CLASSIFIER)
    │
    ├── 5. 权限 Hook 检查 (hooks 子系统)
    │       PreToolUse hooks (agent/http/prompt)
    │
    ├── 6. 通道权限检查 (MCP channel)
    │       远程 MCP 通道权限
    │
    ├── 7. 沙箱检查 (sandbox-adapter)
    │       文件系统/网络限制
    │
    └── 8. 用户交互对话框 (交互模式)
            显示权限请求给用户
```

## 四、权限模块结构

**目录**: `src/utils/permissions/` (25 个文件)

| 文件 | 作用 |
|------|------|
| `permissions.ts` (~52KB) | 核心权限检查编排 |
| `permissionSetup.ts` | 权限初始化 |
| `permissionsLoader.ts` | 从设置加载规则 |
| `permissionRuleParser.ts` | 规则字符串解析 |
| `PermissionRule.ts` | 规则类型 |
| `PermissionResult.ts` | 结果类型 (allow/deny/ask) |
| `PermissionUpdate.ts` | 规则更新持久化 |
| `PermissionUpdateSchema.ts` | 更新 Schema |
| `PermissionMode.ts` | 模式定义 |
| `filesystem.ts` | 文件系统路径权限检查 |
| `pathValidation.ts` | 路径遍历检查 |
| `yoloClassifier.ts` | Auto 模式 AI 分类器 |
| `bashClassifier.ts` | Bash 命令分类器 |
| `classifierDecision.ts` | 分类决策逻辑 |
| `shellRuleMatching.ts` | Shell 规则匹配 |
| `dangerousPatterns.ts` | 危险命令模式 |
| `shadowedRuleDetection.ts` | 重叠规则检测 |
| `permissionExplainer.ts` | Haiku 生成权限说明 |
| `bypassPermissionsKillswitch.ts` | 绕过权限终止开关 |
| `denialTracking.ts` | 拒绝事件追踪 |
| `getNextPermissionMode.ts` | 模式切换 |
| `autoModeState.ts` | Auto 模式状态 |

## 五、权限 Hook 处理层

**目录**: `src/hooks/toolPermission/`

| 模块 | 职责 | 适用场景 |
|------|------|---------|
| `interactiveHandler.ts` | 主 Agent：推入确认队列 + 后台自动检查并发 | 主交互会话 |
| `coordinatorHandler.ts` | 协调器 Worker：顺序自动化检查 → 交互对话框 | 协调器模式 |
| `swarmWorkerHandler.ts` | Swarm Worker：分类器 → 转发给领导 | 团队模式 |

### 交互式流程 (interactiveHandler)

```
1. 推入 ToolUseConfirm 到确认队列
2. 后台并发运行:
   ├── Permission hooks (快速，本地)
   └── Bash 分类器 (慢，需推理)
3. 竞赛: 自动结果 vs 用户交互
4. Resolve-once 模式防止双重决议
```

## 六、沙箱系统

**目录**: `src/utils/sandbox/`

| 文件 | 作用 |
|------|------|
| `sandbox-adapter.ts` | 沙箱运行时代理 (wrapping @anthropic-ai/sandbox-runtime) |
| `sandbox-ui-utils.ts` | 沙箱 UI 辅助 |

### 沙箱配置

从设置读取沙箱配置（多源合并），配置：
- 文件系统读限制
- 文件系统写限制
- 网络限制 (主机模式)
- 违规处理

### 沙箱流程

```
Bash 执行前 → SandboxManager.shouldSandbox()
    → 将 Claude 权限规则转换为沙箱格式
    → 创建沙箱执行环境
    → 处理沙箱违规 (ask callback)
```

## 七、渠道权限 (MCP Channels)

**文件**: `src/services/mcp/channelPermissions.ts`

MCP 服务器通信的额外权限层，管理与远程 MCP 通道的权限交互。

## 八、桥接权限

**文件**: `src/bridge/bridgePermissionCallbacks.ts`

处理远程控制 (CCR) 模式下的权限请求转发。

## 九、权限决策日志

**文件**: `src/hooks/toolPermission/permissionLogging.ts`

所有权限决策统一记录：

```
logPermissionDecision()
  → Statsig 分析事件
  → OTel 计数器
  → 代码编辑工具指标 (语言追踪)
```

## 十、Bash 安全分析

**目录**: `src/utils/bash/`

Bash 子系统的核心安全功能：

1. **语法解析**: 基于 tree-sitter 的完整 Bash 语法解析
2. **AST 安全遍历**: 遍历 AST 节点识别危险模式
3. **命令分类**: 将 shell 命令分为安全/危险/eval 类别
4. **攻击防护**: 时间/节点预算限制对抗性输入

### 危险模式检测

`dangerousPatterns.ts` 维护危险命令模式列表，包括：
- 文件系统破坏 (`rm -rf`, `mkfs`, `dd`)
- 网络攻击 (`nc`, `nmap`)
- 权限提升 (`sudo`, `su`, `chmod 777`)
- 代码执行 (`eval`, `$(...)`, 反引号)

## 十一、安全边界总览

```
┌─────────────────────────────────┐
│         交互式人机决策           │
│    (用户确认/拒绝每个操作)       │
├─────────────────────────────────┤
│         权限规则引擎             │
│    (allow/deny/ask 规则列表)     │
├─────────────────────────────────┤
│         YOLO 分类器              │
│    (AI 自动评估操作安全性)       │
├─────────────────────────────────┤
│          Bash 安全分析           │
│    (tree-sitter AST 安全遍历)    │
├─────────────────────────────────┤
│          沙箱运行时              │
│    (文件系统/网络访问限制)       │
├─────────────────────────────────┤
│       权限 Hook 系统             │
│    (agent/http/prompt hooks)    │
├─────────────────────────────────┤
│      操作系统/进程隔离           │
│    (Bun 运行时 + 内核安全)       │
└─────────────────────────────────┘
```
