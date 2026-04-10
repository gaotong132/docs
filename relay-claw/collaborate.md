## 多智能体协作架构全面深入分析

---

### 一、架构总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              用户层 (CVO)                                    │
│                   Vision · Decisions · Feedback                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Platform Layer (Clowder)                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  Identity   │  │  A2A Router │  │   Skills    │  │  Session Chain      │ │
│  │  Manager    │  │  & Worklist │  │  Framework  │  │  Handoff/Compress   │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────────┘ │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  Memory &   │  │  SOP        │  │  MCP Bridge │  │  InvocationQueue    │ │
│  │  Evidence   │  │  Guardian   │  │  (Callback) │  │  & Tracker          │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        ▼                             ▼                             ▼
┌───────────────┐            ┌───────────────┐            ┌───────────────┐
│  dare CLI     │            │  relayclaw    │            │ agent-teams   │
│  (glm-5)      │            │  (gpt-5.4)    │            │ (acp)         │
│  assistant/小理│            │  office/小九   │            │ agentteams/小码│
└───────────────┘            └───────────────┘            └───────────────┘
```

---

### 二、A2A (Agent-to-Agent) 协议详解

#### 2.1 核心组件与职责

| 组件 | 文件路径 | 核心职责 |
|------|---------|---------|
| **a2a-mentions.ts** | `routing/a2a-mentions.ts` | 解析 @mention，行首匹配，token boundary，上限 2 |
| **route-serial.ts** | `routing/route-serial.ts` | 串行路由，worklist 动态扩展，A2A chain 管理 |
| **WorklistRegistry** | `routing/WorklistRegistry.ts` | 全局 worklist 注册表，支持 concurrent isolation (F108) |
| **callback-a2a-trigger.ts** | `routes/callback-a2a-trigger.ts` | MCP callback A2A 触发，enqueue 到 InvocationQueue |
| **InvocationQueue** | `invocation/InvocationQueue.ts` | FIFO 队列，auto-merge，MAX_QUEUE_DEPTH=5 |

#### 2.2 A2A 触发规则

**代码位置**: `packages/api/src/domains/cats/services/agents/routing/a2a-mentions.ts:64-120`

```
解析规则：
1. 剥离代码块 (```...```) 后再解析
2. 仅匹配行首 @mention（可带前导空白）
3. 长匹配优先 + token boundary（@opus-45 不会误命中 @opus）
4. 过滤自调用、检查猫可用性
5. 上限 MAX_A2A_MENTION_TARGETS = 2
```

**匹配示例**:
```
@codex 请 review    ← ✅ 行首，触发
代码中 @opus 不会触发  ← ❌ 非行首
@opus-45 会匹配 @opus-45  ← ✅ token boundary 保护
```

#### 2.3 双路径触发机制

```
┌───────────────────────────────────────────────────────────────────────────┐
│                        A2A 双路径触发                                      │
└───────────────────────────────────────────────────────────────────────────┘

路径 A: TEXT-SCAN PATH (route-serial.ts:581-779)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  猫 A 回复完成 → textContent 累积完毕
                         │
                         ▼
              parseA2AMentions(textContent, catId)
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
    queuedMessages   a2aCount >=    signal.aborted
    Pending?         maxDepth?
          │              │              │
          ▼              ▼              ▼
       BLOCKED        BLOCKED        BLOCKED
       (fairness)     (depth)        (cancel)
          │
          ▼ (pass gates)
    worklist.push(nextCat)
    worklistEntry.a2aCount++
    worklistEntry.a2aFrom.set(nextCat, catId)
    worklistEntry.a2aTriggerMessageId.set(nextCat, storedMsgId)
                         │
                         ▼
              emit a2a_handoff event

路径 B: CALLBACK PATH (callback-a2a-trigger.ts:59-245)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  猫调用 MCP: cat_cafe_post_message(content="@codex 请看")
                         │
                         ▼
              /api/callbacks/post-message
                         │
                         ▼
              parseA2AMentions(content, senderCatId)
                         │
                         ▼
              ┌─────────────────────────────────────────┐
              │ if (invocationQueue available) {         │
              │   // F122B: Unified queue path           │
              │   invocationQueue.enqueue({              │
              │     source: 'agent',                     │
              │     autoExecute: true,                   │
              │     callerCatId: sender                  │
              │   })                                     │
              │   queueProcessor.tryAutoExecute()        │
              │ }                                        │
              └─────────────────────────────────────────┘
                         │
                         ▼
              触发猫 B 调用（slot-free 时自动执行）
```

#### 2.4 Worklist Pattern（统一调度）

**代码位置**: `packages/api/src/domains/cats/services/agents/routing/WorklistRegistry.ts`

```typescript
interface WorklistEntry {
  list: CatId[];                    // 动态扩展的目标列表
  originalCount: number;            // 原始用户选择的目标数
  a2aCount: number;                 // A2A 深度计数器
  maxDepth: number;                 // 最大深度 (默认 15)
  executedIndex: number;            // 已执行的索引
  a2aFrom: Map<CatId, CatId>;       // 谁触发了谁
  a2aTriggerMessageId: Map<CatId, string>; // replyTo 自动对齐
}
```

**关键设计**:
- **F108 并发隔离**: `parentInvocationId` 作为 key，允许多个 invocation 并行
- **Caller Guard**: 只有当前执行的猫才能 push 到 worklist（防止 stale callback 污染）
- **Dedup Strategy**: 只对 pending tail 去重，已执行的猫可 re-enqueue（支持 A→B→A ping-pong）

#### 2.5 InvocationQueue（队列管理）

**代码位置**: `packages/api/src/domains/cats/services/agents/invocation/InvocationQueue.ts`

```
设计原则：
- InvocationTracker: "谁在跑"（slot mutex）
- InvocationQueue: "谁在等"（FIFO queue）

