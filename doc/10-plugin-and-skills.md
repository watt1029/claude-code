# 插件系统与 Skills

## 一、插件系统架构

### 发现来源 (优先级)

```
1. 市场插件 (marketplace@name 格式)
2. 内置插件 (builtin 市场)
3. 官方市场 (official marketplace)
4. 会话插件 (--plugin-dir CLI flag / SDK options)
5. 捆绑 Skills (自动加载)
```

### 插件目录结构

```
my-plugin/
├── plugin.json          # 可选清单 (元数据)
├── commands/            # 自定义斜杠命令
│   ├── build.md
│   └── deploy.md
├── agents/              # 自定义 AI Agent
│   └── test-runner.md
└── hooks/               # Hook 配置
    └── hooks.json
```

## 二、插件加载器

**目录**: `src/utils/plugins/` (47 个文件)

### 核心加载器

| 文件 | 行数 | 作用 |
|------|------|------|
| `pluginLoader.ts` | ~110KB | 插件发现、验证、加载（主入口） |
| `marketplaceManager.ts` | ~93KB | 市场管理 (搜索/安装/卸载/更新) |
| `schemas.ts` | ~58KB | 插件和市场 Zod Schema |
| `installedPluginsManager.ts` | ~41KB | 已安装插件生命周期管理 |
| `validatePlugin.ts` | ~28KB | 插件完整性验证 |
| `dependencyResolver.ts` | | 插件依赖解析 |
| `pluginAutoupdate.ts` | | 自动更新 |
| `pluginBlocklist.ts` | | 黑名单管理 |
| `pluginVersioning.ts` | | 版本管理 |
| `pluginOptionsStorage.ts` | | 插件选项持久化 |

### 组件加载器

| 文件 | 作用 |
|------|------|
| `loadPluginCommands.ts` | 加载插件中的命令 |
| `loadPluginHooks.ts` | 加载插件中的 hooks |
| `loadPluginAgents.ts` | 加载插件中的 agents |
| `loadPluginOutputStyles.ts` | 加载插件中的输出样式 |
| `mcpbHandler.ts` | MCP Bundle 处理器 |
| `mcpPluginIntegration.ts` | MCP/插件集成 |

### 市场集成

| 文件 | 作用 |
|------|------|
| `officialMarketplace.ts` | 官方市场 |
| `officialMarketplaceGcs.ts` | GCS 上的官方市场 |
| `officialMarketplaceStartupCheck.ts` | 市场启动检查 |
| `marketplaceHelpers.ts` | 市场辅助函数 |
| `zipCache.ts` / `zipCacheAdapters.ts` | 压缩包缓存 |

## 三、内置插件 vs 捆绑 Skills

```
内置插件 (Builtin Plugin)         捆绑 Skills (Bundled Skill)
─────────────────────────        ─────────────────────────
用户可启用/禁用                   自动加载
出现在 /plugin UI 中             不出现
有 plugin.json 清单              无独立清单
persisted to user settings       运行时加载
```

**文件**: `src/plugins/builtinPlugins.ts` — 内置插件注册表
**文件**: `src/plugins/bundled/index.ts` — 当前为空 (脚手架)

## 四、Skills 系统

### 技能注册与调度

**文件**: `src/skills/` 与 `src/commands.ts`

Skills 是一种特殊的命令类型，加载后注册到命令系统中：

```
技能发现路径:
1. 捆绑 skills (src/skills/bundled/)
2. 文件系统 skills 目录 (.claude/skills/)
3. 插件 skills
4. MCP-provided skills
```

### 技能变更检测

**文件**: `src/utils/skills/skillChangeDetector.ts`

使用 `chokidar` 监听技能文件变更：

```typescript
// 监听 → 检测变更 → 延迟稳定性 → 去抖 → 清除缓存 → 通知重载
```

**特性**:
- 稳定性延迟: 1000ms (等待文件写入完成)
- 去抖延迟: 300ms (防止级联重载)
- 轮询模式: 2s (Bun 下有 fs.watch 死锁问题)

### Skills 作为命令

**文件**: `src/commands.ts`

```typescript
getCommands(cwd) → 从多个来源加载命令:
  bundled skills
  → builtin plugin skills
  → skill directory commands
  → workflow commands
  → plugin commands
  → plugin skills
  → built-in commands
```

### Skills 作为工具

**文件**: `src/tools/SkillTool/SkillTool.ts`

Skills 也可通过 `SkillTool` 作为模型可调用的工具暴露：

```typescript
// 模型通过 SkillTool 调用技能
SkillTool.call() → findCommand() → load command → execute
```

## 五、配置与设置

### 插件设置管理

**目录**: `src/services/plugins/`

| 文件 | 作用 |
|------|------|
| `PluginInstallationManager.ts` | 安装管理器 |
| `pluginCliCommands.ts` | CLI 子命令 (`claude plugin *`) |
| `pluginOperations.ts` | 插件操作 |

### CLI 插件子命令

```
claude plugin install      安装插件
claude plugin uninstall    卸载插件
claude plugin list         列出已安装插件
claude plugin search       搜索市场
claude plugin update       更新插件
claude plugin enable/disable   启用/禁用
claude plugin validate     验证插件
claude plugin marketplace *    市场管理
```

### 设置项

插件在 `settings.json` 中的配置：
```json
{
  "plugins": {
    "marketplaces": ["..."],
    "enabled": ["plugin1", "plugin2"]
  }
}
```

## 六、命令系统补充

**目录**: `src/commands/` (80+ 命令)

### 命令类型

```typescript
type Command = 
  | { type: 'local' }       // 进程内执行，返回文本
  | { type: 'local-jsx' }   // 渲染 React UI
  | { type: 'prompt' }      // 扩展为模型内容
```

### 安全命令集

```typescript
REMOTE_SAFE_COMMANDS: 远程模式安全命令
BRIDGE_SAFE_COMMANDS: 桥接模式安全命令
isBridgeSafeCommand(): 运行时检查
```

### 命令命名空间

按功能分类的命令目录：
- `commands/agents/` — Agent 管理
- `commands/mcp/` — MCP 管理
- `commands/config/` — 配置管理
- `commands/bridge/` — 桥接命令
- `commands/branch/` — 分支管理
- `commands/doctor/` — 诊断
- `commands/help/` — 帮助
- `commands/session/` — 会话管理
- `commands/review/` — 代码审查
- `commands/plan/` — 计划模式
- `commands/hooks/` — Hook 管理
- `commands/plugins/` — 插件管理
- `commands/skills/` — Skills 管理
- `commands/clear/` — 清除对话
- `commands/cost/` — 费用查询
- `commands/status/` — 状态查看
- `commands/copy/` — 复制
- `commands/diff/` — 差异查看
- `commands/export/` — 导出
- `commands/memory/` — 记忆管理
- `commands/effort/` — 努力程度设置
- `commands/model/` — 模型切换
- `commands/voice/` — 语音模式
- `commands/vim/` — Vim 模式切换
- 等 80+ 个命令
