# Gateway 与 Agent 交互源码分析

本文档详细分析 OpenClaw 中 Gateway (网关) 与 Agent (智能体) 之间的交互机制,包括消息流转、会话管理、流式响应、错误处理等核心功能。

## 目录

1. [消息接收与路由](#1-消息接收与路由)
2. [Gateway-Agent API 接口](#2-gateway-agent-api-接口)
3. [完整请求-响应周期](#3-完整请求-响应周期)
4. [会话管理与状态](#4-会话管理与状态)
5. [中间件与预处理管道](#5-中间件与预处理管道)
6. [Agent 调用执行](#6-agent-调用执行)
7. [流式响应与分块回复](#7-流式响应与分块回复)
8. [消息队列机制](#8-消息队列机制)
9. [错误处理与超时机制](#9-错误处理与超时机制)
10. [并发消息处理](#10-并发消息处理)
11. [响应规范化与交付](#11-响应规范化与交付)
12. [核心函数签名](#12-核心函数签名)
13. [数据结构与接口](#13-数据结构与接口)

---

## 1. 消息接收与路由

### 1.1 入口点

消息从各个通道(Telegram、Slack、Discord、iMessage 等)接收后,通过通道特定的监听器进入系统。分发流程从 `dispatchInboundMessage` 函数开始。

**文件位置:** `src/auto-reply/dispatch.ts`

**函数签名:**

```typescript
export async function dispatchInboundMessage(params: {
  ctx: MsgContext | FinalizedMsgContext;
  cfg: OpenClawConfig;
  dispatcher: ReplyDispatcher;
  replyOptions?: Omit<GetReplyOptions, "onToolResult" | "onBlockReply">;
  replyResolver?: typeof import("./reply.js").getReplyFromConfig;
}): Promise<DispatchInboundResult>
```

### 1.2 初始处理步骤

1. **上下文最终化:** 使用 `finalizeInboundContext()` 规范化所有上下文字段
2. **重复检测:** 通过 `shouldSkipDuplicateInbound()` 检查消息是否重复
3. **Hook 执行:** 运行 `message_received` hook
4. **会话状态解析:** 从会话存储中获取或初始化会话状态

**代码示例:**

```typescript
// src/auto-reply/dispatch.ts
const finalized = finalizeInboundContext(ctx, cfg);

// 检查是否应跳过重复消息
if (shouldSkipDuplicateInbound(finalized, cfg)) {
  return { skipped: true, reason: "duplicate" };
}

// 执行 message_received hook
await runHook("message_received", {
  context: finalized,
  config: cfg
});

// 解析会话状态
const sessionState = await initSessionState({
  sessionKey: finalized.SessionKey,
  config: cfg
});
```

---

## 2. Gateway-Agent API 接口

### 2.1 核心入口函数: `getReplyFromConfig`

这是 Gateway 调用 Agent 的主要接口函数。

**文件位置:** `src/auto-reply/reply/get-reply.ts`

**函数签名:**

```typescript
export async function getReplyFromConfig(
  ctx: MsgContext,
  opts?: GetReplyOptions,
  configOverride?: OpenClawConfig,
): Promise<ReplyPayload | ReplyPayload[] | undefined>
```

### 2.2 参数说明

**`ctx` (MsgContext):** 消息上下文,包含:
- 发送者信息 (`From`)
- 消息体 (`Body`, `RawBody`, `BodyForCommands`)
- 通道元数据 (`Provider`, `Surface`)
- 会话标识 (`SessionKey`)
- 聊天类型 (`ChatType`: "dm" | "group" | "channel")

**`opts` (GetReplyOptions):** 回调和控制选项,包含:

```typescript
export type GetReplyOptions = {
  runId?: string;                    // 唯一运行标识符
  abortSignal?: AbortSignal;        // 用于取消操作
  images?: ImageContent[];          // 内联附件

  // 生命周期回调
  onAgentRunStart?: (runId: string) => void;
  onReplyStart?: () => Promise<void> | void;
  onTypingCleanup?: () => void;

  // 流式响应回调
  onPartialReply?: (payload: ReplyPayload) => Promise<void> | void;
  onReasoningStream?: (payload: ReplyPayload) => Promise<void> | void;
  onBlockReply?: (payload: ReplyPayload, context?: BlockReplyContext) => Promise<void> | void;
  onToolResult?: (payload: ReplyPayload) => Promise<void> | void;

  // 模型选择回调
  onModelSelected?: (ctx: ModelSelectedContext) => void;

  // 控制选项
  disableBlockStreaming?: boolean;
  blockReplyTimeoutMs?: number;
  skillFilter?: string[];
  hasRepliedRef?: { value: boolean };

  // 心跳运行标识
  isHeartbeat?: boolean;
  heartbeatModelOverride?: string;
};
```

### 2.3 返回值类型: `ReplyPayload`

```typescript
export type ReplyPayload = {
  text?: string;                     // 文本内容
  mediaUrl?: string;                 // 单个媒体 URL
  mediaUrls?: string[];              // 多个媒体 URL
  replyToId?: string;                // 回复目标消息 ID
  replyToTag?: boolean;              // 是否标记回复
  replyToCurrent?: boolean;          // 是否回复当前消息
  audioAsVoice?: boolean;            // 音频是否作为语音消息
  isError?: boolean;                 // 是否为错误消息
  channelData?: Record<string, unknown>; // 通道特定数据
};
```

---

## 3. 完整请求-响应周期

### 3.1 流程图

```
通道消息
    ↓
dispatchReplyFromConfig (dispatch-from-config.ts)
    ↓
 ├─ finalized = finalizeInboundContext(ctx)
 ├─ sessionState = initSessionState(...)
 ├─ directiveResult = resolveReplyDirectives(...)
 ├─ inlineActionResult = handleInlineActions(...)
 └─ runPreparedReply(...)
    ↓
 ├─ 媒体/链接理解
 ├─ 命令授权检查
 ├─ 会话初始化
 ├─ 技能快照构建
 └─ runReplyAgent(...)
    ↓
 runAgentTurnWithFallback (agent-runner-execution.ts)
    ↓
 ├─ runEmbeddedPiAgent(...) [主要 Agent 调用]
 │   ├─ 模型解析
 │   ├─ 认证配置轮转
 │   ├─ 上下文窗口验证
 │   ├─ 工具执行循环
 │   ├─ 错误处理与回退
 │   └─ onBlockReply (流式回调)
 │
 ├─ 会话更新与持久化
 ├─ 使用量跟踪
 └─ 响应构建
    ↓
buildReplyPayloads (agent-runner-payloads.ts)
    ↓
normalizeReplyPayload (normalize-reply.ts)
    ↓
 ReplyDispatcher
 │   ├─ sendToolResult(payload) → onToolResult
 │   ├─ sendBlockReply(payload) → onBlockReply
 │   └─ sendFinalReply(payload) → 最终响应
    ↓
routeReply 或 dispatcher.send*
    ↓
通道交付
```

### 3.2 关键阶段说明

**阶段 1: 消息预处理**
- 上下文最终化
- 重复检测
- Hook 执行

**阶段 2: 会话准备**
- 会话状态加载
- 指令解析 (directive resolution)
- 内联动作处理

**阶段 3: Agent 执行**
- 模型与认证配置选择
- 会话管理器初始化
- API 调用与工具执行

**阶段 4: 响应处理**
- Payload 构建
- 规范化处理
- 通过 Dispatcher 交付

---

## 4. 会话管理与状态

### 4.1 SessionEntry 结构

**文件位置:** `src/config/sessions/types.ts`

```typescript
export type SessionEntry = {
  sessionId: string;                 // UUID,用于标识对话记录
  updatedAt: number;                 // 最后更新时间戳
  sessionFile?: string;              // 对话记录文件路径
  spawnedBy?: string;                // 父会话密钥
  systemSent?: boolean;              // 系统提示是否已发送
  abortedLastRun?: boolean;          // 上次运行是否中止

  // 会话配置
  chatType?: SessionChatType;        // "dm" | "group" | "channel"
  thinkingLevel?: string;            // 思考等级: "off" | "low" | "medium" | "high" | "xhigh"
  verboseLevel?: string;             // 工具输出详细程度
  reasoningLevel?: string;           // Claude 推理等级
  elevatedLevel?: string;            // 提升权限等级
  ttsAuto?: TtsAutoMode;             // 音频自动启用模式
  execHost?: string;                 // 沙箱主机偏好
  modelOverride?: string;            // 每会话模型覆盖
  authProfileOverride?: string;      // 认证配置覆盖

  // 队列配置
  queueMode?: "steer" | "followup" | "collect" | "queue" | "interrupt";

  // 使用量跟踪
  inputTokens?: number;
  outputTokens?: number;
  totalTokens?: number;

  // 技能快照
  skillsSnapshot?: SessionSkillSnapshot;

  // 跨通道元数据
  origin?: SessionOrigin;
};
```

### 4.2 会话密钥解析流程

```
MsgContext.SessionKey
    ↓ (或 CommandTargetSessionKey 用于原生命令)
resolveSessionAgentId(sessionKey, config)
    ↓
Agent 目录与工作区解析
    ↓
会话存储查找 (JSON 文件)
```

### 4.3 状态持久化

**存储位置:**
- 会话存储: `src/config/sessions/store.ts`
- 每个会话密钥映射到一个 `SessionEntry`
- 对话记录单独存储在 SessionManager 文件中

**更新机制:**
- 原子性更新通过 `updateSessionStore()` 和 `updateSessionStoreEntry()` 完成
- 使用 JSON 文件存储,带有锁机制防止并发写入冲突

**代码示例:**

```typescript
// src/config/sessions/store.ts
export async function updateSessionStoreEntry(
  sessionKey: string,
  updates: Partial<SessionEntry>,
  config: OpenClawConfig
): Promise<void> {
  const store = await loadSessionStore(config);
  const existing = store[sessionKey] || {};
  store[sessionKey] = {
    ...existing,
    ...updates,
    updatedAt: Date.now()
  };
  await saveSessionStore(store, config);
}
```

---

## 5. 中间件与预处理管道

### 5.1 应用顺序

Gateway 在调用 Agent 之前,按以下顺序应用中间件:

**1. 上下文最终化** (`finalizeInboundContext`)
- 规范化所有上下文字段
- 解析 Body 变体 (Body, RawBody, BodyForCommands)

```typescript
// src/auto-reply/dispatch-from-config.ts
const finalized = finalizeInboundContext(ctx, cfg);
```

**2. 媒体理解** (`applyMediaUnderstanding`)
- 处理图片附件
- 提取 OCR/视觉数据
- 注入到上下文中

**3. 链接理解** (`applyLinkUnderstanding`)
- 提取 URL 和元数据
- 添加到入站上下文

**4. 命令授权** (`resolveCommandAuthorization`)
- 检查发送者权限
- 验证提升指令

**5. Body 预处理** (`applySessionHints`)
- 追加会话状态提示
- 如果上次运行失败,添加中止指示器

**6. 系统事件前置** (`prependSystemEvents`)
- 添加成员变更事件
- 注入系统公告

**7. 技能快照** (`ensureSkillSnapshot`)
- 构建可用技能提示
- 缓存技能状态

**8. 群组介绍注入** (针对群聊)
- 前置群组激活消息

### 5.2 预处理代码示例

```typescript
// src/auto-reply/reply/dispatch-from-config.ts
async function runPreparedReply(params) {
  // 1. 媒体理解
  await applyMediaUnderstanding(finalized, cfg);

  // 2. 链接理解
  await applyLinkUnderstanding(finalized, cfg);

  // 3. 命令授权
  const authResult = await resolveCommandAuthorization(finalized, cfg);
  if (!authResult.authorized) {
    return { unauthorized: true };
  }

  // 4. 会话初始化
  const sessionCtx = await initSessionState(finalized, cfg);

  // 5. 技能快照
  await ensureSkillSnapshot(sessionCtx, cfg);

  // 6. 运行 Agent
  return await runReplyAgent({
    ctx: finalized,
    sessionCtx,
    cfg,
    opts
  });
}
```

---

## 6. Agent 调用执行

### 6.1 主要 Agent 运行器

**文件位置:** `src/agents/pi-embedded-runner/run.ts`

**函数签名:**

```typescript
export async function runEmbeddedPiAgent(
  params: RunEmbeddedPiAgentParams,
): Promise<EmbeddedPiRunResult>
```

**参数类型:**

```typescript
type RunEmbeddedPiAgentParams = {
  sessionKey: string;
  agentId: string;
  config: OpenClawConfig;
  message: string;
  images?: ImageContent[];
  abortSignal?: AbortSignal;
  onBlockReply?: (payload: ReplyPayload) => void;
  onToolResult?: (payload: ReplyPayload) => void;
  sessionCtx: TemplateContext;
  // ... 其他参数
};
```

### 6.2 Agent 运行循环

**文件位置:** `src/auto-reply/reply/agent-runner-execution.ts`

**主要执行步骤:**

```typescript
export async function runAgentTurnWithFallback(params: {
  commandBody: string;
  followupRun: FollowupRun;
  sessionCtx: TemplateContext;
  opts?: GetReplyOptions;
  typingSignals: TypingSignaler;
  blockReplyPipeline: BlockReplyPipeline | null;
  blockStreamingEnabled: boolean;
  // ... 其他参数
}): Promise<AgentRunLoopResult>
```

### 6.3 执行流程图

```
循环:
├─ 尝试: runEmbeddedAttempt(...)
│  ├─ 模型解析 (provider/model)
│  ├─ 认证配置轮转
│  ├─ SessionManager 初始化
│  ├─ 消息追加
│  ├─ Agent API 调用
│  │  ├─ 工具调用处理
│  │  ├─ 工具结果处理
│  │  └─ 流式事件 (onBlockReply, onPartialReply)
│  └─ 会话更新
│
├─ 错误回退:
│  ├─ 瞬态 HTTP 错误 (带退避重试)
│  ├─ 上下文溢出 (压缩重试)
│  ├─ 认证错误 (轮转到下一个配置)
│  ├─ 速率限制 (冷却配置)
│  └─ 其他错误 (不重试)
│
└─ 返回: EmbeddedPiRunResult
```

### 6.4 重试逻辑代码示例

```typescript
// src/auto-reply/reply/agent-runner-execution.ts
let attemptCount = 0;
const maxAttempts = 3;

while (attemptCount < maxAttempts) {
  try {
    const result = await runEmbeddedAttempt({
      sessionKey,
      agentId,
      message: commandBody,
      sessionCtx,
      opts
    });

    return { success: true, result };

  } catch (error) {
    attemptCount++;

    if (isContextOverflowError(error)) {
      // 尝试压缩会话
      await compactSession(sessionKey);
      continue; // 重试
    }

    if (isAuthAssistantError(error)) {
      // 轮转到下一个认证配置
      const nextProfile = await rotateAuthProfile();
      if (nextProfile) continue;
    }

    if (isRateLimitAssistantError(error)) {
      // 标记当前配置冷却
      await markProfileCooldown(currentProfile);
      const nextProfile = await getNextActiveProfile();
      if (nextProfile) continue;
    }

    if (isTransientHttpError(error) && attemptCount < maxAttempts) {
      // HTTP 5xx 错误,退避重试
      await sleep(2500);
      continue;
    }

    // 其他错误,不重试
    throw error;
  }
}
```

---

## 7. 流式响应与分块回复

### 7.1 Block Reply Pipeline

**文件位置:** `src/auto-reply/reply/block-reply-pipeline.ts`

**类型定义:**

```typescript
export type BlockReplyPipeline = {
  enqueue: (payload: ReplyPayload) => void;           // 入队
  flush: (options?: { force?: boolean }) => Promise<void>; // 刷新
  stop: () => void;                                   // 停止
  hasBuffered: () => boolean;                         // 是否有缓冲
  didStream: () => boolean;                           // 是否已流式传输
  isAborted: () => boolean;                           // 是否已中止
  hasSentPayload: (payload: ReplyPayload) => boolean; // 是否已发送
};
```

### 7.2 流式回调集成

**创建 Pipeline:**

```typescript
// src/auto-reply/reply/agent-runner-execution.ts
const blockReplyPipeline =
  blockStreamingEnabled && opts?.onBlockReply
    ? createBlockReplyPipeline({
        onBlockReply: opts.onBlockReply,
        timeoutMs: blockReplyTimeoutMs,
        coalescing: resolveBlockStreamingCoalescing(...),
        buffer: createAudioAsVoiceBuffer(...)
      })
    : null;
```

### 7.3 流式处理流程

```
1. Agent 发出流式文本增量 (text deltas)
    ↓
2. Pipeline 合并为有意义的块 (段落/换行/句子)
    ↓
3. 调用 onBlockReply() 回调,传递每个块
    ↓
4. Gateway 收集块并发送到通道
    ↓
5. 最终回复批处理剩余块
```

### 7.4 合并策略 (Coalescing)

**配置选项:**

```typescript
type BlockStreamingCoalescing =
  | "paragraph"     // 段落边界合并
  | "newline"       // 换行符合并
  | "sentence"      // 句子边界合并
  | "none";         // 不合并

// 分块参数
type BlockReplyChunking = {
  minChars: number;           // 最小字符数
  maxChars: number;           // 最大字符数
  breakPreference: "text_end" | "message_end"; // 断点偏好
};
```

**代码示例:**

```typescript
// src/auto-reply/reply/block-reply-pipeline.ts
function createBlockReplyPipeline(options: {
  onBlockReply: (payload: ReplyPayload) => Promise<void>;
  timeoutMs: number;
  coalescing: BlockStreamingCoalescing;
  buffer?: AudioAsVoiceBuffer;
}): BlockReplyPipeline {
  let textBuffer = "";
  let sentPayloads = new Set<ReplyPayload>();

  return {
    enqueue: (payload) => {
      textBuffer += payload.text || "";

      // 检查是否应刷新
      if (shouldFlush(textBuffer, options.coalescing)) {
        void flush();
      }
    },

    flush: async (opts) => {
      if (!textBuffer) return;

      const payload = { text: textBuffer };
      textBuffer = "";
      sentPayloads.add(payload);

      await options.onBlockReply(payload);
    },

    // ... 其他方法
  };
}
```

### 7.5 超时处理

```typescript
// src/auto-reply/reply/agent-runner-execution.ts
const withTimeout = async <T>(
  promise: Promise<T>,
  timeoutMs: number,
  timeoutError: Error,
): Promise<T> => {
  if (!timeoutMs || timeoutMs <= 0) return promise;

  let timer: NodeJS.Timeout;
  const timeoutPromise = new Promise<never>((_, reject) => {
    timer = setTimeout(() => reject(timeoutError), timeoutMs);
  });

  try {
    return await Promise.race([promise, timeoutPromise]);
  } finally {
    clearTimeout(timer!);
  }
};
```

---

## 8. 消息队列机制

### 8.1 队列模式

**文件位置:** `src/auto-reply/reply/queue/types.ts`

**模式定义:**

```typescript
type QueueMode =
  | "steer"              // 在后台通道入队
  | "followup"           // 等待当前运行,然后入队
  | "collect"            // 缓冲消息,稍后处理
  | "steer-backlog"      // Steer + 保留积压
  | "steer+backlog"      // Steer 和积压
  | "queue"              // FIFO 队列
  | "interrupt";         // 中断当前运行

type QueueSettings = {
  mode: QueueMode;
  debounceMs?: number;   // 收集延迟 (毫秒)
  cap?: number;          // 最大队列长度 (超出则丢弃)
  dropPolicy?: "old" | "new" | "summarize"; // 丢弃策略
};
```

### 8.2 入队机制

**文件位置:** `src/auto-reply/reply/queue/enqueue.ts`

**函数签名:**

```typescript
export function enqueueFollowupRun(
  queueKey: string,
  followupRun: FollowupRun,
  resolvedQueue: QueueSettings
): void
```

**代码示例:**

```typescript
// src/auto-reply/reply/queue/enqueue.ts
const FOLLOWUP_QUEUES = new Map<string, FollowupQueue>();

export function enqueueFollowupRun(
  queueKey: string,
  followupRun: FollowupRun,
  resolvedQueue: QueueSettings
): void {
  let queue = FOLLOWUP_QUEUES.get(queueKey);

  if (!queue) {
    queue = {
      items: [],
      processing: false,
      settings: resolvedQueue
    };
    FOLLOWUP_QUEUES.set(queueKey, queue);
  }

  // 检查容量
  if (resolvedQueue.cap && queue.items.length >= resolvedQueue.cap) {
    if (resolvedQueue.dropPolicy === "old") {
      queue.items.shift(); // 丢弃最旧的
    } else if (resolvedQueue.dropPolicy === "new") {
      return; // 丢弃新消息
    }
  }

  queue.items.push(followupRun);

  // 触发处理
  void processQueue(queueKey);
}
```

### 8.3 通道级并发控制

**文件位置:** `src/process/command-queue.ts`

**通道类型:**

```typescript
type CommandLane =
  | "global"              // 全局顺序命令处理
  | `session:${string}`   // 每会话顺序
  | `agent:${string}`;    // 每 Agent 并发控制
```

**入队函数:**

```typescript
enqueueCommandInLane(lane: CommandLane, task: () => Promise<void>, opts?: {
  debounceMs?: number;
  timeout?: number;
  onSettled?: () => void;
})
```

**代码示例:**

```typescript
// src/process/command-queue.ts
const LANE_QUEUES = new Map<CommandLane, Promise<void>>();

export function enqueueCommandInLane(
  lane: CommandLane,
  task: () => Promise<void>,
  opts?: EnqueueOptions
): void {
  const currentChain = LANE_QUEUES.get(lane) || Promise.resolve();

  const newChain = currentChain
    .then(async () => {
      if (opts?.debounceMs) {
        await sleep(opts.debounceMs);
      }

      if (opts?.timeout) {
        return await withTimeout(task(), opts.timeout, new Error("Task timeout"));
      }

      return await task();
    })
    .catch((err) => {
      logger.error(`Lane task failed: lane=${lane}`, err);
    })
    .finally(() => {
      opts?.onSettled?.();
    });

  LANE_QUEUES.set(lane, newChain);
}
```

---

## 9. 错误处理与超时机制

### 9.1 错误分类

**文件位置:** `src/agents/pi-embedded-helpers.ts`

**错误检测函数:**

```typescript
// 认证失败
isAuthAssistantError(error): boolean

// 账单问题
isBillingAssistantError(error): boolean

// 速率限制
isRateLimitAssistantError(error): boolean

// 上下文窗口超出
isContextOverflowError(error): boolean
isLikelyContextOverflowError(error): boolean

// 压缩失败
isCompactionFailureError(error): boolean

// 超时
isTimeoutErrorMessage(error): boolean

// 瞬态 HTTP 错误
isTransientHttpError(error): boolean
```

### 9.2 回退路由逻辑

```
认证错误 → 尝试下一个认证配置
速率限制 → 标记配置冷却,尝试下一个
上下文溢出 → 触发压缩 → 使用相同配置重试
瞬态 HTTP 错误 → 2.5秒退避重试
压缩失败 → 重置会话 → 新会话 ID
```

### 9.3 错误处理代码示例

```typescript
// src/auto-reply/reply/agent-runner-execution.ts
async function handleAgentError(
  error: Error,
  context: ErrorContext
): Promise<ErrorHandlingResult> {

  // 1. 认证错误 - 轮转配置
  if (isAuthAssistantError(error)) {
    const nextProfile = await rotateToNextAuthProfile(context.currentProfile);
    if (nextProfile) {
      return { retry: true, profile: nextProfile };
    }
    return { retry: false, reason: "no_available_auth_profiles" };
  }

  // 2. 速率限制 - 冷却并轮转
  if (isRateLimitAssistantError(error)) {
    await markProfileCooldown(context.currentProfile, {
      cooldownUntil: Date.now() + 60_000 // 1分钟冷却
    });

    const nextProfile = await getNextActiveProfile();
    if (nextProfile) {
      return { retry: true, profile: nextProfile };
    }
    return { retry: false, reason: "all_profiles_rate_limited" };
  }

  // 3. 上下文溢出 - 压缩重试
  if (isContextOverflowError(error)) {
    const compacted = await compactSession(context.sessionKey);
    if (compacted) {
      return { retry: true, profile: context.currentProfile };
    }

    // 压缩失败 - 重置会话
    const reset = await resetSessionAfterCompactionFailure(
      context.sessionKey,
      "compaction_failed"
    );

    if (reset) {
      return { retry: true, profile: context.currentProfile, sessionReset: true };
    }

    return { retry: false, reason: "session_reset_failed" };
  }

  // 4. 瞬态 HTTP 错误 - 退避重试
  if (isTransientHttpError(error) && context.attemptCount < 3) {
    await sleep(2500);
    return { retry: true, profile: context.currentProfile };
  }

  // 5. 其他错误 - 不重试
  return { retry: false, reason: "unrecoverable_error" };
}
```

### 9.4 超时配置

**文件位置:** `src/agents/timeout.ts`

```typescript
export function resolveAgentTimeoutMs(opts: {
  cfg: OpenClawConfig;
}): number {
  const provider = opts.cfg.agent?.provider;

  // 不同提供商的默认超时
  const defaults = {
    anthropic: 300_000,    // 5 分钟
    openai: 240_000,       // 4 分钟
    google: 180_000,       // 3 分钟
  };

  return opts.cfg.agent?.timeoutMs
    || defaults[provider]
    || 300_000;
}
```

**Block Reply 超时:**

```typescript
// src/auto-reply/reply/agent-runner-execution.ts
const blockReplyTimeoutMs = opts?.blockReplyTimeoutMs ?? 15_000;
```

---

## 10. 并发消息处理

### 10.1 架构概览

OpenClaw 使用三级并发控制:

1. **全局通道** (`global`): 顺序处理全局命令
2. **每会话通道** (`session:${sessionKey}`): 会话级顺序化
3. **每 Agent 通道** (`agent:${agentId}`): Agent 级并发控制

### 10.2 Reply Dispatcher 序列化

**文件位置:** `src/auto-reply/reply/reply-dispatcher.ts`

**实现逻辑:**

```typescript
export function createReplyDispatcher(options: ReplyDispatcherOptions): ReplyDispatcher {
  let sendChain: Promise<void> = Promise.resolve();
  let pending = 1; // 初始预留
  let completeCalled = false;
  const queuedCounts = { tool: 0, block: 0, final: 0 };

  const enqueue = (kind: ReplyDispatchKind, payload: ReplyPayload): boolean => {
    if (completeCalled && kind !== "final") {
      return false; // 完成后不再接受非最终回复
    }

    queuedCounts[kind] += 1;
    pending += 1;

    sendChain = sendChain
      .then(async () => {
        const normalized = await normalizeReplyPayload(payload, options);
        await deliver(normalized, { kind });
      })
      .catch((err) => {
        options.onError?.(err, { kind });
      })
      .finally(() => {
        pending -= 1;
        queuedCounts[kind] -= 1;

        if (pending === 0) {
          options.unregister?.();
          options.onIdle?.();
        }
      });

    return true;
  };

  return {
    sendToolResult: (payload) => enqueue("tool", payload),
    sendBlockReply: (payload) => enqueue("block", payload),
    sendFinalReply: (payload) => enqueue("final", payload),
    waitForIdle: () => sendChain,
    getQueuedCounts: () => ({ ...queuedCounts }),
    markComplete: () => {
      completeCalled = true;
      pending -= 1; // 释放初始预留
    },
  };
}
```

### 10.3 分发顺序保证

**顺序:**
1. 工具结果 (Tool Results)
2. 分块回复 (Block Replies)
3. 最终回复 (Final Reply)

**人类延迟:**

在分块回复之间插入延迟,模拟人类打字速度:

```typescript
// src/auto-reply/reply/reply-dispatcher.ts
const humanDelayMs = options.humanDelay ?? {
  min: 800,
  max: 2500
};

if (kind === "block" && previousKind === "block") {
  const delay = randomInt(humanDelayMs.min, humanDelayMs.max);
  await sleep(delay);
}
```

### 10.4 活动运行跟踪

**WebChat 示例:**

**文件位置:** `src/gateway/server-chat.ts`

```typescript
function createChatRunRegistry() {
  const activeRuns = new Map<string, ChatRunState>();

  return {
    register: (sessionId: string, runId: string) => {
      activeRuns.set(sessionId, {
        runId,
        buffers: [],
        deltaSentAt: Date.now()
      });
    },

    unregister: (sessionId: string) => {
      activeRuns.delete(sessionId);
    },

    getState: (sessionId: string) => {
      return activeRuns.get(sessionId);
    },

    getActiveCount: () => activeRuns.size
  };
}
```

---

## 11. 响应规范化与交付

### 11.1 Payload 规范化

**文件位置:** `src/auto-reply/reply/normalize-reply.ts`

**函数签名:**

```typescript
export function normalizeReplyPayload(
  payload: ReplyPayload,
  options: {
    responsePrefix?: string;
    responsePrefixContext?: ResponsePrefixContext;
    onHeartbeatStrip?: () => void;
    onSkip?: (reason: string) => void;
  }
): ReplyPayload
```

**处理步骤:**

1. **前缀添加:** 如果配置了 `responsePrefix`,添加到文本开头
2. **心跳标记移除:** 移除 `HEARTBEAT_OK` 等内部标记
3. **空消息过滤:** 跳过空文本和媒体
4. **模板插值:** 应用响应前缀模板

**代码示例:**

```typescript
// src/auto-reply/reply/normalize-reply.ts
export function normalizeReplyPayload(
  payload: ReplyPayload,
  options: NormalizeOptions
): ReplyPayload {
  let { text, mediaUrl, mediaUrls } = payload;

  // 1. 移除心跳标记
  if (text && text.includes("HEARTBEAT_OK")) {
    text = text.replace(/HEARTBEAT_OK/g, "").trim();
    options.onHeartbeatStrip?.();
  }

  // 2. 检查是否为空
  if (!text && !mediaUrl && (!mediaUrls || mediaUrls.length === 0)) {
    options.onSkip?.("empty_payload");
    return { ...payload, text: "" };
  }

  // 3. 应用响应前缀
  if (options.responsePrefix && text) {
    const interpolated = interpolateResponsePrefix(
      options.responsePrefix,
      options.responsePrefixContext
    );
    text = interpolated + text;
  }

  return {
    ...payload,
    text
  };
}
```

### 11.2 响应前缀模板

**文件位置:** `src/auto-reply/reply/response-prefix-template.ts`

**上下文类型:**

```typescript
type ResponsePrefixContext = {
  provider?: string;         // 例: "anthropic"
  model?: string;            // 例: "claude-sonnet-4"
  thinkLevel?: string;       // 例: "medium"
  verboseLevel?: string;     // 例: "default"
  elevatedLevel?: string;    // 例: "elevated"
  sessionId?: string;
  // ... 其他变量
};
```

**模板示例:**

```
配置: responsePrefix: "[{{model}}|{{thinkLevel}}] "
结果: "[claude-sonnet-4|medium] 这是回复内容..."
```

### 11.3 交付到 ReplyDispatcher

**调用序列:**

```typescript
// 工具结果立即发送
dispatcher.sendToolResult({
  text: "执行 ls 命令:\n文件列表..."
});

// 流式内容块
dispatcher.sendBlockReply({
  text: "让我分析一下..."
});

dispatcher.sendBlockReply({
  text: "根据代码结构..."
});

// 最终回复
dispatcher.sendFinalReply({
  text: "分析完成。"
});
```

**Dispatcher 内部交付:**

```typescript
// src/auto-reply/reply/reply-dispatcher.ts
async function deliver(
  payload: ReplyPayload,
  context: { kind: ReplyDispatchKind }
): Promise<void> {
  // 调用用户提供的 deliver 函数
  await options.deliver(payload, context);

  // 或默认路由
  await routeReply({
    payload,
    ctx: options.messageContext,
    config: options.config
  });
}
```

---

## 12. 核心函数签名

### 12.1 消息接收

```typescript
// src/auto-reply/dispatch.ts
async function dispatchInboundMessage(params: {
  ctx: MsgContext | FinalizedMsgContext;
  cfg: OpenClawConfig;
  dispatcher: ReplyDispatcher;
  replyOptions?: Omit<GetReplyOptions, "onToolResult" | "onBlockReply">;
  replyResolver?: typeof getReplyFromConfig;
}): Promise<DispatchInboundResult>
```

### 12.2 Agent 调用

```typescript
// src/auto-reply/reply/get-reply.ts
async function getReplyFromConfig(
  ctx: MsgContext,
  opts?: GetReplyOptions,
  configOverride?: OpenClawConfig,
): Promise<ReplyPayload | ReplyPayload[] | undefined>
```

### 12.3 Agent 执行

```typescript
// src/agents/pi-embedded-runner/run.ts
async function runEmbeddedPiAgent(
  params: RunEmbeddedPiAgentParams,
): Promise<EmbeddedPiRunResult>

type RunEmbeddedPiAgentParams = {
  sessionKey: string;
  agentId: string;
  config: OpenClawConfig;
  message: string;
  images?: ImageContent[];
  abortSignal?: AbortSignal;
  onBlockReply?: (payload: ReplyPayload) => void;
  onToolResult?: (payload: ReplyPayload) => void;
  sessionCtx: TemplateContext;
  // ... 更多参数
};

type EmbeddedPiRunResult = {
  success: boolean;
  text?: string;
  error?: Error;
  inputTokens?: number;
  outputTokens?: number;
  sessionId?: string;
  // ... 更多字段
};
```

### 12.4 Dispatcher 创建

```typescript
// src/auto-reply/reply/reply-dispatcher.ts
function createReplyDispatcher(options: ReplyDispatcherOptions): ReplyDispatcher {
  return {
    sendToolResult: (payload: ReplyPayload) => boolean,
    sendBlockReply: (payload: ReplyPayload) => boolean,
    sendFinalReply: (payload: ReplyPayload) => boolean,
    waitForIdle: () => Promise<void>,
    getQueuedCounts: () => Record<ReplyDispatchKind, number>,
    markComplete: () => void,
  };
}

type ReplyDispatcherOptions = {
  deliver: (payload: ReplyPayload, context: DeliverContext) => Promise<void>;
  onError?: (error: Error, context: ErrorContext) => void;
  onIdle?: () => void;
  unregister?: () => void;
  messageContext: MsgContext;
  config: OpenClawConfig;
  humanDelay?: { min: number; max: number };
  // ... 更多选项
};
```

### 12.5 分发流程

```typescript
// src/auto-reply/reply/dispatch-from-config.ts
async function dispatchReplyFromConfig(params: {
  ctx: FinalizedMsgContext;
  cfg: OpenClawConfig;
  dispatcher: ReplyDispatcher;
  replyOptions?: Omit<GetReplyOptions, "onToolResult" | "onBlockReply">;
  replyResolver?: typeof getReplyFromConfig;
}): Promise<DispatchFromConfigResult>
```

---

## 13. 数据结构与接口

### 13.1 消息上下文 (MsgContext)

**文件位置:** `src/auto-reply/templating.ts`

```typescript
type MsgContext = {
  // 基本身份
  From: string;                      // 发送者 ID
  To: string;                        // 接收者 ID
  Provider: string;                  // 通道提供商 (例: "telegram")
  Surface: string;                   // 通道变体 (例: "telegram-bot")

  // 消息内容
  Body: string;                      // 原始消息文本
  RawBody?: string;                  // 处理前的文本
  BodyForCommands?: string;          // 用于命令解析的文本

  // 会话标识
  SessionKey: string;                // 会话标识符
  MessageSid: string;                // 消息 ID

  // 媒体附件
  MediaPath?: string;                // 附件路径
  MediaUrl?: string;                 // 媒体 URL
  MediaContentType?: string;         // MIME 类型

  // 聊天类型
  ChatType: "dm" | "group" | "channel";

  // 命令元数据
  CommandSource?: "text" | "native" | "webhook";
  CommandTargetSessionKey?: string;  // 目标会话 (用于跨会话命令)

  // 群组相关
  WasMentioned?: boolean;            // Bot 是否在群组中被提及
  GroupMembers?: string[];           // 群组成员列表
  GroupName?: string;                // 群组名称

  // 跨通道
  OriginatingChannel?: string;       // 跨通道来源
  OriginatingTo?: string;            // 跨通道接收者

  // 时间戳
  ReceivedAt?: number;               // 接收时间

  // 其他 30+ 字段...
};
```

### 13.2 Reply Dispatcher

```typescript
type ReplyDispatcher = {
  // 发送工具结果
  sendToolResult: (payload: ReplyPayload) => boolean;

  // 发送流式分块回复
  sendBlockReply: (payload: ReplyPayload) => boolean;

  // 发送最终回复
  sendFinalReply: (payload: ReplyPayload) => boolean;

  // 等待所有消息发送完成
  waitForIdle: () => Promise<void>;

  // 获取队列中的消息数量
  getQueuedCounts: () => Record<ReplyDispatchKind, number>;

  // 标记为完成 (不再接受新的 block/tool 回复)
  markComplete: () => void;
};

type ReplyDispatchKind = "tool" | "block" | "final";
```

### 13.3 GetReplyOptions (完整版)

```typescript
export type GetReplyOptions = {
  // 运行标识
  runId?: string;

  // 取消控制
  abortSignal?: AbortSignal;

  // 媒体附件
  images?: ImageContent[];

  // ===== 生命周期回调 =====
  onAgentRunStart?: (runId: string) => void;
  onReplyStart?: () => Promise<void> | void;
  onTypingCleanup?: () => void;
  onTypingController?: (typing: TypingController) => void;

  // ===== 流式响应回调 =====
  // 部分回复 (Agent 生成的每个文本块)
  onPartialReply?: (payload: ReplyPayload) => Promise<void> | void;

  // 推理流 (Claude 的思考过程)
  onReasoningStream?: (payload: ReplyPayload) => Promise<void> | void;

  // 分块回复 (合并后的文本块,发送给用户)
  onBlockReply?: (payload: ReplyPayload, context?: BlockReplyContext) => Promise<void> | void;

  // 工具结果
  onToolResult?: (payload: ReplyPayload) => Promise<void> | void;

  // 模型选择
  onModelSelected?: (ctx: ModelSelectedContext) => void;

  // ===== 控制选项 =====
  disableBlockStreaming?: boolean;           // 禁用分块流式传输
  blockReplyTimeoutMs?: number;              // 分块回复超时 (默认 15秒)
  skillFilter?: string[];                    // 技能过滤器
  hasRepliedRef?: { value: boolean };        // 是否已回复引用

  // ===== 心跳运行 =====
  isHeartbeat?: boolean;                     // 是否为心跳运行
  heartbeatModelOverride?: string;           // 心跳模型覆盖
};
```

### 13.4 ReplyPayload (完整版)

```typescript
export type ReplyPayload = {
  // 文本内容
  text?: string;

  // 媒体
  mediaUrl?: string;                         // 单个媒体 URL
  mediaUrls?: string[];                      // 多个媒体 URL

  // 回复控制
  replyToId?: string;                        // 回复目标消息 ID
  replyToTag?: boolean;                      // 是否标记回复 (提及用户)
  replyToCurrent?: boolean;                  // 是否回复当前消息

  // 音频控制
  audioAsVoice?: boolean;                    // 音频是否作为语音消息

  // 错误标识
  isError?: boolean;                         // 是否为错误消息

  // 通道特定数据
  channelData?: Record<string, unknown>;     // 通道特定的额外数据

  // 内部元数据 (不发送到通道)
  _metadata?: {
    toolName?: string;
    toolInput?: unknown;
    thinkingContent?: string;
    // ... 其他元数据
  };
};
```

### 13.5 SessionEntry (完整版)

```typescript
export type SessionEntry = {
  // ===== 核心标识 =====
  sessionId: string;                         // UUID,对话记录标识
  sessionKey?: string;                       // 会话密钥 (可选)
  updatedAt: number;                         // 最后更新时间戳
  createdAt?: number;                        // 创建时间戳

  // ===== 文件路径 =====
  sessionFile?: string;                      // 对话记录文件路径

  // ===== 会话关系 =====
  spawnedBy?: string;                        // 父会话密钥
  childSessions?: string[];                  // 子会话列表

  // ===== 状态标志 =====
  systemSent?: boolean;                      // 系统提示是否已发送
  abortedLastRun?: boolean;                  // 上次运行是否中止

  // ===== 聊天配置 =====
  chatType?: SessionChatType;                // "dm" | "group" | "channel"

  // ===== Agent 配置 =====
  thinkingLevel?: string;                    // 思考等级
  verboseLevel?: string;                     // 详细程度
  reasoningLevel?: string;                   // 推理等级
  elevatedLevel?: string;                    // 提升权限等级
  ttsAuto?: TtsAutoMode;                     // TTS 自动模式
  execHost?: string;                         // 执行主机
  modelOverride?: string;                    // 模型覆盖
  authProfileOverride?: string;              // 认证配置覆盖

  // ===== 队列配置 =====
  queueMode?: "steer" | "followup" | "collect" | "queue" | "interrupt";
  queueDebounceMs?: number;                  // 队列防抖延迟
  queueCap?: number;                         // 队列容量上限
  queueDropPolicy?: "old" | "new" | "summarize";

  // ===== 使用量跟踪 =====
  inputTokens?: number;
  outputTokens?: number;
  totalTokens?: number;
  messageCount?: number;                     // 消息计数

  // ===== 技能快照 =====
  skillsSnapshot?: SessionSkillSnapshot;

  // ===== 跨通道元数据 =====
  origin?: SessionOrigin;

  // ===== 其他元数据 =====
  metadata?: Record<string, unknown>;
};
```

---

## 总结

### Gateway-Agent 交互的关键要点

1. **消息流转路径:**
   - 通道 → `dispatchInboundMessage` → `getReplyFromConfig` → `runEmbeddedPiAgent` → Agent API
   - 响应: Agent → `onBlockReply` / `sendFinalReply` → `ReplyDispatcher` → 通道

2. **核心接口:**
   - `getReplyFromConfig`: Gateway 调用 Agent 的主入口
   - `GetReplyOptions`: 提供流式回调和控制选项
   - `ReplyPayload`: 标准化响应格式

3. **会话管理:**
   - 每个会话密钥映射到 `SessionEntry`
   - 对话记录单独存储在 SessionManager 文件
   - 支持跨通道会话和父子会话关系

4. **流式响应:**
   - `BlockReplyPipeline` 合并文本块
   - 支持多种合并策略 (paragraph, newline, sentence)
   - `ReplyDispatcher` 保证发送顺序 (tool → block → final)

5. **错误处理:**
   - 分类错误类型,采用不同回退策略
   - 认证错误 → 轮转配置
   - 上下文溢出 → 压缩会话
   - 速率限制 → 冷却并轮转

6. **并发控制:**
   - 三级通道: 全局、会话、Agent
   - `ReplyDispatcher` 序列化单个会话的所有回复
   - 队列模式支持多种并发策略

7. **预处理管道:**
   - 8 个中间件步骤,包括媒体理解、命令授权、技能快照等
   - 确保 Agent 接收到完整且规范化的上下文

8. **消息队列:**
   - 支持 7 种队列模式 (steer, followup, collect, queue, interrupt 等)
   - 防抖、容量限制、丢弃策略
   - 通道级并发控制

这个架构实现了健壮、并发、支持流式传输的 Agent 交互,具备全面的错误处理和跨多种消息通道的会话管理能力。

---

**文档版本:** 1.0
**最后更新:** 2026-02-19
**相关文档:**
- [Agent Loop 分析](./agent-loop-analysis.md)
- [WhatsApp 集成分析](./whatsapp-integration-analysis.md)
- [消息平台对比](./messaging-platforms-comparison.md)
- [Gateway 和 Chat Channels 分析](./gateway-and-channels-analysis.md)
