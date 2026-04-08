## jiuwenclaw 集成分析

### 架构概览

jiuwenclaw 以 **sidecar（侧车）模式** 集成到 clowder-ai，通过 WebSocket 协议通信：

```
clowder-ai (API) ──WebSocket──> jiuwenclaw sidecar (Python)
     │                              │
     │                              ├─ AgentWebSocketServer (ws://127.0.0.1:AGENT_PORT)
     │                              ├─ WebChannel (内置 Web UI)
     │                              └─ openjiuwen agent 核心引擎
     │
     └─ RelayClawAgentService (TypeScript)
```

### 核心集成点

| 模块 | 文件路径 | 功能 |
|------|----------|------|
| **路径解析** | `packages/api/src/utils/jiuwenclaw-paths.ts:16-66` | 定位 vendored 目录、Python 虚拟环境 |
| **Sidecar 启动** | `packages/api/src/domains/cats/services/agents/providers/relayclaw-sidecar.ts:168` | `spawn(python, ['-m', 'jiuwenclaw.app'])` |
| **WebSocket 连接** | `packages/api/src/domains/cats/services/agents/providers/relayclaw-connection.ts` | 管理 WS 帧队列和连接 |
| **消息转换** | `packages/api/src/domains/cats/services/agents/providers/relayclaw-event-transform.ts` | 转换 jiuwenclaw 事件为 AgentMessage |
| **Agent 服务** | `packages/api/src/domains/cats/services/agents/providers/RelayClawAgentService.ts` | 顶层编排层 |
| **类型定义** | `packages/shared/src/types/relayclaw.ts` | WS 帧和事件类型 |

### 启动流程

1. **`RelayClawAgentService.invoke()`** 调用时，如果 `config.autoStart=true`
2. **`DefaultRelayClawSidecarController.ensureStarted()`** 分配端口、构建环境
3. **`spawn(python, ['-m', 'jiuwenclaw.app'])`** 启动 Python 进程
4. **TCP Probe** 等待 `AGENT_PORT` 就绪（检测日志含 `初始化完成`）
5. **WebSocket 连接** 建立 `ws://127.0.0.1:{agentPort}` 连接
6. **发送请求帧** `{req_method: 'chat.send', params: {...}, is_stream: true}`
7. **流式消费帧** 转换 `chat.delta/chat.final` 等事件为 `AgentMessage`

### 环境注入

Sidecar 启动时注入关键环境变量（`relayclaw-sidecar.ts:117-142`）：
- `API_KEY` / `API_BASE` / `MODEL_NAME` — LLM 配置
- `JIUWENCLAW_AGENT_ROOT` — agent 工作目录
- `JIUWENCLAW_PROJECT_DIR` — 用户项目目录
- `CAT_CAFE_MCP_*` — MCP server 配置

### jiuwenclaw 项目结构

位于 `vendor/jiuwenclaw/`：
- `jiuwenclaw/app.py` — 主入口，启动 WebSocket + ChannelManager
- `jiuwenclaw/agentserver/agent_ws_server.py` — WebSocket 服务端
- `jiuwenclaw/channel/` — 多渠道（飞书、钉钉、Telegram 等）
- `jiuwenclaw/agentserver/tools/` — 工具集（浏览器、搜索、MCP 等）

---

## jiuwenclaw 启动流程分析

