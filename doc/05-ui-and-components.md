# UI 与组件系统

## 一、Ink 终端渲染引擎

**目录**: `src/ink/`

Ink 是 Claude Code 的自定义终端渲染引擎，基于 React reconciler 构建，将 React 组件树渲染为终端字符网格。

### 核心架构

```
React 组件树
     │
     ▼
React Reconciler (reconciler.ts)
     │
     ▼
DOM 元素模型 (dom.ts)
     │
     ▼
Yoga Flexbox 布局 (yoga-layout/)
     │
     ▼
渲染到输出 (render-node-to-output.ts)
     │
     ▼
屏幕缓冲 (screen.ts) + 双缓冲 (frame.ts)
     │
     ▼
终端输出 (terminal.ts → stdout)
```

### 关键模块

| 文件 | 行数 | 作用 |
|------|------|------|
| `ink.tsx` | ~1700 | Ink 主类：终端管理、帧渲染、Yoga 布局、React reconciler、键盘/鼠标事件 |
| `root.ts` | | `render()` / `createRoot()` 入口 |
| `reconciler.ts` | | 自定义 React reconciler host config |
| `screen.ts` | | 屏幕缓冲 (StylePool/CharPool/HyperlinkPool 内存优化) |
| `dom.ts` | | DOM 元素节点模型 |
| `frame.ts` | | 双缓冲帧数据 |
| `optimizer.ts` | | 节点树优化 (剪枝不可见/布局无关节点) |
| `output.ts` | | 节点到终端输出的渲染 |
| `terminal.ts` | | 终端抽象层 |
| `focus.ts` | | 键盘焦点管理 |
| `selection.ts` | | 终端文本选择 |
| `searchHighlight.ts` | | 搜索高亮 |

### 基础组件 (ink/components/)

| 组件 | 作用 |
|------|------|
| `Box.tsx` | Flexbox 布局容器 (Yoga) |
| `Text.tsx` | 文本渲染 |
| `Button.tsx` | 可交互按钮 |
| `Link.tsx` | OSC 8 超链接 |
| `ScrollBox.tsx` | 可滚动容器 |
| `Spacer.tsx` | 弹性占位 |
| `App.tsx` | 根组件：stdin/stdout、键盘/鼠标事件、终端模式 |
| `ErrorOverview.tsx` | React 错误边界显示 |

### 事件系统 (ink/events/)

| 文件 | 作用 |
|------|------|
| `event.ts` | Event 基类 |
| `emitter.ts` | EventEmitter |
| `dispatcher.ts` | 事件分发 |
| `keyboard-event.ts` | 键盘事件 |
| `click-event.ts` | 鼠标点击 |
| `input-event.ts` | 输入事件 |
| `terminal-event.ts` | 终端事件 |

### Hooks (ink/hooks/)

| Hook | 作用 |
|------|------|
| `use-input.ts` | 键盘输入订阅 |
| `use-app.ts` | Ink 实例访问 |
| `use-stdin.ts` | stdin 流访问 |
| `use-animation-frame.ts` | 逐帧动画 |
| `use-interval.ts` | 定时器 |
| `use-selection.ts` | 文本选择 API |
| `use-terminal-focus.ts` | 终端焦点状态 |

### ink.ts 的公开导出

```
render(node, options?)        // 渲染入口
createRoot(options?)          // 创建 Ink 根
Box / Text / Button / Link   // 主题化组件
ThemeProvider / useTheme      // 主题系统
useInput / useApp / useStdin  // Hooks
```

## 二、React 组件体系

**目录**: `src/components/`

### 组件目录总览

