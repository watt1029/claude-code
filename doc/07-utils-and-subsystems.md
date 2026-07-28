# 工具函数与子系统

`src/utils/` 是项目中最大的目录，包含超过 200 个文件，按功能分类到各自的子目录中。

## 一、配置系统

### GlobalConfig / ProjectConfig

**文件**: `src/utils/config.ts`

两级配置：
- **GlobalConfig** (用户级，`~/.claude.json`)
- **ProjectConfig** (项目级，keyed by git root)

**特点**:
- 写穿透缓存 (write-through cache)
- 文件锁防止并发写
- 时间戳备份 (最多 5 个)
- 损坏数据恢复 (备份到 `~/.claude/backups/`)
- `GLOBAL_CONFIG_KEYS` (67 个) + `PROJECT_CONFIG_KEYS` (3 个)

## 二、设置系统

**目录**: `src/utils/settings/` (19+ 文件)

### 多源合并 (优先级递增)

```
userSettings   (~/.claude/settings.json)
    → projectSettings (.claude/settings.json)
    → localSettings  (.claude/settings.local.json, gitignored)
    → flagSettings   (--settings CLI flag)
    → policySettings (企业托管设置)
```

### MDM 集成

**目录**: `src/utils/settings/mdm/`

Windows 注册表 (HKCU) 和 MDM 策略加载器，用于企业托管部署。

### 核心文件

| 文件 | 作用 |
|------|------|
| `settings.ts` | 设置加载、合并、持久化 |
| `types.ts` | Zod Schema 定义 |
| `validation.ts` | 设置验证 + 错误格式化 |
| `changeDetector.ts` | 文件变更检测 |
| `constants.ts` | 设置来源定义 |

## 三、Hooks 子系统

**目录**: `src/utils/hooks/` (18 个文件)

### Hook 类型

| 类型 | 执行器 |
|------|--------|
| **Agent Hook** | `execAgentHook.ts` |
| **HTTP Hook** | `execHttpHook.ts` (+ SSRF 保护 `ssrfGuard.ts`) |
| **Prompt Hook** | `execPromptHook.ts` |

### Hook 事件