### 完整启动流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    clowder-ai API (TypeScript)                              │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. RelayClawAgentService.invoke()                                           │
│    └─> ensureConnected(signal, options)                                     │
│         └─> sidecar.ensureStarted(options, signal)  ← config.autoStart=true │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│              relayclaw-sidecar.ts (DefaultRelayClawSidecarController)       │
├─────────────────────────────────────────────────────────────────────────────┤
│ 2. buildRuntime(options)                                                    │
│    ├─> resolveJiuwenClawAppDir()     → vendor/jiuwenclaw                    │
│    ├─> resolveJiuwenClawPythonBin()  → .venv/Scripts/python.exe             │
│    └─> 构建 env: { API_KEY, API_BASE, MODEL_NAME, HOME, ... }               │
│                                                                              │
│ 3. allocatePort() → 动态分配 AGENT_PORT + WEB_PORT                           │
│                                                                              │
│ 4. spawn(python, ['-m', 'jiuwenclaw.app'], { cwd, env })                    │
│    │                                                                         │
│    │  环境变量:                                                              │
│    │  ├─ AGENT_PORT=随机端口                                                 │
│    │  ├─ WEB_PORT=随机端口                                                   │
│    │  ├─ API_KEY / API_BASE / MODEL_NAME                                     │
│    │  ├─ HOME=.cat-cafe/relayclaw/{catId}                                    │
│    │  └─ CAT_CAFE_MCP_* (MCP 配置)                                           │
│                                                                              │
│ 5. tcpProbe('127.0.0.1', agentPort) + isSidecarReady(logs)                  │
│    等待日志包含: "[JiuWenClaw] 初始化完成" 或 "WebChannel 已启动"             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    jiuwenclaw/app.py (Python _run)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│ 6. 环境读取:                                                                 │
│    ├─ agent_port = os.getenv("AGENT_PORT", "18092")                         │
│    ├─ web_host = os.getenv("WEB_HOST", "127.0.0.1")                         │
│    └─ web_port = os.getenv("WEB_PORT", "19000")                             │
│                                                                              │
│ 7. 创建核心组件:                                                             │
│    ├─ agent = JiuWenClaw()                          # Agent 实例             │
│    │   └─> await agent.create_instance()            # 初始化 ReActAgent    │
│    │        ├─ 加载 config.yaml 配置                                         │
│    │        ├─ 配置 LLM (OpenAI/OpenRouter)                                  │
│    │        ├─ 注册 tools (memory, browser, mcp, todo...)                   │
│    │        └─ 初始化权限引擎                                                │
│    │                                                                         │
│    ├─ server = AgentWebSocketServer(agent, port=agent_port)                 │
│    │   └─> await server.start()                      # 启动 WS 服务端       │
│    │                                                                         │
│    ├─ client = WebSocketAgentServerClient()                                 │
│    │   └─> await client.connect(uri)                 # 连接到 AgentServer  │
│    │                                                                         │
│    ├─ message_handler = MessageHandler(client)                              │
│    │   └─> await message_handler.start_forwarding()  # 启动消息转发循环     │
│    │                                                                         │
│    ├─ channel_manager = ChannelManager(message_handler)                     │
│    ├─ heartbeat_service = GatewayHeartbeatService(...)                      │
│    └─ web_channel = WebChannel(config)                                      │
│                                                                              │
│ 8. 注册 Web handlers (config.get/set, chat.send, skills.*, cron.*, ...)     │
│                                                                              │
│ 9. await web_channel.start()                        # 启动 WebChannel      │
│    └─> 打印 "[JiuWenClaw] 初始化完成"                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                   relayclaw-connection.ts (RelayClawConnectionManager)      │
├─────────────────────────────────────────────────────────────────────────────┤
│ 10. connect(url = ws://127.0.0.1:{agentPort})                               │
│     ├─> new WebSocket(url)                                                  │
│     └─> 等待 connection.ack 事件                                            │
│                                                                              │
│ 11. 消息路由:                                                                │
│     ├─> frame.request_id → requestQueues.get(request_id)                   │
│     └─> queue.put(frame)                                                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 关键组件职责

| 组件 | 文件 | 职责 |
|------|------|------|
| **RelayClawAgentService** | `RelayClawAgentService.ts` | 顶层编排：sidecar 启动、WS 连接、请求发送、帧消费 |
| **DefaultRelayClawSidecarController** | `relayclaw-sidecar.ts:37-216` | Python 进程生命周期管理、端口分配、环境构建 |
| **RelayClawConnectionManager** | `relayclaw-connection.ts:47-161` | WebSocket 连接管理、帧队列路由 |
| **FrameQueue** | `relayclaw-connection.ts:3-29` | 异步队列：生产者-消费者模式 |
| **JiuWenClaw** | `jiuwenclaw/agentserver/interface.py:142` | Python Agent 核心：ReActAgent 初始化、请求处理 |
| **AgentWebSocketServer** | `jiuwenclaw/agentserver/agent_ws_server.py:48` | Python WS 服务端：处理 AgentRequest |
| **MessageHandler** | `jiuwenclaw/gateway/message_handler.py:34` | 消息转发：user_messages → AgentServer → robot_messages |
| **WebSocketAgentServerClient** | `jiuwenclaw/gateway/agent_client.py:81` | Python 内部 WS 客户端：Gateway ↔ AgentServer 通信 |

### 请求处理流程

```
用户请求 → RelayClawAgentService.invoke()
    │
    ├─> buildRequest() → { req_method: 'chat.send', params: { query, ... }, is_stream: true }
    │
    ├─> connection.send(request)
    │       │
    │       └── WebSocket ────────────────────────────────────────────────┐
    │                                                                     │
    ▼                                                                     ▼
consumeFrames()                                            AgentWebSocketServer._handle_message()
    │                                                                     │
    ├─> queue.take() ←───────────────────────────────────────┐           ├─> AgentRequest 解析
    │                                                         │           ├─> is_stream=true
    ├─> transformRelayClawChunk(frame)                       │           │   └─> _handle_stream()
    │       │                                                 │           │       └─> agent.process_message_stream()
    │       └─> event_type 映射:                              │           │
    │           • chat.delta → text                          │           └─> yield AgentResponseChunk
    │           • chat.tool_call → tool_use                   │
    │           • chat.error → error                          │
    │                                                         │
    └─> yield AgentMessage ◄──────────────────────────────────┘
```

### 环境变量传递链

```
clowder-ai 配置                          jiuwenclaw 环境变量
─────────────────────────────────────────────────────────────
callbackEnv.API_KEY                  →   API_KEY
callbackEnv.API_BASE                 →   API_BASE
config.modelName                     →   MODEL_NAME
homeDir                              →   HOME
                                     →   JIUWENCLAW_AGENT_ROOT
workingDirectory                     →   JIUWENCLAW_PROJECT_DIR
CAT_CAFE_MCP_SERVER_PATH             →   CAT_CAFE_MCP_SERVER_PATH
CAT_CAFE_MCP_COMMAND                 →   CAT_CAFE_MCP_COMMAND
动态分配                              →   AGENT_PORT
动态分配                              →   WEB_PORT
```

---
