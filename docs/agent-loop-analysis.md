# Agent Loop 源代码分析

> 本文档详细分析了 OpenClaw 中 Agent Loop（代理循环）的源代码实现

## 目录

1. [概述](#概述)
2. [核心架构](#核心架构)
3. [主要文件结构](#主要文件结构)
4. [嵌套循环架构](#嵌套循环架构)
5. [执行流程](#执行流程)
6. [错误处理与容错](#错误处理与容错)
7. [状态管理](#状态管理)
8. [关键代码解析](#关键代码解析)

---

## 概述

OpenClaw 的 Agent Loop 是一个多层嵌套的执行系统，负责管理 AI 代理的完整生命周期。它实现了：

- **多层重试机制**：认证配置轮换、模型降级、自动压缩
- **流式处理**：实时工具调用和消息流
- **错误恢复**：上下文溢出、速率限制、认证失败的自动处理
- **插件系统**：可扩展的钩子机制

## 核心架构

```
┌─────────────────────────────────────────────────────────────┐
│                   runEmbeddedPiAgent                        │
│                    (主入口函数)                              │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  认证配置循环 (Auth Profile Loop)                     │ │
│  │  • 遍历可用的认证配置                                 │ │
│  │  • 跳过冷却期内的配置                                 │ │
│  │  • 认证失败时自动切换下一个配置                       │ │
│  │  ┌─────────────────────────────────────────────────┐ │ │
│  │  │  主执行循环 (Main Agent Loop)                    │ │ │
│  │  │  • while(true) 无限循环直到成功或耗尽选项        │ │ │
│  │  │  ┌───────────────────────────────────────────┐  │ │ │
│  │  │  │  runEmbeddedAttempt                       │  │ │ │
│  │  │  │  (单次执行尝试)                           │  │ │ │
│  │  │  │  ┌─────────────────────────────────────┐ │  │ │ │
│  │  │  │  │  subscribeEmbeddedPiSession         │ │  │ │ │
│  │  │  │  │  (事件订阅与流处理)                 │ │  │ │ │
│  │  │  │  │  • 工具执行循环                     │ │  │ │ │
│  │  │  │  │  • 消息流处理                       │ │  │ │ │
│  │  │  │  │  • 实时状态更新                     │ │  │ │ │
│  │  │  │  └─────────────────────────────────────┘ │  │ │ │
│  │  │  └───────────────────────────────────────────┘  │ │ │
│  │  │  • 错误分类与处理：                              │ │ │
│  │  │    - 上下文溢出 → 自动压缩 → 重试               │ │ │
│  │  │    - 认证失败 → 切换配置                         │ │ │
│  │  │    - 思考级别不支持 → 降级 → 重试               │ │ │
│  │  │    - 速率限制 → 标记冷却 → 切换配置             │ │ │
│  │  └─────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 主要文件结构

### 核心执行文件

| 文件路径 | 行数 | 主要功能 |
|---------|------|---------|
| `src/agents/pi-embedded-runner/run.ts` | ~950 | 主循环编排、认证轮换、重试逻辑 |
| `src/agents/pi-embedded-runner/run/attempt.ts` | ~1064 | 单次执行尝试、会话初始化、工具设置 |
| `src/agents/pi-embedded-runner/runs.ts` | ~145 | 活动运行跟踪、消息队列、中止控制 |
| `src/agents/pi-embedded-subscribe.ts` | ~500+ | 事件订阅处理、流式消息管理 |

### 关键支持文件

| 文件路径 | 功能 |
|---------|------|
| `src/auto-reply/reply/agent-runner-execution.ts` | 高层执行包装、模型降级 |
| `src/auto-reply/reply/agent-runner.ts` | 自动回复编排 |
| `src/agents/model-fallback.ts` | 模型降级逻辑 |
| `src/agents/pi-tools.before-tool-call.ts` | 工具调用钩子 |

## 嵌套循环架构

### 第一层：认证配置循环

```typescript
// src/agents/pi-embedded-runner/run.ts:382-409
while (profileIndex < profileCandidates.length) {
  const candidate = profileCandidates[profileIndex];

  // 跳过冷却期内的配置
  if (candidate && isProfileInCooldown(authStore, candidate)) {
    profileIndex += 1;
    continue;
  }

  // 应用认证配置
  await applyApiKeyInfo(profileCandidates[profileIndex]);
  break;
}
```

**功能**：
- 遍历所有可用的认证配置（API 密钥）
- 自动跳过冷却期内的配置
- 认证失败时自动切换到下一个配置
- 支持用户锁定特定配置

### 第二层：主执行循环

```typescript
// src/agents/pi-embedded-runner/run.ts:418+
while (true) {
  // 执行单次尝试
  const attempt = await runEmbeddedAttempt({
    sessionId: params.sessionId,
    provider,
    modelId,
    model,
    thinkLevel,
    // ... 其他参数
  });

  // 处理上下文溢出
  if (contextOverflowError) {
    if (overflowCompactionAttempts < MAX_OVERFLOW_COMPACTION_ATTEMPTS) {
      overflowCompactionAttempts++;
      await compactEmbeddedPiSessionDirect({...});
      continue; // 重试
    }
  }

  // 处理认证/速率限制失败
  if (authFailure || rateLimitFailure) {
    const rotated = await advanceAuthProfile();
    if (rotated) {
      continue; // 使用新配置重试
    }
    throw new FailoverError(...);
  }

  // 成功 - 退出循环
  return buildEmbeddedRunPayloads(...);
}
```

**功能**：
- `while(true)` 循环直到成功或耗尽所有恢复选项
- 自动处理多种错误场景
- 智能重试策略

### 第三层：工具执行循环

```typescript
// 在 pi-agent-core SDK 内部管理
// subscribeEmbeddedPiSession 监听事件
const subscription = subscribeEmbeddedPiSession({
  session: activeSession,
  runId: params.runId,
  hookRunner: getGlobalHookRunner(),
  // 事件处理器
  onToolResult: params.onToolResult,
  onBlockReply: params.onBlockReply,
  // ...
});
```

**功能**：
- 监听并处理代理生成的工具调用
- 流式处理代理的响应
- 管理思考块和最终响应的状态

## 执行流程

### 1. 初始化阶段

```typescript
// run.ts
const agentDir = params.agentDir ?? resolveOpenClawAgentDir();
const { model, error, authStorage, modelRegistry } = resolveModel(
  provider,
  modelId,
  agentDir,
  params.config
);

// 上下文窗口验证
const ctxGuard = evaluateContextWindowGuard({...});
if (ctxGuard.shouldBlock) {
  throw new FailoverError("Model context window too small");
}
```

### 2. 认证配置解析

```typescript
// run.ts:245-278
const authStore = ensureAuthProfileStore(agentDir);
const profileOrder = resolveAuthProfileOrder({
  cfg: params.config,
  store: authStore,
  provider,
  preferredProfile: preferredProfileId,
});

const profileCandidates = lockedProfileId
  ? [lockedProfileId]  // 用户锁定的配置
  : profileOrder.length > 0
    ? profileOrder      // 所有可用配置
    : [undefined];      // 回退到默认
```

### 3. 单次执行尝试 (runEmbeddedAttempt)

```typescript
// attempt.ts:207-1063
export async function runEmbeddedAttempt(
  params: EmbeddedRunAttemptParams
): Promise<EmbeddedRunAttemptResult> {

  // 1. 设置工作区和沙箱
  const sandbox = await resolveSandboxContext({...});
  const effectiveWorkspace = sandbox?.enabled
    ? sandbox.workspaceDir
    : resolvedWorkspace;

  // 2. 加载技能和启动文件
  const skillEntries = loadWorkspaceSkillEntries(effectiveWorkspace);
  const { bootstrapFiles, contextFiles } =
    await resolveBootstrapContextForRun({...});

  // 3. 创建工具
  const tools = createOpenClawCodingTools({
    exec: { ...params.execOverrides, elevated: params.bashElevated },
    sandbox,
    messageProvider: params.messageChannel,
    // ... 工具配置
  });

  // 4. 构建系统提示
  const appendPrompt = buildEmbeddedSystemPrompt({
    workspaceDir: effectiveWorkspace,
    defaultThinkLevel: params.thinkLevel,
    reasoningLevel: params.reasoningLevel,
    tools,
    // ... 系统提示参数
  });

  // 5. 初始化会话
  const sessionLock = await acquireSessionWriteLock({
    sessionFile: params.sessionFile
  });

  sessionManager = guardSessionManager(
    SessionManager.open(params.sessionFile),
    {...}
  );

  // 6. 创建代理会话
  ({ session } = await createAgentSession({
    model: params.model,
    thinkingLevel: mapThinkingLevel(params.thinkLevel),
    tools: builtInTools,
    customTools: allCustomTools,
    sessionManager,
    settingsManager,
  }));

  // 7. 订阅事件流
  const subscription = subscribeEmbeddedPiSession({
    session: activeSession,
    onToolResult: params.onToolResult,
    onBlockReply: params.onBlockReply,
    // ... 事件处理器
  });

  // 8. 发送提示并等待完成
  await activeSession.prompt(effectivePrompt, { images });
  await waitForCompactionRetry();

  // 9. 返回结果
  return {
    aborted,
    promptError,
    sessionIdUsed,
    assistantTexts,
    toolMetas,
    lastAssistant,
    // ...
  };
}
```

### 4. 事件订阅处理 (subscribeEmbeddedPiSession)

```typescript
// pi-embedded-subscribe.ts
export function subscribeEmbeddedPiSession(params) {
  const state = {
    assistantTexts: [],
    toolMetas: [],
    blockState: { thinking: false, final: false },
    // ... 状态追踪
  };

  // 创建事件处理器
  const context: EmbeddedPiSubscribeContext = {
    state,
    params,
    pushAssistantText,
    emitToolSummary,
    // ... 辅助函数
  };

  const eventHandler = createEmbeddedPiSessionEventHandler(context);

  // 监听代理事件
  session.agent.on("agent_start", eventHandler.handleAgentStart);
  session.agent.on("text_delta", eventHandler.handleTextDelta);
  session.agent.on("text_end", eventHandler.handleTextEnd);
  session.agent.on("tool_execution_start", eventHandler.handleToolExecutionStart);
  session.agent.on("tool_execution_end", eventHandler.handleToolExecutionEnd);
  session.agent.on("message_end", eventHandler.handleMessageEnd);
  // ...

  return {
    assistantTexts: state.assistantTexts,
    toolMetas: state.toolMetas,
    unsubscribe,
    waitForCompactionRetry,
    // ...
  };
}
```

## 错误处理与容错

### 1. 上下文溢出恢复

```typescript
// run.ts:503-667
if (contextOverflowError) {
  const overflowDiagId = createCompactionDiagId();

  // 尝试自动压缩（最多3次）
  if (!isCompactionFailure &&
      overflowCompactionAttempts < MAX_OVERFLOW_COMPACTION_ATTEMPTS) {
    overflowCompactionAttempts++;

    const compactResult = await compactEmbeddedPiSessionDirect({
      sessionId: params.sessionId,
      provider,
      model: modelId,
      trigger: "overflow",
      diagId: overflowDiagId,
    });

    if (compactResult.compacted) {
      autoCompactionCount += 1;
      continue; // 重试
    }
  }

  // 备用方案：截断过大的工具结果
  if (!toolResultTruncationAttempted) {
    const hasOversized = sessionLikelyHasOversizedToolResults({
      messages: attempt.messagesSnapshot,
      contextWindowTokens: ctxInfo.tokens,
    });

    if (hasOversized) {
      toolResultTruncationAttempted = true;
      const truncResult = await truncateOversizedToolResultsInSession({
        sessionFile: params.sessionFile,
        contextWindowTokens,
      });

      if (truncResult.truncated) {
        overflowCompactionAttempts = 0; // 重置计数器
        continue; // 重试
      }
    }
  }

  // 所有恢复尝试失败 - 返回错误
  return {
    payloads: [{
      text: "Context overflow: prompt too large for the model. " +
            "Try /reset or use a larger-context model.",
      isError: true,
    }],
    meta: { error: { kind: "context_overflow", message: errorText } }
  };
}
```

### 2. 认证失败与速率限制处理

```typescript
// run.ts:776-866
const authFailure = isAuthAssistantError(lastAssistant);
const rateLimitFailure = isRateLimitAssistantError(lastAssistant);
const failoverFailure = isFailoverAssistantError(lastAssistant);

if (authFailure || rateLimitFailure || failoverFailure || timedOut) {
  // 标记配置失败
  if (lastProfileId) {
    await markAuthProfileFailure({
      store: authStore,
      profileId: lastProfileId,
      reason: timedOut ? "timeout" : assistantFailoverReason,
      cfg: params.config,
    });
  }

  // 尝试切换到下一个配置
  const rotated = await advanceAuthProfile();
  if (rotated) {
    continue; // 使用新配置重试
  }

  // 如果配置了降级，抛出 FailoverError
  if (fallbackConfigured) {
    throw new FailoverError(message, {
      reason: assistantFailoverReason ?? "unknown",
      provider,
      model: modelId,
      profileId: lastProfileId,
    });
  }
}
```

### 3. 思考级别降级

```typescript
// run.ts:739-774
const fallbackThinking = pickFallbackThinkingLevel({
  message: errorText,
  attempted: attemptedThinking,
});

if (fallbackThinking) {
  log.warn(
    `unsupported thinking level for ${provider}/${modelId}; ` +
    `retrying with ${fallbackThinking}`
  );
  thinkLevel = fallbackThinking;
  continue; // 使用新思考级别重试
}
```

**思考级别降级顺序**：
- `extended` → `basic` → `off`
- 确保不会无限重试相同级别

## 状态管理

### 活动运行跟踪

```typescript
// runs.ts
const ACTIVE_EMBEDDED_RUNS = new Map<string, EmbeddedPiQueueHandle>();

export function setActiveEmbeddedRun(
  sessionId: string,
  handle: EmbeddedPiQueueHandle
) {
  ACTIVE_EMBEDDED_RUNS.set(sessionId, handle);
  logSessionStateChange({
    sessionId,
    state: "processing",
    reason: "run_started",
  });
}

export function clearActiveEmbeddedRun(
  sessionId: string,
  handle: EmbeddedPiQueueHandle
) {
  if (ACTIVE_EMBEDDED_RUNS.get(sessionId) === handle) {
    ACTIVE_EMBEDDED_RUNS.delete(sessionId);
    logSessionStateChange({
      sessionId,
      state: "idle",
      reason: "run_completed",
    });
    notifyEmbeddedRunEnded(sessionId);
  }
}
```

**功能**：
- 全局跟踪每个会话的活动运行
- 支持消息队列（在流式传输期间）
- 支持运行中止
- 等待运行完成的机制

### 消息队列机制

```typescript
// runs.ts:21-38
export function queueEmbeddedPiMessage(
  sessionId: string,
  text: string
): boolean {
  const handle = ACTIVE_EMBEDDED_RUNS.get(sessionId);

  if (!handle) return false;
  if (!handle.isStreaming()) return false;
  if (handle.isCompacting()) return false;

  void handle.queueMessage(text);
  return true;
}
```

**使用场景**：
- 在代理流式响应时排队用户消息
- 支持"中途转向"（steering）功能
- 避免在压缩期间排队

### 等待运行结束

```typescript
// runs.ts:71-103
export function waitForEmbeddedPiRunEnd(
  sessionId: string,
  timeoutMs = 15_000
): Promise<boolean> {
  if (!ACTIVE_EMBEDDED_RUNS.has(sessionId)) {
    return Promise.resolve(true);
  }

  return new Promise((resolve) => {
    const waiter = {
      resolve,
      timer: setTimeout(() => {
        // 超时处理
        waiters.delete(waiter);
        resolve(false);
      }, Math.max(100, timeoutMs)),
    };

    waiters.add(waiter);
    EMBEDDED_RUN_WAITERS.set(sessionId, waiters);
  });
}
```

## 关键代码解析

### 1. 使用量累加器

```typescript
// run.ts:77-160
type UsageAccumulator = {
  input: number;
  output: number;
  cacheRead: number;
  cacheWrite: number;
  total: number;
  // 仅最后一次 API 调用的缓存字段（非累加）
  lastCacheRead: number;
  lastCacheWrite: number;
  lastInput: number;
};

const mergeUsageIntoAccumulator = (
  target: UsageAccumulator,
  usage: ReturnType<typeof normalizeUsage>
) => {
  if (!hasUsageValues(usage)) return;

  target.input += usage.input ?? 0;
  target.output += usage.output ?? 0;
  target.cacheRead += usage.cacheRead ?? 0;
  target.cacheWrite += usage.cacheWrite ?? 0;
  target.total += usage.total ?? (/* 计算 */);

  // 仅跟踪最后一次调用的缓存字段
  // 用于准确的上下文大小报告
  target.lastCacheRead = usage.cacheRead ?? 0;
  target.lastCacheWrite = usage.cacheWrite ?? 0;
  target.lastInput = usage.input ?? 0;
};
```

**为什么需要 `lastCacheRead/Write`？**

在工具调用循环中，每次 API 调用都会报告 `cacheRead ≈ 当前上下文大小`。如果简单累加，N 次调用会得到 `N × 上下文大小`，这会夸大实际的上下文使用量。因此使用最后一次调用的值来准确反映当前上下文。

### 2. 会话写锁

```typescript
// attempt.ts:470-472
const sessionLock = await acquireSessionWriteLock({
  sessionFile: params.sessionFile,
});

try {
  // ... 执行逻辑
} finally {
  await sessionLock.release();
}
```

**作用**：
- 防止并发访问导致的会话文件损坏
- 确保同一时间只有一个操作修改会话
- 自动在执行完成后释放锁

### 3. 中止信号处理

```typescript
// attempt.ts:666-792
const runAbortController = new AbortController();

const abortRun = (isTimeout = false, reason?: unknown) => {
  aborted = true;
  if (isTimeout) {
    timedOut = true;
    runAbortController.abort(reason ?? makeTimeoutAbortReason());
  } else {
    runAbortController.abort(reason);
  }
  void activeSession.abort();
};

// 超时定时器
const abortTimer = setTimeout(
  () => {
    log.warn(`embedded run timeout: runId=${params.runId}`);
    abortRun(true);
  },
  Math.max(1, params.timeoutMs)
);

// 监听外部中止信号
if (params.abortSignal) {
  params.abortSignal.addEventListener("abort", onAbort, { once: true });
}
```

**功能**：
- 支持超时自动中止
- 支持外部中止信号（用户取消）
- 区分超时和用户中止

### 4. 图像检测与注入

```typescript
// attempt.ts:870-904
const imageResult = await detectAndLoadPromptImages({
  prompt: effectivePrompt,
  workspaceDir: effectiveWorkspace,
  model: params.model,
  existingImages: params.images,
  historyMessages: activeSession.messages,
  maxBytes: MAX_IMAGE_BYTES,
  sandbox: sandbox?.enabled ? { root: sandbox.workspaceDir } : undefined,
});

// 将历史图像注入到原始消息位置
const didMutate = injectHistoryImagesIntoMessages(
  activeSession.messages,
  imageResult.historyImagesByIndex
);

if (didMutate) {
  // 持久化消息变更，避免重复扫描/重新加载
  activeSession.agent.replaceMessages(activeSession.messages);
}

// 发送提示
if (imageResult.images.length > 0) {
  await activeSession.prompt(effectivePrompt, {
    images: imageResult.images
  });
} else {
  await activeSession.prompt(effectivePrompt);
}
```

**智能功能**：
- 自动检测提示中引用的图像文件路径
- 扫描对话历史中的图像引用
- 支持沙箱路径限制
- 仅在模型支持视觉时加载图像
- 避免重复加载

### 5. 插件钩子集成

```typescript
// attempt.ts:819-843
const hookRunner = getGlobalHookRunner();

// before_agent_start 钩子
if (hookRunner?.hasHooks("before_agent_start")) {
  const hookResult = await hookRunner.runBeforeAgentStart(
    {
      prompt: params.prompt,
      messages: activeSession.messages,
    },
    {
      agentId: hookAgentId,
      sessionKey: params.sessionKey,
      sessionId: params.sessionId,
      workspaceDir: params.workspaceDir,
    }
  );

  // 允许插件在提示前添加上下文
  if (hookResult?.prependContext) {
    effectivePrompt = `${hookResult.prependContext}\n\n${params.prompt}`;
  }
}

// agent_end 钩子（fire-and-forget）
if (hookRunner?.hasHooks("agent_end")) {
  hookRunner.runAgentEnd({
    messages: messagesSnapshot,
    success: !aborted && !promptError,
    error: promptError ? describeUnknownError(promptError) : undefined,
  }, { /* context */ })
  .catch((err) => log.warn(`agent_end hook failed: ${err}`));
}
```

**可用钩子**：
- `before_agent_start` - 预执行设置
- `before_tool_call` / `after_tool_call` - 工具拦截
- `before_compaction` / `after_compaction` - 压缩监控
- `session_start` / `session_end` - 会话生命周期
- `agent_end` - 执行完成后分析

## 性能优化

### 1. 会话管理器缓存

```typescript
// session-manager-cache.ts
await prewarmSessionFile(params.sessionFile);
sessionManager = guardSessionManager(
  SessionManager.open(params.sessionFile),
  { /* guards */ }
);
trackSessionManagerAccess(params.sessionFile);
```

**优化**：
- 预热会话文件以加快访问
- 跟踪访问模式
- 缓存会话管理器实例

### 2. 工具结果截断

```typescript
// run.ts:587-633
if (!toolResultTruncationAttempted) {
  const hasOversized = sessionLikelyHasOversizedToolResults({
    messages: attempt.messagesSnapshot,
    contextWindowTokens,
  });

  if (hasOversized) {
    toolResultTruncationAttempted = true;
    const truncResult = await truncateOversizedToolResultsInSession({
      sessionFile: params.sessionFile,
      contextWindowTokens,
    });

    if (truncResult.truncated) {
      // 会话现在更小了；允许再次尝试压缩
      overflowCompactionAttempts = 0;
      continue;
    }
  }
}
```

**场景**：
- 单个工具结果超过上下文窗口
- 压缩无法进一步缩减
- 自动截断并重试

### 3. 缓存 TTL 追踪

```typescript
// attempt.ts:957-966
const shouldTrackCacheTtl =
  params.config?.agents?.defaults?.contextPruning?.mode === "cache-ttl" &&
  isCacheTtlEligibleProvider(params.provider, params.modelId);

if (shouldTrackCacheTtl) {
  appendCacheTtlTimestamp(sessionManager, {
    timestamp: Date.now(),
    provider: params.provider,
    modelId: params.modelId,
  });
}
```

**作用**：
- 支持基于缓存 TTL 的上下文修剪
- 仅对支持的提供商启用
- 在提示+压缩重试完成后追加时间戳

## 总结

OpenClaw 的 Agent Loop 是一个精心设计的多层系统，具有：

### 优势

1. **强大的错误恢复**：自动处理上下文溢出、认证失败、速率限制
2. **智能重试机制**：多层嵌套循环，逐步降级
3. **流式处理**：实时工具调用和响应流
4. **可扩展性**：丰富的插件钩子系统
5. **并发安全**：会话写锁、中止信号、状态追踪

### 关键设计模式

1. **嵌套循环架构**：认证配置 → 主执行 → 工具调用
2. **状态机模式**：思考块状态、代码块状态、压缩状态
3. **事件驱动**：基于 EventEmitter 的流式处理
4. **资源管理**：锁、中止控制器、定时器清理
5. **降级策略**：思考级别、模型、认证配置

### 执行保证

- ✅ 自动恢复常见错误
- ✅ 优雅的降级和失败
- ✅ 详细的日志记录和诊断
- ✅ 并发安全的会话访问
- ✅ 资源清理（锁、定时器、监听器）

---

**文档版本**: 1.0
**最后更新**: 2026-02-14
**分析范围**: OpenClaw v0.x Agent Loop 核心实现