`hookEvents.ts` 提供通用事件广播系统：
- 始终发射: `SessionStart`, `Setup`
- 可选: `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `Notification`, `Stop`, `SubagentStart`

### 关键文件

| 文件 | 作用 |
|------|------|
| `sessionHooks.ts` | 会话生命周期钩子 |
| `hooksSettings.ts` | 从设置读取 hook 配置 |
| `hooksConfigManager.ts` | Hook 事件元数据 |
| `hooksConfigSnapshot.ts` | 配置快照 (防止隐藏修改) |
| `AsyncHookRegistry.ts` | 异步 hook 注册/分发 |
| `fileChangedWatcher.ts` | 文件变更监听 |
| `postSamplingHooks.ts` | 采样后 hook 处理 |
| `apiQueryHookHelper.ts` | API 查询 hook 辅助 |
| `registerFrontmatterHooks.ts` | 从 CLAUDE.md 注册 hooks |
| `registerSkillHooks.ts` | 从 skills 定义注册 hooks |

## 四、权限系统 (详见 08-permissions)

## 五、Bash 子系统

**目录**: `src/utils/bash/` (17+ 文件)

基于 **tree-sitter** 的完整 Bash 语法解析器和安全分析器。

| 文件 | 行数 | 作用 |
|------|------|------|
| `bashParser.ts` | ~130KB | 底层 Bash 解析器 (WASM/NAPI) |
| `ast.ts` | ~112KB | AST 节点定义 + 安全遍历器 |
| `commands.ts` | ~51KB | 命令类型分类 (安全/危险/eval) |
| `parser.ts` | | 高层解析 API (环境变量提取、命令定位) |
| `heredoc.ts` | ~31KB | Heredoc 解析 |
| `ShellSnapshot.ts` | | Shell 状态快照 (别名、函数、环境变量) |

**特性**:
- 50ms 超时 / 50K 节点上限
- `PARSE_ABORTED` 哨兵值 (对抗性输入保护)
- 命令分类用于权限系统
- 门控标记: `TREE_SITTER_BASH`, `TREE_SITTER_BASH_SHADOW`

## 六、模型子系统

**目录**: `src/utils/model/` (15+ 文件)

| 文件 | 作用 |
|------|------|
| `model.ts` | 核心模型解析 (多源: session override → --model → env → settings) |
| `modelStrings.ts` | 模型名称/标识符 |
| `modelOptions.ts` | 模型选择器选项 |
| `modelAllowlist.ts` | 模型白名单 |
| `modelCapabilities.ts` | 模型能力检测 (推理等) |
| `providers.ts` | API 提供商检测 (Anthropic/Bedrock/Vertex) |
| `aliases.ts` | 模型别名映射 (sonnet → sonnet-4-6) |
| `deprecation.ts` | 模型弃用处理 |
| `bedrock.ts` | AWS Bedrock 支持 |

## 七、Git 子系统

**目录**: `src/utils/git/` (3 个文件)

从文件系统直接读取 Git 状态，无需 `git` 子进程：

| 文件 | 作用 |
|------|------|
| `gitFilesystem.ts` | 读取 HEAD、引用 (松散/packed)、工作树检测 |
| `gitConfigParser.ts` | Git 配置解析 (INI-like) |
| `gitignore.ts` | `.gitignore` 操作 |

## 八、遥测系统

**目录**: `src/utils/telemetry/` (9 个文件)

基于 **OpenTelemetry** 构建：

| 文件 | 作用 |
|------|------|
| `instrumentation.ts` | OTel SDK 初始化 (动态导入导出器，避免加载 6 种协议 ~1.2MB) |
| `sessionTracing.ts` | 会话级追踪 |
| `perfettoTracing.ts` | Perfetto 追踪格式导出 |
| `betaSessionTracing.ts` | 测试性追踪特性 |
| `bigqueryExporter.ts` | BigQuery 日志导出 |
| `events.ts` | 事件定义 |
| `logger.ts` | OTel 日志配置 |

## 九、插件系统 (详见 10-plugin-and-skills)

## 十、记忆系统

**目录**: `src/memdir/` + `src/utils/memory/`

| 模块 | 作用 |
|------|------|
| `memdir/memdir.ts` | 核心记忆系统：构建记忆提示、读取 MEMORY.md |
| `memdir/memoryTypes.ts` | 记忆类型分类 (user/feedback/project/reference) |
| `memdir/paths.ts` | 记忆路径解析 |
| `memdir/findRelevantMemories.ts` | 基于前文选择最相关记忆 |
| `memdir/memoryScan.ts` | 记忆文件扫描 |
| `memdir/memoryAge.ts` | 记忆时效性检查 |
| `utils/memory/types.ts` | 记忆类型定义 (User/Project/Local/Managed/AutoMem/TeamMem) |

## 十一、原生 TypeScript 模块

**目录**: `src/native-ts/` (之前是 Rust NAPI 模块，纯 TS 重写)

| 模块 | 文件 | 作用 |
|------|------|------|
| `color-diff/index.ts` | ~1000 行 | 语法高亮差异显示 (highlight.js) |
| `file-index/index.ts` | ~370 行 | 模糊文件搜索 (nucleo 风格评分) |
| `yoga-layout/` | ~2650 行 | Yoga Flexbox 布局引擎纯 TS 实现 |

## 十二、Vim 模式

**目录**: `src/vim/` (5 个文件)

完整的类 Vim 编辑模式，作为纯函数状态机实现：

| 文件 | 行数 | 作用 |
|------|------|------|
| `types.ts` | ~6KB | 状态机类型 (INSERT/NORMAL/CommandState) |
| `transitions.ts` | ~12KB | 状态转移表 |
| `motions.ts` | ~2KB | 纯移动函数 |
| `operators.ts` | ~16KB | 操作符执行 (d/c/y/x/p/r) |
| `textObjects.ts` | ~5KB | 文本对象 (iw/aw/it/at/ip/ap) |

## 十三、Schema 定义

**文件**: `src/schemas/hooks.ts` (~8KB)

提取的 Zod schemas，用于中断循环依赖：
- `BashCommandHookSchema`
- `PromptHookSchema`
- `HttpHookSchema`
- `AgentHookSchema`

## 十四、数据迁移

**文件**: `src/migrations/` (11 个文件)

一次性幂等迁移函数，在启动时运行：

| 迁移 | 作用 |
|------|------|
| `migrateFennecToOpus.ts` | 模型名称迁移 |
| `migrateOpusToOpus1m.ts` | Opus → Opus 1M |
| `migrateSonnet45ToSonnet46.ts` | Sonnet 4.5 → 'sonnet' 别名 |
| `migrateAutoUpdatesToSettings.ts` | 自动更新迁移 |
| 等 | ... |

## 十五、Bootstrap 全局状态

**文件**: `src/bootstrap/state.ts` (~56KB)

**所有**进程状态通过此文件管理。它是启动时初始化的全局状态源。文件首行警告："DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE."

主要状态：
- 工作目录 / 项目根
- 使用量追踪 (cost, duration, tokents)
- 模型配置
- 交互模式标志
- OpenTelemetry providers
- 设置缓存和 hook matcher

## 十六、类型系统

**目录**: `src/types/`

| 文件 | 作用 |
|------|------|
| `command.ts` | 命令类型 (PromptCommand/LocalCommand/LocalJSXCommandContext) |
| `hooks.ts` | Hook 类型系统 |
| `permissions.ts` | 权限模式类型 |
| `plugin.ts` | 插件类型系统 |
| `logs.ts` | 日志类型 |
| `ids.ts` | 会话 ID 类型 |
| `generated/` | Protobuf 生成的类型 (事件、认证、GrowthBook) |
