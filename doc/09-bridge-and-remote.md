# 桥接系统与远程连接

## 一、Bridge System (远程控制)

**目录**: `src/bridge/` (31 个文件)

Bridge 系统实现了"远程控制 (Remote Control)"功能，使本地 Claude Code CLI 能够连接到云端托管的会话。

### 架构概览

```
本地 CLI (REPL/守护进程)
    │
    ├── Bridge (src/bridge/)
    │   ├── 环境注册
    │   ├── 工作轮询
    │   ├── 消息路由
    │   └── 会话管理
    │
    ├── Remote (src/remote/)
    │   ├── WebSocket 连接
    │   └── 消息适配
    │
    └── Server (src/server/)
        └── Direct Connect (本地 WebSocket)
```

### 两种模式

| 模式 | 入口 | 传输 |
|------|------|------|
| **REPL 桥接** | `initReplBridge.ts` → `replBridge.ts` | 环境 API / CCR v2 |
| **守护进程桥接** | `bridgeMain.ts` | 环境 API (工作池模式) |

### 核心文件

| 文件 | 行数 | 作用 |
|------|------|------|
| `bridgeMain.ts` | ~115KB | 守护进程桥接主入口：环境注册、工作轮询、会话生成 |
| `replBridge.ts` | ~100KB | REPL 桥接核心：环境注册、工作轮询、消息发送 |
| `remoteBridgeCore.ts` | ~39KB | "无环境"桥接核心 (直连 OAuth + WebSocket) |
| `initReplBridge.ts` | ~24KB | REPL 初始化包装 |
| `bridgeApi.ts` | ~18KB | HTTP 客户端 (Axios) |
| `bridgeMessaging.ts` | ~16KB | 共享消息处理 |
| `replBridgeTransport.ts` | ~16KB | 传输层抽象 (SSE/WebSocket) |
| `types.ts` | ~10KB | 核心类型定义 |
| `bridgeUI.ts` | ~17KB | 守护进程 UI 显示 |
| `createSession.ts` | ~12KB | 桥接会话创建 |
| `bridgeEnabled.ts` | ~8KB | 权限门控 |
| `jwtUtils.ts` | ~9KB | JWT token 生命周期 |
| `sessionRunner.ts` | ~18KB | 会话生成逻辑 |
| `pollConfig.ts` | | 轮询间隔配置 |

### 桥接生命周期

```
1. 初始化: initReplBridge()
   → 读取 bootstrap 状态 (cwd, sessionId, git, OAuth)
   → 委托到 initBridgeCore()

2. 注册: POST /environments 
   → 注册到 CCR 环境
   
3. 轮询: GET /poll
   → 获取待处理工作 (WorkData)
   → 包含 work_id, session_id, commands
   
4. 处理: 收到工作
   → 回应工作 (POST /ack)
   → 建立会话 (sessionRunner.ts)
   → 执行工作
   
5. 通信:
   → 发送消息: POST /messages
   → 接收消息: SSE / WebSocket
   → 心跳: POST /heartbeat
   
6. 完成:
   → 发送结果
   → 停止
   → 取消注册
```

### 传输方式

| 版本 | 传输 | 描述 |
|------|------|------|
| v1 | 环境 API | SSE + HTTP 轮询 |
| v2 | CCR v2 | WebSocket + SSE (CCRClient) |

v2 使用 `remoteBridgeCore.ts`，绕过环境 API，直接通过 OAuth 连接：
```
POST /sessions → POST /sessions/{id}/bridge → 获取 JWT → SSE/WebSocket
```

## 二、Remote Session (远程会话)

**目录**: `src/remote/` (4 个文件)

将本地 CLI 连接到远程 CCR 容器中的会话。

| 文件 | 作用 |
|------|------|
| `RemoteSessionManager.ts` (~9KB) | 高层管理器：包装 SessionsWebSocket，消息分发 |
| `SessionsWebSocket.ts` (~13KB) | WebSocket 传输层：重连、ping/pong、消息解析 |
| `sdkMessageAdapter.ts` (~9KB) | SDK 消息 ←→ 内部消息类型转换 |
| `remotePermissionBridge.ts` (~2KB) | 合成权限请求的 AssistantMessage |

### 连接生命周期

```
1. 连接: WebSocket → CCR session-ingress
2. 保持: 30s ping/pong 心跳
3. 重连: 2s 延迟，最多 5 次
4. 永久关闭: 4003 (未授权) 不再重连
5. 临时重试: 4001 (会话未找到，compact 期间)
```

## 三、Direct Connect Server

**目录**: `src/server/` (3 个文件)

运行本地 WebSocket 服务器，外部客户端可以直连（不经过 CCR）。

| 文件 | 作用 |
|------|------|
| `directConnectManager.ts` (~6KB) | DirectConnectSessionManager |
| `createDirectConnectSession.ts` (~2KB) | 创建直连会话 |
| `types.ts` | Zod schema + ServerConfig/SessionState |

**功能**:
- 本地端口监听
- UNIX socket 支持
- 空闲超时
- 会话持久化 (`~/.claude/server-sessions.json`)

## 四、Upstream Proxy

**目录**: `src/upstreamproxy/` (2 个文件)

在 CCR 容器中实现本地 CONNECT 代理，用于路由出站流量通过 CCR 基础设施。

| 文件 | 作用 |
|------|------|
| `upstreamproxy.ts` (~10KB) | 初始化：读取 token、设置 ptrace 保护、启动 relay |
| `relay.ts` (~15KB) | TCP 服务器：接受 CONNECT → WebSocket 隧道 |

**功能**:
- 容器出站流量 MITM
- 组织凭据注入
- NO_PROXY 配置 (loopback/RFC1918/API/GitHub/包注册表)

## 五、网络传输层

**目录**: `src/cli/transports/`

| 文件 | 用途 |
|------|------|
| `HybridTransport.ts` | 混合传输 |
| `SSETransport.ts` | Server-Sent Events |
| `WebSocketTransport.ts` | WebSocket |
| `ccrClient.ts` | CCR 客户端 |
| `SerialBatchEventUploader.ts` | 串行批量事件上传 |
| `WorkerStateUploader.ts` | 工作进程状态上传 |

## 六、消息适配

从 CCR 到 REPL 的消息转换：

```
SDK Message (CCR)                    REPL Message (内部)
─────────────────                    ──────────────────
SDKAssistantMessage      →           AssistantMessage
SDKPartialAssistantMessage →         StreamEvent
SDKResultMessage         →           SystemMessage
SDKStatusMessage         →           SystemMessage
SDKToolProgressMessage   →           (工具进度)
SDKCompactBoundaryMessage →          CompactBoundaryMessage
```

## 七、安全考虑

| 机制 | 文件 |
|------|------|
| JWT token 刷新 | `bridge/jwtUtils.ts` |
| 可信设备 token | `bridge/trustedDevice.ts` |
| 权限终止开关 | `utils/permissions/bypassPermissionsKillswitch.ts` |
| SSRF 保护 | `utils/hooks/ssrfGuard.ts` |
| mTLS 配置 | `utils/mtls.ts` |
| 代理配置 | `utils/proxy.ts` |
| CA 证书 | `utils/caCerts.ts` |