scopeKey = `${threadId}:${userId}` — 天然用户隔离

Entry 类型：
- source: 'user' | 'connector' | 'agent'
- autoExecute: boolean (F122B)
- callerCatId?: string (A2A 来源追踪)
```

**关键方法**:
- `enqueue()`: 自动 merge 同源同目标消息
- `hasQueuedUserMessagesForThread()`: A2A fairness gate
- `hasActiveOrQueuedAgentForCat()`: 跨路径 dedup

---

### 三、Session Handoff Strategy

#### 3.1 三种策略对比

| 策略 | 触发条件 | 行为 | 适用场景 |
|------|---------|------|---------|
| **Handoff** | `sessionChain: true` | Seal → Bootstrap 新 session | 长对话，保留上下文 |
| **Compress** | Provider-native | 压缩但不换 session | 短对话 |
| **Hybrid** | Handoff + compress | 长 session 换，短 session 压缩 | 混合场景 |

#### 3.2 Session 生命周期

**代码位置**: `packages/api/src/domains/cats/services/session/SessionSealer.ts`

```
Session Lifecycle:
━━━━━━━━━━━━━━━━━

  active → [threshold reached] → requestSeal()
                                     │
                                     ▼ (CAS transition)
                                   sealing
                                     │
                                     ▼
                               finalize() {
                                 1. flush transcript (JSONL)
                                 2. write extractive digest
                                 3. update ThreadMemory
                                 4. generate handoff digest (generative)
                                 5. status: sealed
                               }
                                     │
                                     ▼
                                  sealed
```

**安全机制**:
- **CAS Transition**: 只有 `active → sealing` 有效，防止竞态
- **Finalize Timeout**: 30s 超时，强制 seal
- **Global Reaper**: startup 时 force-seal stuck sessions (>5min)
- **Audit Log**: 所有 finalize failure 记录到 `data/audit-logs/`

#### 3.3 Bootstrap Context 构建

**代码位置**: `packages/api/src/domains/cats/services/session/SessionBootstrap.ts`

```
Bootstrap Token Budget: 2000 tokens
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

优先级 (高 → 低):
  1. [Session Identity] — 总是保留
  2. [MCP Tool Recall] — 总是保留
  3. [Thread Memory] — 跨 session 知识
  4. [Previous Session Digest] — 上次做了什么
  5. [Task Snapshot] — 任务状态
  6. [Project Knowledge Recall] — 自动生成，最低优先

