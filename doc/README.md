# Claude Code 源码架构文档

> 本文档通过走读 [Claude Code](https://github.com/anthropics/claude-code) 开源项目的 `src/` 源码，
> 梳理其整体架构、模块划分和关键设计模式，供学习 code agent 架构设计参考。

## 目录

| 章节 | 内容 | 核心源码目录 |
|------|------|-------------|
| [01-arch-overview](01-arch-overview.md) | 整体架构总览、分层设计、数据流 | `src/` 全局 |
| [02-entrypoints-and-cli](02-entrypoints-and-cli.md) | 入口点、初始化启动流程、CLI 子命令系统 | `src/main.tsx`, `src/entrypoints/`, `src/cli/`, `src/setup.ts` |
| [03-query-engine-and-tools](03-query-engine-and-tools.md) | 查询引擎、工具(Tool)系统、工具编排 | `src/QueryEngine.ts`, `src/Tool.ts`, `src/tools/`, `src/services/tools/` |
| [04-tasks-and-agents](04-tasks-and-agents.md) | 任务抽象、Agent 执行、后台任务 | `src/Task.ts`, `src/tasks/`, `src/utils/swarm/`, `src/coordinator/` |
| [05-ui-and-components](05-ui-and-components.md) | Ink 终端渲染引擎、React 组件体系 | `src/ink/`, `src/components/`, `src/hooks/`, `src/screens/` |
| [06-services-layer](06-services-layer.md) | 服务层：API、MCP、Analytics、LSP 等 | `src/services/` |
| [07-utils-and-subsystems](07-utils-and-subsystems.md) | 工具函数和子系统概览 | `src/utils/` |
| [08-permissions-and-security](08-permissions-and-security.md) | 权限系统、沙箱、安全机制 | `src/utils/permissions/`, `src/hooks/toolPermission/`, `src/utils/sandbox/`, `src/utils/bash/` |
| [09-bridge-and-remote](09-bridge-and-remote.md) | 桥接模式、远程控制、CCR 集成 | `src/bridge/`, `src/remote/`, `src/server/`, `src/upstreamproxy/` |
| [10-plugin-and-skills](10-plugin-and-skills.md) | 插件系统、Skills、命令扩展 | `src/plugins/`, `src/utils/plugins/`, `src/skills/`, `src/commands/`, `src/commands.ts` |

## 项目概览

Claude Code 是一个 **基于终端的 AI 编程助手 (Code Agent)**，使用户能在终端中通过自然语言与 AI 协作完成软件开发任务。

### 技术栈

| 层面 | 技术选型 |
|------|---------|
| 运行时 | **Bun** (JavaScript/TypeScript 运行时 + 打包工具) |
| 语言 | TypeScript (严格模式) |
| 终端 UI | **Ink** (基于 React 的终端渲染框架) + 自定义 reconciler |
| 布局引擎 | **Yoga** (Flexbox 布局，自包含 TypeScript 实现) |
| API | **Anthropic Claude API** (Messages API) |
| 数据验证 | **Zod v4** |
| 遥测 | **OpenTelemetry** (OTLP/HTTP/gRPC/Console) |
| 特征标记 | **GrowthBook** |
| 认证 | **OAuth 2.0** + API Key |
| 构建 | Bun 内置打包 + `bun:bundle` 编译时特征标记 |
