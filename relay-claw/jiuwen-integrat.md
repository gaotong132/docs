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

## jiuwenclaw 作为 Sidecar 拉起的完整流程

### 1. 架构概览

```
RelayClawAgentService (Node.js API 层)
    ↓
DefaultRelayClawSidecarController (Sidecar 控制器)
    ↓
spawn() → Python 子进程
    ↓
jiuwenclaw/.venv/Scripts/python.exe -m jiuwenclaw.app
```

---

### 2. 启动入口

**`RelayClawAgentService.invoke()`** (`RelayClawAgentService.ts:115-159`)

```typescript
const runtime = this.getOrCreateScopeRuntime(scope);
await this.ensureConnected(runtime, signal, options);
```

调用 `sidecar.ensureStarted()` 启动 sidecar。

---

### 3. Sidecar 控制逻辑

**`DefaultRelayClawSidecarController.ensureStarted()`** (`relayclaw-sidecar.ts:78-107`)

1. 构建 runtime 配置 (`buildRuntime`)
2. 计算 runtimeHash 检查是否需要重启
3. 调用 `start()` 启动进程

---

### 4. Python Bin 选择逻辑

**`resolveJiuwenClawPythonBin()`** (`jiuwenclaw-paths.ts:35-67`)

```typescript
// 优先级顺序:
1. 显式配置 (CAT_CAFE_RELAYCLAW_PYTHON 环境变量或 config.pythonBin)
2. vendor/jiuwenclaw/.venv/Scripts/python.exe  ← 实际选中
3. tools/python/python.exe (bundled Python)
4. legacy path (/usr/code/relay-claw/.venv)
```

**实际选中路径：**
```
D:\project\gaotong\relay-claw\vendor\jiuwenclaw\.venv\Scripts\python.exe
```

---

### 5. 启动命令

**`buildRelayClawLaunchCommand()`** (`relayclaw-sidecar.ts:336-350`)

```typescript
if (runtime.useExecutable) {
    // 打包后的 exe
    return { command: executablePath, args: ['--desktop-run-app'] };
}
// 源码模式（当前使用）
return {
    command: pythonBin,    // .venv/Scripts/python.exe
    args: ['-m', 'jiuwenclaw.app'],
    cwd: appDir,           // vendor/jiuwenclaw/
};
```

---

### 6. 环境变量注入

**`buildRuntime()`** (`relayclaw-sidecar.ts:137-203`)

关键环境变量：
| 变量 | 来源 |
|------|------|
| `API_KEY` | callbackEnv |
| `API_BASE` | callbackEnv |
| `MODEL_NAME` | config.modelName |
| `HOME` | `.cat-cafe/relayclaw/{catId}` |
| `AGENT_PORT` | 动态分配 |
| `WEB_PORT` | 动态分配 |
| `JIUWENCLAW_PROJECT_DIR` | workingDirectory |

Windows 平台还会注入：
```typescript
spawnEnv.PYTHONIOENCODING = 'utf-8';
spawnEnv.PYTHONUTF8 = '1';
// prepend bundled Python to PATH
withBundledPythonPath(spawnEnv, ...)
```

---

### 7. 就绪检测

**`isRelayClawRuntimeReady()`** (`relayclaw-sidecar.ts:352-369`)

等待条件：
1. TCP probe agentPort 成功
2. 日志包含 `[JiuWenClaw] 初始化完成` 或 `WebChannel 已启动`
3. 或 TCP probe webPort 成功

超时默认 180 秒 (`startupTimeoutMs`)

---

### 8. 为什么是 uv 环境

`vendor/jiuwenclaw/.venv` 由 uv 创建（`pyvenv.cfg` 显示 `uv = 0.11.3`）。

该 venv 的 stdlib 指向 uv 的基础 Python：
```
C:\Users\HW\AppData\Roaming\uv\python\cpython-3.12-windows-x86_64-none\Lib\
```

该目录存在 `EXTERNALLY-MANAGED` 文件，导致 pip 检测到 externally-managed-environment。

---

### 9. 调用时序图

```
用户请求 → RelayClawAgentService.invoke()
              ↓
         resolveScope() → 按 API_KEY/API_BASE/MODEL_NAME 计算 scopeHash
              ↓
         getOrCreateScopeRuntime() → 创建 SidecarController
              ↓
         ensureConnected()
              ↓
         sidecar.ensureStarted()
              ↓
         buildRuntime() → 解析 pythonBin, appDir, 环境变量
              ↓
         spawn(python, ['-m', 'jiuwenclaw.app'], {cwd: appDir, env: spawnEnv})
              ↓
         等待 TCP probe + 日志初始化标记
              ↓
         WebSocket 连接建立 → 开始对话
```

---