Drop Order: recall → task → digest → threadMemory
```

**Bootstrap 结构**:
```
[Session Continuity — Session #N]
This is session #N of M total sessions.

[Thread Memory — K sessions]
{rolling summary across sealed sessions}

[Previous Session Summary]
Duration: 14:30 → 15:45 (75min)
Tools used: search_evidence, post_message
Files touched:
  - packages/api/src/... (modified)
  - docs/features/... (created)

[Session Recall — Available Tools]
- cat_cafe_search_evidence: Start here
- cat_cafe_read_session_events(view=handoff)
- cat_cafe_read_invocation_detail
```

#### 3.4 Handoff Digest 生成

**代码位置**: `packages/api/src/domains/cats/services/session/HandoffDigestGenerator.ts`

```
Generative Digest (Haiku):
━━━━━━━━━━━━━━━━━━━━━━━━━

模型: claude-haiku-4-5-20251001
超时: 5s
输出上限: 1024 tokens
输入上限: 16000 chars (~4000 tokens)

输入:
  - handoffSummaries: per-invocation 摘要
  - extractiveDigest: 文件/工具/错误统计
  - recentMessages: 最近 8 条消息

输出格式:
  ## Session Summary
  - What was accomplished
  - Key decisions made
  - Files changed
  - Errors encountered
```

---

### 四、Identity & Context 系统

#### 4.1 双层身份注入

**代码位置**: `packages/api/src/domains/cats/services/context/SystemPromptBuilder.ts`

```
┌─────────────────────────────────────────────────────────────┐
│                   Static Identity (session-level)           │
│  ─────────────────────────────────────────────────────────  │
│  你是 布偶猫/宪宪 (Claude Opus)，由 Anthropic 提供的 AI 猫猫。 │
│  角色：架构师和核心开发者                                     │
│  性格：...                                                   │
│                                                             │
│  ## 协作                                                     │
│  你可以 @队友: @codex / @gemini                              │
│  格式：另起一行行首写 @猫名                                   │
│                                                             │
│  ## 队友名册                                                 │
│  | 猫猫 | @mention | 擅长 | 注意 |                           │
│  | 缅因猫/砚砚 | @codex | 代码审查 | — |                     │
│                                                             │
│  ## 工作流（主动 @ 触发点）                                   │
│  - 完成开发 → @codex 请 review                               │
│                                                             │
│  ## 家规 (L0 Governance Digest)                              │
│  P1 每步产物是终态基座 P2 自主跑完 SOP...                     │
│                                                             │
│  MCP 工具文档 (仅 Claude)                                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              Dynamic Invocation Context (per-message)       │
│  ─────────────────────────────────────────────────────────  │
│  Identity: 布偶猫/宪宪 (@opus, model=opus-4.6)              │
│  Direct message from 缅因猫(codex); reply to 缅因猫          │
│  你的队友：                                                  │
│  - 缅因猫/砚砚：代码审查                                     │
│  当前模式：你是第 2/3 只被召唤的猫                           │
│  A2A 出口检查：回复前问"到我这里结束了吗？"                   │
│  Voice Mode: OFF                                            │
│  SOP: F042 stage=impl → load skill: tdd                     │
└─────────────────────────────────────────────────────────────┘
```

#### 4.2 L0 Governance Digest

**注入位置**: `SystemPromptBuilder.ts:226-239`

```
家规摘要 (~500 tokens):
━━━━━━━━━━━━━━━━━━━

原则: P1 面向终态 P2 自主跑完SOP P3 方向正确>速度 P4 单一真相源 P5 可验证才算完成

世界观: W1 猫是Agent不是API W2 共享才成团队 W3 用户是CVO...

纪律:
- 不冒充其他猫
- 实事求是——结论基于多源证据
- @是路由指令——发前问"到我这里结束了吗？"
- runtime 禁止擅自重启
- commit 必须带签名 [昵称/模型🐾]

Magic Words:
-「脚手架」= 你在偷懒 → 停，审视产物是否终态
-「绕路了」= 局部最优全局绕路 → 重规划
-「喵约」= 你忘了约定 → 重读家规
-「星星罐子」= P0 不可逆风险 → 立刻停止
```

#### 4.3 Workflow Triggers (per-breed)

```typescript
const WORKFLOW_TRIGGERS = {
  ragdoll: [
    '- 完成开发/修复 → @缅因猫 请 review',
    '- 修完 review 意见 → @缅因猫 确认修复',
    '- 遇到视觉/体验问题 → @暹罗猫 征询',
  ],
  'maine-coon': [
    '- 完成 review → @布偶猫 通知结果',
    '- Review 布偶猫代码：每个发现必须有明确立场',
    '- 接球后默认静默执行',
    '- 出口一问：我这条消息结尾有没有 @ 下一棒？',
  ],
  siamese: [
    '- 完成设计 → 分别 @布偶猫 和 @缅因猫 请确认',
  ],
};
```

---

### 五、MCP 协作工具

**代码位置**: `packages/mcp-server/src/tools/callback-tools.ts`

| 工具 | 功能 | 关键参数 |
|------|------|---------|
| `cat_cafe_post_message` | 发消息，触发 A2A | `content`, `threadId?`, `replyTo?` |
| `cat_cafe_multi_mention` | 并行拉 1-3 只猫 | `searchEvidenceRefs[]` 或 `overrideReason` (硬检查) |
| `cat_cafe_cross_post_message` | 跨 thread 通知 | `targetThreadId`, `content` |
| `cat_cafe_register_pr_tracking` | PR review routing | `prNumber`, `repoOwner`, `repoName` |
| `cat_cafe_get_pending_mentions` | 获取待处理的 @mentions | `includeAcked?` |
| `cat_cafe_get_thread_context` | 读取 thread 消息 | `limit`, `keyword?` |
| `cat_cafe_start_vote` | 发起投票 | `voters[]`, `options[]`, `deadline` |
| `cat_cafe_search_evidence` | 搜索知识库 | `q`, `limit` |

**家规 §13 元思考触发器**:
```
调用 multi_mention 前:
  必须先搜后问
  MCP 层硬检查：缺少 searchEvidenceRefs[] 且无 overrideReason → 拒绝

触发器场景:
  A: 高影响决策 → 先搜 docs/decisions/
  B: 跨领域问题 → 先搜对应领域文档
  C: 高不确定性 → 先搜历史讨论
  D: 信息不足 → 先 search(messages/docs/evidence)
  E: 新领域侦查 → 从 docs/features/README.md 顺藤摸瓜
```

---

### 六、Cross-Cat Review Protocol

**配置来源**: `cat-config.json` + `shared-rules.md`

```json
{
  "reviewPolicy": {
    "requireDifferentFamily": true,
    "preferActiveInThread": true,
    "preferLead": true,
    "excludeUnavailable": true
  }
}
```

**Reviewer Resolution Logic** (`SystemPromptBuilder.ts:578-667`):
```
1. 收集所有 peer-reviewer 角色的猫
2. 过滤掉自己
3. 按 family 分组:
   - crossFamily: 不同家族
   - sameFamily: 同家族
   - unavailable: 没猫粮
4. 优先选择 crossFamily
5. 如果 crossFamily 为空 → fallback sameFamily
```

**Review 规则** (`shared-rules.md:143-156`):
```
1. Reviewer 每个发现必须有明确立场，禁止"修不修都行"
2. Author 收到意见必须独立判断，不能全盘接受
3. 零分歧 = 走过场，双方反思
4. 证据权重排序:
   需求/AC 原文 > 能跑的 feature > Review 意见
5. 改坏能跑的功能 = P0，不管 review 理论多优雅
```

---

### 七、安全护栏

#### 7.1 A2A 安全机制

| 机制 | 实现位置 | 作用 |
|------|---------|------|
| **Depth Limit** | `MAX_A2A_DEPTH = 15` | 防止无限 chain |
| **Queue Fairness Gate** | `hasQueuedUserMessagesForThread()` | 用户消息优先 |
| **Cross-path Dedup** | `hasActiveOrQueuedAgentForCat()` | 防止重复 dispatch |
| **Stale Callback Rejection** | `registry.isLatest()` | 拒绝过期 invocation |
| **Caller Guard** | `pushToWorklist()` | 只有当前执行的猫才能 push |
| **Owner Guard** | `unregisterWorklist()` | 防止 finally 删除新 worklist |

#### 7.2 Session 安全机制

| 机制 | 实现位置 | 作用 |
|------|---------|------|
| **CAS Seal Transition** | `requestSeal()` | 只有 `active → sealing` |
| **Finalize Timeout** | 30s | 超时强制 seal |
| **Global Reaper** | `reconcileAllStuck()` | 清理 stuck sessions |
| **Bootstrap Token Cap** | 2000 tokens | 防止 context bloat |
| **Handoff Sanitization** | `sanitizeHandoffBody()` | 防止 prompt injection |

#### 7.3 共享状态保护

```
三层防御:
━━━━━━━━━

L1: .githooks/pre-commit
    → 非 main 分支 commit 共享文件 → 硬拦

L2: invoke-single-cat.ts runtime preflight
    → unpushed 共享状态 → 硬拦调用

L3: .github/workflows/shared-state-guard.yml
    → PR 含共享状态变更 → 硬拦
```

---

### 八、审计与可观测性

**代码位置**: `packages/api/src/domains/cats/services/orchestration/EventAuditLog.ts`

```
Audit Event Types:
━━━━━━━━━━━━━━━━━

调用级:
- CAT_INVOKED: CLI spawn 前
- CAT_RESPONDED: done 消息后
- CAT_ERROR: 调用错误
- A2A_HANDOFF: 猫猫互调
- CLI_TOOL_STARTED/COMPLETED: 工具执行

Session级:
- SESSION_BIND: 手动绑定 CLI session
- SEAL_FINALIZE_FAILED: seal 失败

治理级:
- DEBATE_WINNER: 辩论冠军
- REVIEW_APPROVED: review 批准
- DECISION_MADE: 重要决策
- MEMORY_PUBLISH_*: 记忆治理

存储: ./data/audit-logs/audit-YYYY-MM-DD.ndjson
格式: JSONL (每行一个 JSON 对象)
```

---

### 九、设计决策与权衡

| 决策 | 理由 | 权衡 |
|------|------|------|
| **行首 @mention 才触发** | 防止代码示例/叙述中误触发 | 需要显式路由意图 |
| **MAX_A2A_MENTION_TARGETS = 2** | 防止 fan-out | 多猫广播需 `multi_mention` |
| **MAX_A2A_DEPTH = 15** | 允许 review ping-pong | 长链风险，queue gate 缓解 |
| **Unified worklist (F27)** | callback + text-scan 单一路径 | 需要 cross-path dedup |
| **Slot-aware Invocation** | 多猫并发 (F108) | 复杂状态管理 |
| **Static identity in -p** | 抵抗 Claude context compression | 非 Claude 需 per-message 注入 |
| **L0 Governance inline** | 猫看不到 shared-rules.md 链接内容 | 增加 ~500 tokens |

---

### 十、关键文件清单

| 文件 | 行数 | 核心职责 |
|------|------|---------|
| `route-serial.ts` | 1009 | 串行路由，A2A chain，worklist 管理 |
| `InvocationQueue.ts` | 484 | FIFO 队列，auto-merge，cross-user |
| `SystemPromptBuilder.ts` | 691 | 身份注入，governance，workflow triggers |
| `WorklistRegistry.ts` | 207 | 全局 worklist，concurrent isolation |
| `SessionSealer.ts` | 331 | Session 生命周期，finalize，reaper |
| `SessionBootstrap.ts` | 318 | Bootstrap context，token budget |
| `callback-a2a-trigger.ts` | 411 | MCP A2A 触发，enqueue/fallback |
| `shared-rules.md` | 411 | 家规（单一真相源） |
| `cat-cafe-skills/manifest.yaml` | 1191 | Skills 路由定义 |

---


# 多智能体协作子系统全面深入分析

## 一、InvocationTracker 与 InvocationQueue 协作机制

### 1.1 InvocationTracker (Slot-Aware Mutex)

**代码位置**: `packages/api/src/domains/cats/services/agents/invocation/InvocationTracker.ts`

```
核心职责：追踪"谁在跑"
━━━━━━━━━━━━━━━━━━━━━━

设计原则:
- Key: `${threadId}:${catId}` (slotKey)
- F108: 多槽并发，同猫同 thread 仍保持单锁语义
- 不同猫可并发，同一猫新旧调用会 abort 旧调用
```

**关键数据结构**:
```typescript
interface ActiveInvocation {
  controller: AbortController;  // 取消信号
  userId: string;               // 所属用户
  catId: string;                // 槽位标识
  catIds: string[];             // 目标猫列表（用于广播取消通知）
}

private active = new Map<string, ActiveInvocation>();  // 活跃槽位
private deleting = new Set<string>();                  // 删除保护
```

**核心方法流程**:

```
start(threadId, catId, userId, catIds):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  1. 检查 deleting set → 如果正在删除，返回 pre-aborted controller
  2. 获取 slotKey = `${threadId}:${catId}`
  3. abort 现有同槽位 invocation (preempted)
  4. 创建新 AbortController
  5. 注册到 active map
  6. 返回 controller

tryStartThread(threadId, catId, userId, catIds): (F122)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  1. 原子检查：任何槽位活跃？→ return null
  2. 非抢占式：NEVER abort 现有 invocation
  3. 成功则注册并返回 controller

guardDelete(threadId):
━━━━━━━━━━━━━━━━━━━━━
  1. 检查 deleting set
  2. 检查任何槽位活跃
  3. 通过 → 加入 deleting set，返回 guard
  4. 失败 → 返回 { acquired: false }

cancel(threadId, catId, requestUserId?, abortReason?):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  1. 获取槽位 invocation
  2. 验证 userId 匹配（可选）
  3. abort + 从 active 移除
  4. 返回 { cancelled: true, catIds }
```

**并发隔离 (F108)**:
```
Thread A:
  Slot: thread-1:opus    → 运行中
  Slot: thread-1:codex   → 运行中（并发）

Thread B:
  Slot: thread-1:opus    → abort 旧的，启动新的
  Slot: thread-1:gemini  → 新启动（与 codex 并发）
```

### 1.2 InvocationQueue (FIFO 队列)

**代码位置**: `packages/api/src/domains/cats/services/agents/invocation/InvocationQueue.ts`

```
核心职责：追踪"谁在等"
━━━━━━━━━━━━━━━━━━━━━━

设计原则:
- scopeKey = `${threadId}:${userId}` — 天然用户隔离
- MAX_QUEUE_DEPTH = 5
- 同源同目标自动 merge
```

**Entry 类型**:
```typescript
interface QueueEntry {
  id: string;
  threadId: string;
  userId: string;
  content: string;
  messageId: string | null;
  mergedMessageIds: string[];      // merge 的消息 ID
  source: 'user' | 'connector' | 'agent';
  targetCats: string[];
  intent: string;
  status: 'queued' | 'processing';
  createdAt: number;
  autoExecute: boolean;            // F122B: 自动执行
  callerCatId?: string;            // F122B: A2A 来源
  senderMeta?: { id; name? };      // F134: connector sender
  resumeCatId?: CatId;             // resume 目标
}
```

**关键方法详解**:

```
enqueue(input):
━━━━━━━━━━━━━━━━
  1. 检查 merge 条件:
     - tail.status === 'queued'
     - same source, intent, targetCats
     - source !== 'connector' (F134: connector 永不 merge)
  
  2. Merge 成功 → 更新 content，保存 preMergeSnapshot
  
  3. Merge 失败 → 检查容量:
     - queuedCount >= MAX_QUEUE_DEPTH → { outcome: 'full' }
  
  4. 创建新 entry → 保存 originalContent

rollbackMerge(threadId, userId, entryId):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  1. 恢复 preMergeSnapshot
  2. 删除 snapshot

rollbackEnqueue(threadId, userId, entryId):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  场景：enqueue 后 write 失败，需要回滚
  
  1. 如果 content 已被 merge（!= original）:
     - strip 原始内容前缀
     - promote mergedMessageId 到 messageId
  2. 否则：remove entry entirely
```

**Cross-User 方法**:
```
peekOldestAcrossUsers(threadId):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  遍历所有 scopeKey，找 createdAt 最小的 queued entry

markProcessingAcrossUsers(threadId):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  原子标记最老的 entry 为 processing

hasQueuedUserMessagesForThread(threadId):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  A2A fairness gate — 只检查 source='user'

hasActiveOrQueuedAgentForCat(threadId, catId):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Cross-path dedup — 检查 queued + processing
```

### 1.3 协作流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│              InvocationTracker + InvocationQueue 协作                  │
└─────────────────────────────────────────────────────────────────────────┘

消息到达 → AgentRouter.routeExecution()
              │
              ├─ 检查 InvocationTracker.has(threadId, catId)
              │    │
              │    ├─ 有活跃调用 → abort 旧的
              │    └─ 无活跃调用 → 继续
              │
              ├─ InvocationTracker.start(threadId, catId, userId, catIds)
              │    │
              │    └─ 返回 AbortController
              │
              ├─ 检查 InvocationQueue.size(threadId, userId)
              │    │
              │    ├─ 有排队消息 → 加入队列
              │    │                  │
              │    │                  └─ 返回 { outcome: 'enqueued' }
              │    │
              │    └─ 无排队 → 直接执行
              │
              └─ 执行完成后:
                   │
                   ├─ InvocationTracker.complete(threadId, catId, controller)
                   │
                   └─ QueueProcessor.onInvocationComplete(threadId, catId, status)
                        │
                        ├─ succeeded → tryExecuteNextAcrossUsers()
                        └─ canceled/failed → pause slot, emit queue_paused
```

---

## 二、QueueProcessor 队列处理引擎

**代码位置**: `packages/api/src/domains/cats/services/agents/invocation/QueueProcessor.ts`

### 2.1 核心状态管理

```typescript
class QueueProcessor {
  private processingSlots = new Set<string>();      // F108: 槽位 mutex
  private pausedSlots = new Map<string, 'canceled' | 'failed'>();  // 暂停原因
  private entryCompleteHooks = new Map<string, EntryCompleteHook>(); // F122B: 完成回调
}
```

### 2.2 执行流程

```
executeEntry(entry):
━━━━━━━━━━━━━━━━━━━

Phase 1: 准备
──────────────
  1. invocationRecordStore.create({ idempotencyKey: `queue-${entry.id}` })
  2. invocationTracker.start(threadId, primaryCat, userId, targetCats)
  3. backfill messageId
  4. mark running
  5. broadcast intent_mode

Phase 2: 标记已投递 (F098-D)
─────────────────────────────
  for each (messageId + mergedMessageIds):
    messageStore.markDelivered(mid, deliveredNow)
    emit messages_delivered

Phase 3: 执行
─────────────
  for await (msg of router.routeExecution(...)):
    - 收集 text content (responseText)
    - 收集 outbound turns (per-cat text/richBlocks)
    - 调用 streamingHook.onStreamChunk()
    - broadcast agent message

Phase 4: 完成处理
─────────────────
  if (aborted):
    finalStatus = 'canceled'
  else:
    ack cursors
    finalStatus = 'succeeded'

Phase 5: Outbound 投递 (F088)
──────────────────────────────
  for each outboundTurn:
    outboundHook.deliver(threadId, content, catId, richBlocks, threadMeta)

Phase 6: 清理
─────────────
  invocationTracker.complete(threadId, primaryCat, controller)
  queue.removeProcessedAcrossUsers(threadId, entry.id)
  emit queue_updated
  fire entryCompleteHook
```

### 2.3 自动执行机制 (F122B)

```
tryAutoExecute(threadId):
━━━━━━━━━━━━━━━━━━━━━━━━

  1. 获取所有 autoExecute entries (按 createdAt 排序)
  
  2. for each entry:
       ├─ 检查 processingSlots.has(slotKey)
       ├─ 检查 invocationTracker.has(threadId, catId)
       │
       ├─ slot free → markProcessingById
       │              │
       │              └─ void executeEntry(entry).then(
       │                    (status) => {
       │                      processingSlots.delete(sk)
       │                      onInvocationComplete(...)
       │                    }
       │                  )
       │
       └─ continue scanning (parallel dispatch)

关键特性：
- 多猫并行 dispatch（不等待上一个完成）
- Per-cat slot mutex 防止冲突
- Fire-and-forget execution
```

### 2.4 暂停/恢复机制

```
onInvocationComplete(threadId, catId, status):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  succeeded:
    ├─ pausedSlots.delete(slotKey)
    └─ if (queue.hasQueued):
         tryExecuteNextAcrossUsers()
         tryAutoExecute()

  canceled/failed:
    ├─ if (!queue.hasQueued):
    │     pausedSlots.delete(slotKey)
    │     return
    │
    └─ pausedSlots.set(slotKey, status)
        emitPausedToQueuedUsers()

processNext(threadId, userId): (手动恢复)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  clearPause(threadId)
  tryExecuteNextForUser(threadId, userId)
```

---

## 三、ThreadMemory Rolling Summary

**代码位置**: `packages/api/src/domains/cats/services/session/buildThreadMemory.ts`

### 3.1 设计原则

```
ThreadMemory 目的:
━━━━━━━━━━━━━━━━━
跨 session 持久知识 — 滚动摘要，每 seal 一个 session 就追加

Token Budget:
  maxTokens = min(3000, floor(maxPromptTokens * 0.03))
  floor 1200 (hard minimum)

存储位置: ThreadStore.threadMemory
```

### 3.2 生成流程

```
buildThreadMemory(existing, newDigest, maxTokens):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Step 1: 格式化新 session 行
──────────────────────────
  formatSessionLine(digest, sessionNumber):
    └─ "Session #N (14:30-15:45, 75min): tool1, tool2. Files: file1.ts, file2.ts. 2 errors."

Step 2: Prepend 新行
────────────────────
  existingLines = existing?.summary?.split('\n') ?? []
  allLines = [newLine, ...existingLines]

Step 3: Token budget 裁剪
─────────────────────────
  while (estimateTokens(summary) > maxTokens && allLines.length > 1):
    allLines.pop()  // 移除最旧的

Step 4: Hard-cap 单行
─────────────────────
  if (estimateTokens(summary) > maxTokens):
    truncate to ratio

Step 5: 返回
───────────
  {
    v: 1,
    summary,
    sessionsIncorporated: mergeCount,
    updatedAt: Date.now()
  }
```

### 3.3 更新时机

```
SessionSealer.doFinalize():
━━━━━━━━━━━━━━━━━━━━━━━━━━

  if (threadStore && transcriptReader):
    digest = transcriptReader.readDigest(sessionId)
    existingMemory = threadStore.getThreadMemory(threadId)
    maxTokens = min(3000, floor(maxPromptTokens * 0.03))
    updated = buildThreadMemory(existingMemory, digest, maxTokens)
    threadStore.updateThreadMemory(threadId, updated)
```

### 3.4 Bootstrap 注入

```
SessionBootstrap.buildSessionBootstrap():
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Priority order (高 → 低):
    1. [Session Identity] — always keep
    2. [MCP Tool Recall] — always keep
    3. [Thread Memory] — highest variable priority
    4. [Previous Session Digest]
    5. [Task Snapshot]
    6. [Project Knowledge Recall]

  Drop order: recall → task → digest → threadMemory
```

---

## 四、Draft Store 与 F5 恢复机制

**代码位置**: `packages/api/src/domains/cats/services/stores/ports/DraftStore.ts`

### 4.1 设计目的

```
问题：猫正在流式输出，用户 F5 刷新 → 所有内容丢失
解决：流式过程中定期持久化到 Redis，刷新后恢复
```

### 4.2 数据结构

```typescript
interface DraftRecord {
  userId: string;
  threadId: string;
  invocationId: string;    // 主键（支持并行 streaming）
  catId: CatId;
  content: string;         // 累积的文本内容
  toolEvents?: unknown[];  // 工具调用记录
  thinking?: string;       // thinking blocks
  updatedAt: number;
}
```

### 4.3 Redis 存储

```
Key 设计:
  draft:{userId}:{threadId}:{invocationId}  → Hash (draft details)
  drafts:idx:{userId}:{threadId}            → Set (invocationId members)

TTL: 300s (5 minutes)
```

### 4.4 写入流程

```
route-serial.ts 流式循环中:
━━━━━━━━━━━━━━━━━━━━━━━━━

  FLUSH_INTERVAL_MS = 2000
  FLUSH_CHAR_DELTA = 2000

  for await (msg of invokeSingleCat(...)):
    if (msg.type === 'text'):
      textContent += msg.content
      
      // 检查 flush 条件
      charDelta = textContent.length - lastFlushLen
      if (neverFlushed || now - lastFlushTime >= 2000ms || charDelta >= 2000):
        draftStore.upsert({
          userId, threadId, invocationId, catId,
          content: textContent,
          toolEvents: collectedToolEvents,
          thinking: thinkingContent
        })

    // Tool event heartbeat
    if (msg.type === 'tool_use' || 'tool_result'):
      if (now - lastFlushTime >= 2000ms):
        draftStore.touch(userId, threadId, invocationId)  // 续命 TTL
```

### 4.5 恢复流程

```
前端 F5 刷新 → GET /api/threads/:threadId/drafts
                               │
                               ▼
              RedisDraftStore.getByThread(userId, threadId)
                               │
                               ▼
              返回 DraftRecord[]
                               │
                               ▼
              前端渲染为 "Recovering draft..." bubble
```

### 4.6 清理时机

```
成功完成流式输出后:
  messageStore.append(...) → 存储 persist
  draftStore.delete(userId, threadId, invocationId)

失败/取消时:
  finally {
    draftStore.delete(userId, threadId, invocationId)
  }
```

---

## 五、Evidence Store 与 MCP 工具集成

### 5.1 Evidence Store 架构

```
存储: SQLite (per-project)
文件: {projectPath}/.cat-cafe/evidence.sqlite
引擎: FTS5 full-text search

Schema:
  CREATE TABLE evidence (
    id TEXT PRIMARY KEY,
    content TEXT,
    sourceType TEXT,
    sourceId TEXT,
    anchor TEXT,
    tags TEXT,  -- JSON array
    metadata TEXT,  -- JSON object
    createdAt INTEGER,
    updatedAt INTEGER
  );
  
  CREATE VIRTUAL TABLE evidence_fts USING fts5(
    content, sourceType, anchor, tags
  );
```

### 5.2 MCP 工具：search_evidence

**代码位置**: `packages/mcp-server/src/tools/callback-memory-tools.ts`

```typescript
searchEvidenceInputSchema = {
  query: z.string().trim().min(1),
  limit: z.number().int().min(1).max(20).optional().default(5),
  budget: z.enum(['low', 'mid', 'high']).optional(),
  tags: z.string().optional(),
  tagsMatch: z.enum(['any', 'all', 'any_strict', 'all_strict']).optional()
}
```

**Budget Profiles**:
```
low:   limit=3, maxSnippetLen=100
mid:   limit=5, maxSnippetLen=200  (default)
high:  limit=10, maxSnippetLen=400
```

**Tag Matching**:
```
any:        至少匹配一个 tag (OR)
all:        匹配所有 tag (AND)
any_strict: 必须有 tag，至少匹配一个
all_strict: 必须有 tag，匹配所有
```

### 5.3 MCP 工具：reflect

```typescript
reflectInputSchema = {
  query: z.string().trim().min(1)
}

实现：
  1. search_evidence(query, limit=10)
  2. 合并 hits 为 context
  3. 调用 LLM 生成 insight
  4. 返回合成结果
```

### 5.4 Session Chain 工具

**代码位置**: `packages/mcp-server/src/tools/session-chain-tools.ts`

```
工具列表:
━━━━━━━━

cat_cafe_list_session_chain(threadId, catId?, limit?):
  └─ GET /api/threads/:threadId/sessions

cat_cafe_read_session_events(sessionId, cursor?, limit?, view?):
  ├─ view='raw'     → full JSONL events
  ├─ view='chat'    → role/content pairs
  └─ view='handoff' → per-invocation summaries (RECOMMENDED)

cat_cafe_read_session_digest(sessionId):
  └─ GET /api/sessions/:sessionId/digest

cat_cafe_read_invocation_detail(sessionId, invocationId):
  └─ GET /api/sessions/:sessionId/invocations/:invocationId

cat_cafe_session_search(threadId, query, cats?, limit?, scope?):
  ├─ scope='digests'    → search extractive digests
  ├─ scope='transcripts' → search JSONL transcripts
  └─ scope='both'       → search both (default)
```

**使用模式**:
```
推荐流程:
  1. search_evidence(query) → 找相关知识
  2. read_session_events(view='handoff') → 获取 session 概览
  3. read_invocation_detail(sessionId, invocationId) → 深入特定调用
```

---

## 六、Routing Policy 系统

**代码位置**: `packages/api/src/domains/cats/services/stores/ports/ThreadStore.ts`

### 6.1 数据结构

```typescript
type ThreadRoutingScope = 'review' | 'architecture';

interface ThreadRoutingRule {
  preferCats?: CatId[];     // 优先路由
  avoidCats?: CatId[];      // 避免路由（除非显式 @mention）
  reason?: string;          // 人类可读原因
  expiresAt?: number;       // 过期时间（epoch ms）
}

interface ThreadRoutingPolicyV1 {
  v: 1;
  scopes?: Partial<Record<ThreadRoutingScope, ThreadRoutingRule>>;
}
```

### 6.2 使用场景

```
示例：某 thread 因预算原因不想用 Opus review

routingPolicy = {
  v: 1,
  scopes: {
    review: {
      avoidCats: ['opus'],
      reason: 'budget',
      expiresAt: Date.now() + 7 * 24 * 60 * 60 * 1000  // 7 天后过期
    }
  }
}
```

### 6.3 注入到 System Prompt

**代码位置**: `SystemPromptBuilder.ts:482-514`

```
注入格式:
  Routing: review avoid @opus, @gemini (budget); architecture prefer @opus

注入时机: buildInvocationContext()
持久性: per-invocation，抵抗 context compression
```

---

## 七、Thread Store 扩展功能

### 7.1 Participant Activity Tracking (F032)

```typescript
interface ThreadParticipantActivity {
  catId: CatId;
  lastMessageAt: number;    // 最后消息时间戳
  messageCount: number;      // 消息计数
}

存储: participantActivity Map<`${threadId}:${catId}`, activity>
```

**用途**:
```
1. Reviewer matching: preferActiveInThread=true
   → 选择 thread 中最近活跃的猫

2. Active participant hint in SystemPromptBuilder:
   "最近活跃：缅因猫(codex)"
```

### 7.2 Mention Routing Feedback (F046 D3)

```typescript
interface ThreadMentionRoutingFeedback {
  sourceMessageId?: string;
  sourceTimestamp: number;
  items: Array<{
    targetCatId: CatId;
    reason: 'no_action' | 'cross_paragraph';
  }>;
}
```

**流程**:
```
1. 猫 A 回复中 @codex 但未用行首格式
   → 系统 detect non-routed mention

2. 记录到 mentionRoutingFeedback

3. 猫 A 下次调用时：
   SystemPromptBuilder 注入:
   "[路由提醒] 上次你提到了 @codex 但没有用行首 @ 路由。
    如果需要对方行动，请在行首独立一行写 @句柄。"

4. consume 后清除
```

### 7.3 Voting State (F079)

```typescript
interface VotingStateV1 {
  v: 1;
  question: string;
  options: string[];
  votes: Record<string, string>;  // catId/userId → option
  anonymous: boolean;
  deadline: number;
  createdBy: string;
  status: 'active' | 'closed';
  voters?: string[];              // designated voters
  initiatedByCat?: string;        // cat-initiated vote
}
```

**自动关闭逻辑** (`route-serial.ts:586-663`):
```
每条消息检查 [VOTE:option] 标记:
  1. 提取 votedOption
  2. 验证: deadline 未过期 + voter 在 voters 列表
  3. 记录 vote
  4. 检查 checkVoteCompletion()
  5. 全部投完 → 生成结果 richBlock，broadcast，clear state
```

### 7.4 Bootcamp State (F087)

```typescript
type BootcampPhase =
  | 'phase-0-select-cat'
  | 'phase-1-intro'
  | 'phase-2-env-check'
  | 'phase-3-config-help'
  | 'phase-3.5-advanced'
  | 'phase-4-task-select'
  | 'phase-5-kickoff'
  | 'phase-6-design'
  | 'phase-7-dev'
  | 'phase-8-review'
  | 'phase-9-complete'
  | 'phase-10-retro'
  | 'phase-11-farewell';

interface BootcampStateV1 {
  v: 1;
  phase: BootcampPhase;
  leadCat?: CatId;
  selectedTaskId?: string;
  envCheck?: Record<string, { ok, version?, note? }>;
  advancedFeatures?: Record<string, 'available' | 'unavailable' | 'skipped'>;
  startedAt: number;
  completedAt?: number;
}
```

**注入**: `SystemPromptBuilder` 注入 `Bootcamp Mode: phase=N → load bootcamp-guide skill`

---

## 八、系统间协作总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        核心子系统协作关系图                                  │
└─────────────────────────────────────────────────────────────────────────────┘

                         ┌─────────────────────┐
                         │   AgentRouter       │
                         │ routeExecution()    │
                         └─────────┬───────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        ▼                          ▼                          ▼
┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐
│ InvocationTracker │    │ InvocationQueue   │    │ QueueProcessor    │
│ "谁在跑"          │    │ "谁在等"          │    │ 队列执行引擎      │
└───────────────────┘    └───────────────────┘    └───────────────────┘
        │                          │                          │
        │                          │                          │
        └──────────────────────────┼──────────────────────────┘
                                   │
                                   ▼
                         ┌─────────────────────┐
                         │  Session Chain      │
                         │  Handoff Strategy   │
                         └─────────┬───────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        ▼                          ▼                          ▼
┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐
│ ThreadMemory      │    │ Evidence Store    │    │ Draft Store       │
│ 滚动摘要          │    │ 知识库 (SQLite)   │    │ F5 恢复  │
└───────────────────┘    └───────────────────┘    └───────────────────┘
        │                          │                          │
        │                          │                          │
        └──────────────────────────┼──────────────────────────┘
                                   │
                                   ▼
                         ┌─────────────────────┐
                         │   MCP Tools         │
                         │ search_evidence     │
                         │ reflect             │
                         │ session_chain_*     │
                         └─────────────────────┘
```

---

## 九、关键设计决策总结

| 子系统 | 关键决策 | 理由 |
|--------|---------|------|
| **InvocationTracker** | Slot-aware mutex (F108) | 允许不同猫并发，同一猫新旧调用 abort |
| **InvocationQueue** | scopeKey = `${threadId}:${userId}` | 天然用户隔离，cross-user FIFO |
| **QueueProcessor** | Fire-and-forget execution | 避免阻塞队列链 |
| **ThreadMemory** | Token budget = min(3000, 3% maxPrompt) | 平衡上下文保留与 token 成本 |
| **Draft Store** | TTL 300s + touch续命 | 长工具调用期间保持存活 |
| **Evidence Store** | SQLite + FTS5 | 本地持久化，全文搜索，无外部依赖 |
| **Routing Policy** | Per-thread + expiry | 临时偏好，自动过期 |

---