| 目录 | 作用 |
|------|------|
| **PromptInput/** | 聊天输入框：输入框、历史搜索、底部提示栏、模式指示器、语音指示器 |
| **messages/** | 消息渲染：助手文本/思考/工具使用、用户消息、系统消息 |
| **permissions/** | 权限对话框 (每种工具的独立对话框 + 通用框架) |
| **design-system/** | 设计系统：ThemedBox/ThemedText、Dialog、Pane、ProgressBar、Tabs |
| **agents/** | Agent 编辑器：列表、详情、模型选择、工具选择、验证 |
| **mcp/** | MCP 服务器配置 UI |
| **tasks/** | 后台任务管理 UI |
| **LogoV2/** | 启动 Logo、欢迎界面、通知 |
| **FeedbackSurvey/** | 交互后调查问卷 |
| **Settings/** | 设置面板 (Config/Status/Usage) |
| **Spinner/** | 旋转指示动画帧系统 |
| **HelpV2/** | 帮助系统 |
| **diff/** | 差异显示 (DiffDetailView/DiffDialog) |
| **shell/** | Shell 输出渲染 |
| **wizard/** | 多步向导框架 (WizardProvider) |
| **skills/** | Skills 菜单 |
| **teams/** | 团队状态显示 |
| **memory/** | 记忆文件选择器 |
| **sandbox/** | 沙箱设置面板 |
| **ui/** | 基础 UI 组件 (OrderedList/TreeSelect) |
| **TrustDialog/** | 信任对话框 |
| **StructuredDiff/** | 结构化差异显示 |

## 三、Hooks 体系

**目录**: `src/hooks/`

### Hooks 分类

**输入与交互**:
`useTextInput.ts`, `useVimInput.ts`, `useInputBuffer.ts`, `useArrowKeyHistory.tsx`, `useSearchInput.ts`, `usePasteHandler.ts`, `useCommandKeybindings.tsx`, `useGlobalKeybindings.tsx`, `useCopyOnSelect.ts`, `useTerminalSize.ts`

**工具与权限**:
`useCanUseTool.tsx`, `useMergedTools.ts`, `useMergedCommands.ts`, `useMergedClients.ts`

**会话与状态**:
`useRemoteSession.ts`, `useAssistantHistory.ts`, `useSessionBackgrounding.ts`, `useCancelRequest.ts`

**命令与任务**:
`useCommandQueue.ts`, `useAwaySummary.ts`, `useTasksV2.ts`, `useTaskListWatcher.ts`

**IDE 集成**:
`useIDEIntegration.tsx`, `useIdeAtMentioned.ts`, `useIdeSelection.ts`

**插件与配置**:
`useSettings.ts`, `useManagePlugins.ts`, `usePluginRecommendationBase.tsx`

### toolPermission Hooks

**目录**: `src/hooks/toolPermission/`

| 模块 | 作用 |
|------|------|
| **PermissionContext.ts** | 冻结的权限上下文，提供决策日志、hook 执行、队列操作 |
| **interactiveHandler.ts** | 主 Agent 交互式权限流 (推入确认队列 + 后台自动化检查) |
| **coordinatorHandler.ts** | 协调器 Worker 权限流 (顺序执行自动化检查) |
| **swarmWorkerHandler.ts** | Swarm Worker 权限流 (转发给领导决策) |
| **permissionLogging.ts** | 权限决策集中日志 (Statsig + OTel + 代码编辑指标) |

### 通知 Hooks

**目录**: `src/hooks/notifs/`

16 个通知 hook：自动模式不可用、订阅切换、弃用警告、快速模式、IDE 状态指示器、安装消息、LSP 初始化、MCP 连接、模型迁移、npm 弃用、插件自动更新、插件安装、速率限制、设置错误、启动、队友关闭。

## 四、屏幕 Screens

| 屏幕 | 文件 | 作用 |
|------|------|------|
| **REPL** | `screens/REPL.tsx` | 主聊天屏幕 (消息列表、输入框、权限对话框、通知) |
| **Doctor** | `screens/Doctor.tsx` | `/doctor` 诊断屏幕 |
| **ResumeConversation** | `screens/ResumeConversation.tsx` | 会话恢复/历史浏览 |

## 五、状态管理

**目录**: `src/state/`

| 文件 | 作用 |
|------|------|
| `AppStateStore.ts` | 核心 store 定义 (AppState 接口 + getDefaultAppState) |
| `AppState.tsx` | React Provider (AppStateProvider) + useAppState/useSetAppState |
| `store.ts` | 自定义 store 实现 (subscribe/getState/setState) |
| `selectors.ts` | 派生状态选择器 |

## 六、上下文 Context

**目录**: `src/context/`

| 文件 | 作用 |
|------|------|
| `notifications.tsx` | 通知系统上下文 |
| `modalContext.tsx` | 模态对话框上下文 |
| `overlayContext.tsx` | 覆盖层渲染上下文 |
| `mailbox.tsx` | 邮件箱消息传递 |
| `voice.tsx` | 语音模式上下文 |
| `QueuedMessageContext.tsx` | 消息队列处理上下文 |
