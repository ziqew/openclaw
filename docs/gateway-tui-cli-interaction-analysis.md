# Gateway 与 TUI/CLI 客户端交互源码分析

本文档详细分析 OpenClaw 中 Gateway (网关) 与 TUI (终端用户界面) 和 CLI (命令行界面) 客户端之间的交互机制,包括连接协议、通信模式、认证授权、实时事件流等核心功能。

## 目录

1. [客户端类型与连接模式](#1-客户端类型与连接模式)
2. [连接协议 (WebSocket)](#2-连接协议-websocket)
3. [帧格式与消息结构](#3-帧格式与消息结构)
4. [认证与会话管理](#4-认证与会话管理)
5. [通信协议 - 核心 RPC 方法](#5-通信协议-核心-rpc-方法)
6. [实时事件流 (TUI)](#6-实时事件流-tui)
7. [CLI 命令执行流程](#7-cli-命令执行流程)
8. [请求-响应模式](#8-请求-响应模式)
9. [重连与错误处理](#9-重连与错误处理)
10. [核心数据结构与接口](#10-核心数据结构与接口)
11. [TUI vs CLI 差异对比](#11-tui-vs-cli-差异对比)
12. [错误处理与恢复机制](#12-错误处理与恢复机制)
13. [授权与作用域保护](#13-授权与作用域保护)

---

## 1. 客户端类型与连接模式

### 1.1 客户端模式

**文件位置:** `src/gateway/protocol/client-info.ts`

**客户端模式类型:**

```typescript
type ClientMode =
  | "cli"         // 命令行界面客户端
  | "ui"          // TUI (终端用户界面) 客户端
  | "webchat"     // Web 聊天界面
  | "backend"     // 后端系统组件
  | "node"        // 节点 (Agent 执行主机) 角色
  | "probe"       // 健康检查/探测客户端
  | "test";       // 测试客户端
```

**客户端名称标识:**

```typescript
export const GATEWAY_CLIENT_NAMES = {
  CLI: "cli",
  GATEWAY_CLIENT: "gateway-client",    // TUI 实际使用
  WEBCHAT_UI: "webchat-ui",
  CONTROL_UI: "openclaw-control-ui",
  NODE_HOST: "node-host",
} as const;

export const GATEWAY_CLIENT_MODES = {
  CLI: "cli",
  UI: "ui",
  WEBCHAT: "webchat",
  BACKEND: "backend",
  NODE: "node",
  PROBE: "probe",
  TEST: "test",
} as const;
```

### 1.2 客户端能力标识

```typescript
export const GATEWAY_CLIENT_CAPS = {
  TOOL_EVENTS: "tool_events",          // 支持工具执行事件
  REASONING_STREAM: "reasoning_stream", // 支持推理流
  BLOCK_STREAMING: "block_streaming",   // 支持分块流式传输
} as const;
```

---

## 2. 连接协议 (WebSocket)

### 2.1 协议详情

**传输层:**
- **协议:** WebSocket (`ws://` 或 `wss://`)
- **默认 URL:** `ws://127.0.0.1:18789`
- **协议版本:** 通过 `PROTOCOL_VERSION` 常量管理兼容性
- **最大负载:** 每连接 25 MB
- **序列化:** JSON

**文件位置:** `src/gateway/client.ts`, `src/gateway/server/ws-connection.ts`

### 2.2 连接流程

```
客户端                              Gateway
  |                                   |
  |-- WebSocket Open ----------------->|
  |                                   |
  |<-- connect.challenge event -------|  (可选 nonce)
  |                                   |
  |-- connect(ConnectParams) 请求 ---->|
  |                                   | 验证认证
  |                                   | 分配 connId
  |<-- hello-ok 响应 -----------------|
  |    (协议版本、功能列表、系统快照)     |
  |                                   |
  |-- 轮询/订阅事件 ------------------>|
  |                                   |
  |<-- tick 事件 (心跳) --------------|  (默认每 30 秒)
  |                                   |
```

**连接建立代码示例:**

```typescript
// src/gateway/client.ts
export class GatewayClient {
  start() {
    this.ws = new WebSocket(this.url, {
      maxPayload: 25 * 1024 * 1024   // 25 MB
    });

    this.ws.on("open", () => {
      this.sendConnect();
    });

    this.ws.on("message", (data) => {
      const frame = JSON.parse(data.toString());
      this.handleFrame(frame);
    });

    this.ws.on("close", (code, reason) => {
      this.reconnect();
    });
  }

  private sendConnect() {
    const connectParams: ConnectParams = {
      minProtocol: 1,
      maxProtocol: PROTOCOL_VERSION,
      client: {
        id: this.opts.clientName,
        displayName: this.opts.clientDisplayName,
        version: VERSION,
        platform: process.platform,
        mode: this.opts.mode,
        instanceId: this.opts.instanceId || randomUUID()
      },
      caps: this.opts.caps || [],
      commands: this.opts.commands || [],
      auth: {
        token: this.opts.token,
        password: this.opts.password
      },
      role: this.opts.role || "operator",
      scopes: this.opts.scopes || []
    };

    this.ws?.send(JSON.stringify({
      type: "req",
      id: randomUUID(),
      method: "connect",
      params: connectParams
    }));
  }
}
```

---

## 3. 帧格式与消息结构

### 3.1 请求帧 (客户端 → 服务器)

```typescript
{
  type: "req",
  id: string,              // UUID,用于关联请求和响应
  method: string,          // RPC 方法名,如 "chat.send", "sessions.list"
  params?: unknown         // 方法特定的参数
}
```

**示例:**

```typescript
{
  type: "req",
  id: "550e8400-e29b-41d4-a716-446655440000",
  method: "chat.send",
  params: {
    sessionKey: "main",
    message: "Hello, world!",
    idempotencyKey: "run-123"
  }
}
```

### 3.2 响应帧 (服务器 → 客户端)

```typescript
{
  type: "res",
  id: string,              // 匹配请求的 ID
  ok: boolean,             // 成功/失败指示器
  payload?: unknown,       // 响应数据 (成功时)
  error?: ErrorShape       // 错误详情 (失败时)
}
```

**ErrorShape 结构:**

```typescript
type ErrorShape = {
  code: string,            // 错误码,如 "INVALID_REQUEST"
  message: string,         // 人类可读的错误消息
  details?: unknown,       // 附加上下文
  retryable?: boolean,     // 客户端是否应重试
  retryAfterMs?: number    // 退避建议 (毫秒)
};
```

**示例:**

```typescript
// 成功响应
{
  type: "res",
  id: "550e8400-e29b-41d4-a716-446655440000",
  ok: true,
  payload: {
    runId: "run-123",
    status: "started"
  }
}

// 错误响应
{
  type: "res",
  id: "550e8400-e29b-41d4-a716-446655440000",
  ok: false,
  error: {
    code: "INVALID_REQUEST",
    message: "session not found",
    details: { sessionKey: "invalid-key" }
  }
}
```

### 3.3 事件帧 (服务器 → 客户端广播)

```typescript
{
  type: "event",
  event: string,           // 事件名称,如 "chat", "agent", "tick"
  payload?: unknown,       // 事件特定数据
  seq?: number,            // 序列号 (用于间隙检测)
  stateVersion?: {         // 状态版本跟踪
    presence?: number,
    health?: number
  }
}
```

**示例:**

```typescript
{
  type: "event",
  event: "chat",
  seq: 105,
  payload: {
    runId: "run-123",
    sessionKey: "main",
    seq: 0,
    state: "delta",
    message: {
      content: [
        { type: "text", text: "Hello" }
      ]
    }
  }
}
```

---

## 4. 认证与会话管理

### 4.1 认证方法

**文件位置:** `src/gateway/auth.ts`

**支持的认证方式:**

1. **基于令牌 (Token-based):** 静态认证令牌
2. **基于密码 (Password-based):** 共享密码认证
3. **设备令牌 (Device-token):** 设备特定的持久化令牌
4. **Tailscale:** 来自 Tailscale 网络的身份

**认证参数结构:**

```typescript
type ConnectAuthParams = {
  token?: string,          // 静态令牌
  password?: string,       // 共享密码
  device?: {
    id: string,            // 设备 ID
    publicKey: string,     // RSA 公钥 (PEM 格式)
    signature: string,     // 设备负载签名
    signedAt: number,      // 签名时间戳
    nonce?: string         // 可选 nonce (来自 challenge)
  }
};
```

### 4.2 设备认证流程

```
客户端                              Gateway
  |                                   |
  |-- 加载/创建设备 ID -------------    |  (持久化设备身份)
  |                                   |
  |-- 用私钥签名设备负载 ------------    |  (RSA 签名)
  |                                   |
  |-- 发送 connect() 包含:             |
  |    device.id                      |
  |    device.publicKey               |
  |    device.signature               |
  |    device.signedAt                |
  |                                   |
  |                                   | 验证签名
  |                                   | 生成设备令牌
  |<-- 存储设备令牌 -------------------|  (在 hello-ok 中返回)
  |<-- 作用域 (权限) ------------------|
  |                                   |
```

**设备身份持久化:**

**存储位置:** `~/.openclaw/device-identity.json`

```json
{
  "deviceId": "device-abc123",
  "privateKey": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----",
  "publicKey": "-----BEGIN PUBLIC KEY-----\n...\n-----END PUBLIC KEY-----",
  "createdAt": 1707926400000
}
```

**签名验证代码示例:**

```typescript
// src/gateway/auth.ts
function verifyDeviceSignature(params: {
  deviceId: string;
  publicKey: string;
  signature: string;
  signedAt: number;
  nonce?: string;
}): boolean {
  const payload = [
    params.deviceId,
    params.signedAt.toString(),
    params.nonce || ""
  ].join(":");

  const verifier = crypto.createVerify("RSA-SHA256");
  verifier.update(payload);
  verifier.end();

  return verifier.verify(
    params.publicKey,
    params.signature,
    "base64"
  );
}
```

### 4.3 授权作用域

**作用域列表:**

```typescript
const SCOPES = {
  ADMIN: "operator.admin",           // 完全管理访问权限
  READ: "operator.read",             // 只读操作 (查看会话、日志等)
  WRITE: "operator.write",           // 写操作 (发送聊天、修改设置)
  APPROVALS: "operator.approvals",   // 执行审批操作
  PAIRING: "operator.pairing",       // 设备/节点配对操作
} as const;
```

**作用域检查示例:**

```typescript
// src/gateway/server-methods.ts
function checkMethodScopes(method: string, scopes: string[]): ErrorShape | null {
  // 管理员拥有所有权限
  if (scopes.includes("operator.admin")) {
    return null;
  }

  // 检查写操作权限
  if (WRITE_METHODS.has(method) && !scopes.includes("operator.write")) {
    return errorShape(ErrorCodes.INVALID_REQUEST, "missing operator.write scope");
  }

  // 检查审批权限
  if (APPROVAL_METHODS.has(method) && !scopes.includes("operator.approvals")) {
    return errorShape(ErrorCodes.INVALID_REQUEST, "missing operator.approvals scope");
  }

  return null;
}
```

---

## 5. 通信协议 - 核心 RPC 方法

### 5.1 聊天操作

#### **chat.send**

**文件位置:** `src/gateway/server-methods/chat.ts`

**请求参数:**

```typescript
{
  sessionKey: string,           // 会话标识符
  message: string,              // 用户消息
  thinking?: string,            // 内部推理 (可选)
  deliver?: boolean,            // 是否发送到通道
  attachments?: Array<{         // 媒体附件
    type?: string,
    mimeType?: string,
    fileName?: string,
    content?: string           // base64 编码
  }>,
  timeoutMs?: number,           // 操作超时
  idempotencyKey: string        // 去重密钥 (runId)
}
```

**响应 (立即确认):**

```typescript
{
  runId: string,                // 聊天运行标识符
  status: "started" | "in_flight"
}
```

**流式事件:**

```typescript
{
  event: "chat",
  payload: {
    runId: string,
    sessionKey: string,
    seq: number,                // 运行内的序列号
    state: "delta" | "final" | "aborted" | "error",
    message?: {
      content: Array<{
        type: "text" | "tool_use" | "tool_result",
        text?: string,
        // ... 其他内容块字段
      }>
    },
    errorMessage?: string,      // 如果 state="error"
    usage?: {                   // Token 使用量
      input: number,
      output: number
    },
    stopReason?: string         // 运行结束原因
  }
}
```

**代码示例:**

```typescript
// src/gateway/server-methods/chat.ts
"chat.send": async ({ params, respond, context, client }) => {
  const p = params as ChatSendParams;

  // 验证参数
  if (!p.sessionKey || !p.message) {
    respond(false, undefined, errorShape(
      ErrorCodes.INVALID_REQUEST,
      "missing required params"
    ));
    return;
  }

  // 生成或使用提供的 runId
  const runId = p.idempotencyKey || randomUUID();

  // 检查去重
  const cached = context.dedupe.get(`chat:${runId}`);
  if (cached) {
    respond(cached.ok, cached.payload, cached.error, { cached: true });
    return;
  }

  // 检查是否已在运行中
  const activeExisting = context.chatAbortControllers.get(runId);
  if (activeExisting) {
    respond(true, { runId, status: "in_flight" }, undefined, { cached: true });
    return;
  }

  // 立即响应
  respond(true, { runId, status: "started" }, undefined);

  // 异步执行聊天
  void executeChatRun({
    runId,
    sessionKey: p.sessionKey,
    message: p.message,
    context,
    onEvent: (evt) => {
      // 广播事件到所有订阅的客户端
      context.broadcast("chat", evt);
    }
  });
}
```

#### **chat.history**

**请求参数:**

```typescript
{
  sessionKey: string,
  limit?: number              // 最大检索消息数
}
```

**响应:**

```typescript
[
  {
    role: "user" | "assistant",
    content: unknown,           // 消息内容块
    timestamp: number
  }
]
```

#### **chat.abort**

**请求参数:**

```typescript
{
  sessionKey: string,
  runId?: string              // 特定运行或全部
}
```

**响应:**

```typescript
{
  ok: boolean,
  aborted: boolean,
  runIds: string[]
}
```

### 5.2 会话管理

#### **sessions.list**

**响应包含所有会话及元数据列表:**

```typescript
{
  ts: number,
  path: string,
  count: number,
  sessions: Array<{
    key: string,
    sessionId?: string,
    updatedAt?: number,
    model?: string,
    contextTokens?: number,
    inputTokens?: number,
    outputTokens?: number,
    totalTokens?: number,
    label?: string
  }>
}
```

#### **sessions.patch**

**请求 - 部分更新:**

```typescript
{
  key: string,
  model?: string,
  contextTokens?: number,
  thinkingLevel?: string,
  verboseLevel?: string,
  reasoningLevel?: string,
  sendPolicy?: string
}
```

**响应:**

```typescript
{
  key: string,
  updated: boolean,
  [fieldName]: newValue
}
```

### 5.3 Agent 操作

#### **agents.list**

**响应:**

```typescript
{
  defaultId: string,
  mainKey: string,
  scope: "per-sender" | "global",
  agents: Array<{
    id: string,
    name?: string
  }>
}
```

---

## 6. 实时事件流 (TUI)

### 6.1 事件类型

**文件位置:** `src/gateway/protocol/schema/events.ts`

**流式传输到客户端的事件类型:**

1. **chat** - 聊天完成事件 (流式传输)
2. **agent** - Agent 执行事件
3. **tick** - 心跳 (默认每 30 秒)
4. **shutdown** - Gateway 关闭通知
5. **system-presence** - 系统状态变更
6. **exec.approval.requested** - 审批工作流事件
7. **exec.approval.resolved** - 审批解决事件
8. **device.pair.requested** - 设备配对事件
9. **node.pair.requested** - 节点配对事件

### 6.2 流式组装器

**文件位置:** `src/tui/tui-stream-assembler.ts`

**功能:**
- 累积 delta 事件到完整消息
- 分别跟踪 thinking 与 content
- 处理文本块和工具事件
- 检测边界文本块子集
- 当 "final" 事件到达时组装最终状态

**代码示例:**

```typescript
export class TuiStreamAssembler {
  private thinking: Array<ContentBlock> = [];
  private content: Array<ContentBlock> = [];
  private state: "delta" | "final" | "aborted" | "error" = "delta";

  update(evt: ChatEvent) {
    if (evt.state === "delta") {
      // 累积增量
      if (evt.message?.thinking) {
        this.thinking = this.mergeContentBlocks(
          this.thinking,
          evt.message.thinking
        );
      }
      if (evt.message?.content) {
        this.content = this.mergeContentBlocks(
          this.content,
          evt.message.content
        );
      }
    } else {
      // 最终状态
      this.state = evt.state;
      if (evt.message?.thinking) {
        this.thinking = evt.message.thinking;
      }
      if (evt.message?.content) {
        this.content = evt.message.content;
      }
    }
  }

  getAssembled(): AssembledMessage {
    return {
      thinking: this.thinking,
      content: this.content,
      state: this.state
    };
  }

  private mergeContentBlocks(
    existing: ContentBlock[],
    incoming: ContentBlock[]
  ): ContentBlock[] {
    // 合并逻辑:处理文本增量和工具块
    // ...
  }
}
```

### 6.3 TUI 事件处理循环

**文件位置:** `src/tui/tui-event-handlers.ts`

```typescript
export function handleChatEvent(
  payload: ChatEvent,
  state: TuiState
) {
  // 同步到当前会话
  syncSessionKey();

  // 如果是不同会话,忽略
  if (payload.sessionKey !== state.currentSessionKey) {
    return;
  }

  // 跟踪已完成的运行 (避免重复处理)
  if (state.finalizedRuns.has(payload.runId)) {
    if (payload.state !== "delta") return;
  }

  // 添加到活动运行
  noteSessionRun(payload.runId);

  // 根据状态更新 TUI 显示
  switch (payload.state) {
    case "delta":
      // 追加到聊天日志
      appendToChat(payload);
      break;

    case "final":
      // 标记为完成,显示使用量
      markAsComplete(payload);
      showUsage(payload.usage);
      state.finalizedRuns.add(payload.runId);
      break;

    case "aborted":
      // 标记为取消
      markAsCancelled(payload);
      break;

    case "error":
      // 显示错误消息
      showError(payload.errorMessage);
      break;
  }
}
```

---

## 7. CLI 命令执行流程

### 7.1 基本 CLI RPC 流程

```
CLI 输入                       Gateway Client         Gateway Server
  |                                 |                       |
  |-- 解析命令               ---    |                       |
  |    (如 send message)            |                       |
  |                                 |                       |
  |-- 解析 Gateway URL/Auth ---     |                       |
  |    (从配置或环境变量)             |                       |
  |                                 |                       |
  |-- 创建 GatewayClient -----      |                       |
  |                                 |                       |
  |-- Start() ----------------->  WebSocket Connected     |
  |                                 |                       |
  |                                 |-- connect() -------->|
  |                                 |<-- hello-ok ---------|
  |                                 |                       |
  |-- request(method, params) --->  |-- req frame ------->|
  |    (如 chat.send)               |                      |
  |                                 |<-- res frame --------|
  |                                 |                      |
  |-- formatOutput(result)    ---   |                       |
  |-- 显示 JSON 或格式化结果          |                       |
  |    到 stdout                    |                       |
  |                                 |                       |
  |-- Stop() ----------------->    WebSocket Closed      |
```

### 7.2 CLI Gateway 集成

**文件位置:** `src/cli/gateway-rpc.ts`

**调用包装器:**

```typescript
// CLI 调用: callGatewayFromCli(method, opts, params)
async function callGatewayFromCli(
  method: string,
  opts: GatewayRpcOpts,
  params?: unknown
) {
  // 使用 callGateway(),它会:
  // 1. 创建一次性 GatewayClient
  // 2. 等待 hello-ok
  // 3. 发送单个请求
  // 4. 返回响应
  // 5. 关闭连接

  return withProgress({
    label: `Gateway ${method}`,
    indeterminate: true,
    enabled: !opts.json
  }, async () =>
    await callGateway({
      url: opts.url,
      token: opts.token,
      password: opts.password,
      method,
      params,
      expectFinal: opts.expectFinal,
      timeoutMs: Number(opts.timeout ?? 10_000),
      clientName: GATEWAY_CLIENT_NAMES.CLI,
      mode: GATEWAY_CLIENT_MODES.CLI
    })
  )
}
```

**文件位置:** `src/gateway/call.ts`

**callGateway 实现:**

```typescript
export async function callGateway(opts: CallGatewayOpts): Promise<unknown> {
  return new Promise((resolve, reject) => {
    const client = new GatewayClient({
      url: opts.url,
      token: opts.token,
      password: opts.password,
      clientName: opts.clientName || GATEWAY_CLIENT_NAMES.CLI,
      mode: opts.mode || GATEWAY_CLIENT_MODES.CLI,
      role: "operator",
      scopes: ["operator.admin", "operator.approvals", "operator.pairing"],
      onHelloOk: async () => {
        try {
          // 发送单个请求
          const result = await client.request(opts.method, opts.params, {
            expectFinal: opts.expectFinal
          });
          resolve(result);
        } catch (err) {
          reject(err);
        } finally {
          client.stop();
        }
      },
      onClose: (code, reason) => {
        if (!settled) {
          reject(new Error(formatCloseError(code, reason)));
        }
      },
      onConnectError: (err) => {
        reject(err);
      }
    });

    client.start();
  });
}
```

### 7.3 CLI 命令示例

**发送聊天消息:**

```bash
openclaw send --session main "Hello, world!"
```

**内部执行:**

```typescript
// src/cli/commands/send.ts
async function sendCommand(opts: SendOpts) {
  const result = await callGatewayFromCli("chat.send", opts, {
    sessionKey: opts.session || "main",
    message: opts.message,
    idempotencyKey: randomUUID()
  });

  if (opts.json) {
    console.log(JSON.stringify(result, null, 2));
  } else {
    console.log(`✓ Sent to session: ${opts.session}`);
    console.log(`  Run ID: ${result.runId}`);
  }
}
```

---

## 8. 请求-响应模式

### 8.1 即时响应

**用于:**
- 配置获取/设置
- sessions.list
- 大多数查询操作

**流程:**

```
Client                          Gateway
  |                                |
  |-- req (id: "123") ----------->|
  |                                | 处理请求
  |<-- res (id: "123") -----------|
  |    { ok: true, payload: ... } |
  |                                |
```

### 8.2 流式响应 (expectFinal=true)

**用于:**
- Agent 运行
- 长时间运行的操作

**流程:**

```
Client                          Gateway
  |                                |
  |-- req (id: "123") ----------->|
  |    { expectFinal: true }      |
  |                                |
  |<-- res (id: "123") -----------|
  |    { ok: true,                |
  |      payload: {               |
  |        status: "accepted"     |
  |      }                        |
  |    }                          |
  |                                |
  | [客户端继续等待]                 |
  |                                |
  |<-- res (id: "123") -----------|  (最终响应)
  |    { ok: true,                |
  |      payload: {               |
  |        result: ...            |
  |      },                       |
  |      final: true              |
  |    }                          |
  |                                |
```

**代码示例:**

```typescript
// src/gateway/client.ts
async request<T>(
  method: string,
  params?: unknown,
  opts?: { expectFinal?: boolean }
): Promise<T> {
  const requestId = randomUUID();

  return new Promise((resolve, reject) => {
    this.pending.set(requestId, {
      resolve: (value) => resolve(value as T),
      reject,
      expectFinal: opts?.expectFinal || false
    });

    this.ws?.send(JSON.stringify({
      type: "req",
      id: requestId,
      method,
      params
    }));
  });
}

private handleResponse(frame: ResponseFrame) {
  const pending = this.pending.get(frame.id);
  if (!pending) return;

  if (frame.ok) {
    if (pending.expectFinal && !frame.final) {
      // 继续等待最终响应
      return;
    }
    pending.resolve(frame.payload);
  } else {
    pending.reject(new Error(frame.error?.message || "unknown error"));
  }

  this.pending.delete(frame.id);
}
```

### 8.3 即发即弃事件

**用于:**
- 服务器推送通知
- 心跳 (tick)
- 聊天事件流

**流程:**

```
Client                          Gateway
  |                                |
  | [客户端订阅事件]                  |
  |                                |
  |<-- event (seq: 100) -----------|
  |    { event: "chat", ... }     |
  |                                |
  |<-- event (seq: 101) -----------|
  |<-- event (seq: 102) -----------|
  |                                |
```

---

## 9. 重连与错误处理

### 9.1 退避重试策略

**文件位置:** `src/gateway/client.ts`

**指数退避算法:**

```
连接尝试:
  第 1 次失败: 等待 1 秒,重试
  第 2 次失败: 等待 2 秒,重试
  第 3 次失败: 等待 4 秒,重试
  第 4 次失败: 等待 8 秒,重试
  ...
  最大退避: 30 秒
```

**代码实现:**

```typescript
export class GatewayClient {
  private backoffMs = 1000;
  private maxBackoffMs = 30_000;

  private reconnect() {
    if (this.closed) return;

    setTimeout(() => {
      this.backoffMs = Math.min(this.backoffMs * 1.5, this.maxBackoffMs);
      this.start();
    }, this.backoffMs);
  }

  private onConnected() {
    // 重置退避
    this.backoffMs = 1000;
  }
}
```

### 9.2 心跳监控 (Tick Watch)

**功能:**
- Gateway 每 `tickIntervalMs` (默认 30 秒) 发送 "tick" 事件
- 客户端跟踪 `lastTick` 时间戳
- 如果间隙 > `tickIntervalMs * 2`,关闭连接 (tick 超时)
- 触发立即重连尝试

**代码示例:**

```typescript
// src/gateway/client.ts
private tickWatchInterval?: NodeJS.Timeout;

private startTickWatch() {
  const tickIntervalMs = this.hello?.policy?.tickIntervalMs || 30_000;
  const tickTimeoutMs = tickIntervalMs * 2;

  this.tickWatchInterval = setInterval(() => {
    const now = Date.now();
    if (now - this.lastTickMs > tickTimeoutMs) {
      console.warn("Tick timeout - closing connection");
      this.ws?.close(1000, "tick timeout");
    }
  }, tickIntervalMs);
}

private handleTickEvent() {
  this.lastTickMs = Date.now();
}
```

### 9.3 间隙检测

**序列号跟踪:**

```typescript
private lastSeq: number | null = null;

private handleEvent(frame: EventFrame) {
  const seq = frame.seq;

  if (seq !== null && this.lastSeq !== null && seq > this.lastSeq + 1) {
    // 检测到间隙
    this.opts.onGap?.({
      expected: this.lastSeq + 1,
      received: seq
    });
  }

  this.lastSeq = seq;
}
```

### 9.4 关闭代码

**WebSocket 关闭代码:**
- `1000`: 正常关闭
- `1006`: 异常关闭 (无关闭帧)
- `1008`: 策略违规 (认证失败、格式错误)
- `1012`: 服务重启

**错误格式化:**

```typescript
function formatCloseError(code: number, reason: string): string {
  switch (code) {
    case 1000:
      return "Connection closed normally";
    case 1006:
      return "Connection lost (abnormal closure)";
    case 1008:
      return `Policy violation: ${reason}`;
    case 1012:
      return "Service restarting";
    default:
      return `Connection closed (${code}): ${reason}`;
  }
}
```

---

## 10. 核心数据结构与接口

### 10.1 ConnectParams (客户端 hello)

```typescript
type ConnectParams = {
  minProtocol: number,
  maxProtocol: number,
  client: {
    id: string,                // 客户端名称/ID
    displayName?: string,      // 显示名称
    version: string,           // 客户端版本
    platform: string,          // 平台 (linux, darwin, win32)
    mode: string,              // "ui", "cli", "backend"
    instanceId?: string        // 此客户端的会话 UUID
  },
  caps: string[],              // 客户端能力
  commands?: string[],         // 支持的命令
  permissions?: Record<string, boolean>,
  auth: {
    token?: string,            // 静态令牌
    password?: string,         // 共享密码
    device?: {                 // 设备身份
      id: string,
      publicKey: string,
      signature: string,
      signedAt: number,
      nonce?: string
    }
  },
  role: string,                // "operator", "node"
  scopes: string[]             // 权限作用域
};
```

### 10.2 HelloOk (服务器欢迎)

```typescript
type HelloOk = {
  type: "hello-ok",
  protocol: number,            // 协商的协议版本
  server: {
    version: string,           // 服务器版本
    commit?: string,           // Git commit
    host?: string,             // 主机名
    connId: string             // 连接 ID (用于跟踪)
  },
  features: {
    methods: string[],         // 可用的 RPC 方法
    events: string[]           // 可用的事件类型
  },
  snapshot: Snapshot,          // 初始系统状态
  auth?: {
    deviceToken: string,       // 要存储的持久化令牌
    role: string,              // 分配的角色
    scopes: string[],          // 授予的作用域
    issuedAtMs?: number        // 发行时间
  },
  policy: {
    maxPayload: number,        // 每消息最大字节数
    maxBufferedBytes: number,  // 断开前的缓冲限制
    tickIntervalMs: number     // 心跳间隔
  }
};
```

### 10.3 ChatEvent (聊天事件负载)

```typescript
type ChatEvent = {
  runId: string,               // 聊天运行标识符
  sessionKey: string,          // 会话密钥
  seq: number,                 // 运行内的序列号
  state: "delta" | "final" | "aborted" | "error",
  message?: {
    thinking?: Array<ContentBlock>,  // 思考内容块
    content?: Array<ContentBlock>    // 输出内容块
  },
  errorMessage?: string,       // 如果 state="error"
  usage?: {
    input: number,             // 输入 token
    output: number             // 输出 token
  },
  stopReason?: string          // 运行结束原因
};

type ContentBlock = {
  type: "text" | "tool_use" | "tool_result" | "image",
  text?: string,
  id?: string,
  name?: string,
  input?: unknown,
  content?: unknown,
  is_error?: boolean
};
```

---

## 11. TUI vs CLI 差异对比

| 方面 | TUI | CLI |
|--------|-----|-----|
| **连接** | 长连接 WebSocket | 单请求连接 |
| **客户端模式** | "ui" | "cli" |
| **事件流** | 订阅事件 | 无订阅 |
| **交互** | 交互式实时 UI | 每次调用一个命令 |
| **超时** | 动态 (用户交互) | 固定 (默认 30 秒) |
| **状态跟踪** | 维护会话状态 | 每次调用无状态 |
| **显示** | 终端渲染 (TUI 库) | JSON/文本输出 |
| **轮询** | 事件驱动 | 仅请求-响应 |
| **去重** | 聊天运行跟踪 | 参数中的幂等性密钥 |
| **实例 ID** | 持久化 (每会话) | 临时 (每请求) |
| **能力** | tool_events, reasoning_stream | 无特殊能力 |

### 11.1 TUI 客户端创建

**文件位置:** `src/tui/gateway-chat.ts`

```typescript
this.client = new GatewayClient({
  url,
  token,
  password,
  clientName: GATEWAY_CLIENT_NAMES.GATEWAY_CLIENT,
  clientDisplayName: "openclaw-tui",
  mode: GATEWAY_CLIENT_MODES.UI,
  caps: [
    GATEWAY_CLIENT_CAPS.TOOL_EVENTS,
    GATEWAY_CLIENT_CAPS.REASONING_STREAM
  ],
  role: "operator",
  scopes: ["operator.admin"],
  instanceId: randomUUID(),                    // 每会话实例

  onHelloOk: (hello) => {
    this.hello = hello;
    this.resolveReady?.();
    this.onConnected?.();
  },

  onEvent: (evt) => {
    // 处理所有事件类型
    this.onEvent?.({
      event: evt.event,
      payload: evt.payload,
      seq: evt.seq
    });
  },

  onClose: (code, reason) => {
    this.onDisconnected?.(reason);
  },

  onGap: (info) => {
    console.warn(`Event gap detected: expected ${info.expected}, received ${info.received}`);
    this.onGap?.(info);
  }
});

this.client.start();
```

### 11.2 CLI 客户端创建

**文件位置:** `src/gateway/call.ts`

```typescript
const client = new GatewayClient({
  url,
  token,
  password,
  clientName: GATEWAY_CLIENT_NAMES.CLI,
  mode: GATEWAY_CLIENT_MODES.CLI,
  role: "operator",
  scopes: ["operator.admin", "operator.approvals", "operator.pairing"],

  onHelloOk: async () => {
    try {
      // 发送单个请求然后关闭
      const result = await client.request(method, params, {
        expectFinal: opts.expectFinal
      });
      resolve(result);
      client.stop();
    } catch (err) {
      reject(err);
      client.stop();
    }
  },

  onClose: (code, reason) => {
    // 如果在 settle 之前关闭,则报错
    if (!settled) {
      reject(new Error(formatCloseError(code, reason)));
    }
  },

  onConnectError: (err) => {
    reject(err);
  }
});

client.start();
```

---

## 12. 错误处理与恢复机制

### 12.1 服务器端验证

**文件位置:** `src/gateway/server-methods/chat.ts`

```typescript
"chat.send": async ({ params, respond, context }) => {
  // 验证参数
  const validated = validateChatSendParams(params);
  if (!validated.ok) {
    respond(false, undefined,
      errorShape(ErrorCodes.INVALID_REQUEST,
        `invalid params: ${formatValidationErrors(validated.errors)}`
      )
    );
    return;
  }

  // ... 处理请求
}
```

### 12.2 幂等性

**去重缓存:**

```typescript
// src/gateway/server-context.ts
export class GatewayContext {
  dedupe = new Map<string, {
    ok: boolean;
    payload?: unknown;
    error?: ErrorShape;
  }>();

  // 在处理前检查
  const cached = context.dedupe.get(`chat:${clientRunId}`);
  if (cached) {
    respond(cached.ok, cached.payload, cached.error, {
      cached: true
    });
    return;
  }

  // 缓存结果 (TTL: 5 分钟)
  context.dedupe.set(`chat:${clientRunId}`, {
    ok: true,
    payload: { runId: clientRunId, status: "started" }
  });

  setTimeout(() => {
    context.dedupe.delete(`chat:${clientRunId}`);
  }, 5 * 60 * 1000);
}
```

**运行中检查:**

```typescript
// 检查是否已在运行中
const activeExisting = context.chatAbortControllers.get(clientRunId);
if (activeExisting) {
  respond(true, {
    runId: clientRunId,
    status: "in_flight"
  }, undefined, {
    cached: true,
    runId: clientRunId
  });
  return;
}
```

### 12.3 客户端错误处理

**连接错误:**

```typescript
// src/gateway/client.ts
if (!this.ws || this.ws.readyState !== WebSocket.OPEN) {
  throw new Error("gateway not connected");
}

// 超时保护
const timeoutMs = opts?.timeoutMs || 30_000;
const timer = setTimeout(() => {
  const pending = this.pending.get(requestId);
  if (pending) {
    pending.reject(new Error("request timeout"));
    this.pending.delete(requestId);
  }
}, timeoutMs);
```

**响应验证:**

```typescript
private handleResponse(frame: ResponseFrame) {
  const pending = this.pending.get(frame.id);
  if (!pending) return;

  clearTimeout(pending.timer);

  if (frame.ok) {
    pending.resolve(frame.payload);
  } else {
    const error = new Error(frame.error?.message ?? "unknown error");
    (error as any).code = frame.error?.code;
    (error as any).details = frame.error?.details;
    pending.reject(error);
  }

  this.pending.delete(frame.id);
}
```

---

## 13. 授权与作用域保护

### 13.1 方法授权

**文件位置:** `src/gateway/server-methods.ts`

**方法分类:**

```typescript
const READ_METHODS = new Set([
  "health", "logs.tail", "channels.status", "status",
  "models.list", "agents.list", "sessions.list", "node.list"
]);

const WRITE_METHODS = new Set([
  "send", "agent", "agent.wait", "wake",
  "chat.send", "chat.abort", "node.invoke",
  "sessions.patch", "config.set"
]);

const APPROVAL_METHODS = new Set([
  "exec.approval.request", "exec.approval.resolve"
]);

const PAIRING_METHODS = new Set([
  "device.pair.request", "device.pair.approve",
  "node.pair.request", "node.pair.approve"
]);
```

**授权检查:**

```typescript
function checkMethodScopes(
  method: string,
  scopes: string[]
): ErrorShape | null {
  // 管理员拥有所有权限
  if (scopes.includes("operator.admin")) {
    return null;
  }

  // 检查审批权限
  if (APPROVAL_METHODS.has(method) && !scopes.includes("operator.approvals")) {
    return errorShape(
      ErrorCodes.INVALID_REQUEST,
      "missing operator.approvals scope"
    );
  }

  // 检查写操作权限
  if (WRITE_METHODS.has(method) && !scopes.includes("operator.write")) {
    return errorShape(
      ErrorCodes.INVALID_REQUEST,
      "missing operator.write scope"
    );
  }

  // 检查配对权限
  if (PAIRING_METHODS.has(method) && !scopes.includes("operator.pairing")) {
    return errorShape(
      ErrorCodes.INVALID_REQUEST,
      "missing operator.pairing scope"
    );
  }

  return null;
}
```

### 13.2 事件作用域保护

**文件位置:** `src/gateway/server-broadcast.ts`

**事件作用域映射:**

```typescript
const EVENT_SCOPE_GUARDS: Record<string, string[]> = {
  "exec.approval.requested": ["operator.approvals"],
  "exec.approval.resolved": ["operator.approvals"],
  "device.pair.requested": ["operator.pairing"],
  "node.pair.requested": ["operator.pairing"]
};
```

**广播过滤:**

```typescript
export function broadcast(
  event: string,
  payload: unknown,
  context: GatewayContext
) {
  const requiredScopes = EVENT_SCOPE_GUARDS[event];

  for (const client of context.connectedClients.values()) {
    // 只发送给拥有所需作用域的客户端
    if (requiredScopes && !hasEventScope(client, requiredScopes)) {
      continue;
    }

    client.socket.send(JSON.stringify({
      type: "event",
      event,
      payload,
      seq: context.nextSeq++
    }));
  }
}

function hasEventScope(client: GatewayWsClient, requiredScopes: string[]): boolean {
  const clientScopes = client.connect?.scopes || [];

  // 管理员拥有所有作用域
  if (clientScopes.includes("operator.admin")) {
    return true;
  }

  // 检查是否拥有所有所需作用域
  return requiredScopes.every(scope => clientScopes.includes(scope));
}
```

---

## 总结

### Gateway-TUI/CLI 架构的核心特点

1. **统一的 WebSocket 协议**
   - TUI 和 CLI 都使用相同的 WebSocket 协议
   - JSON-RPC 风格的消息格式
   - 支持请求-响应和事件流

2. **客户端模式差异**
   - **CLI:** 短连接,单请求-响应,无状态
   - **TUI:** 长连接,事件订阅,有状态

3. **认证机制**
   - 支持令牌、密码、设备身份多种认证方式
   - 基于作用域的细粒度授权
   - 持久化设备令牌用于免密重连

4. **实时事件流 (TUI)**
   - 服务器推送聊天事件、系统事件、心跳
   - 序列号跟踪和间隙检测
   - 流式组装器累积增量事件

5. **健壮性保证**
   - 指数退避重连 (最大 30 秒)
   - 心跳监控 (默认 30 秒间隔)
   - 幂等性保证 (通过 idempotencyKey)
   - 去重缓存 (5 分钟 TTL)

6. **错误处理**
   - 结构化错误响应 (code, message, details)
   - 重试建议 (retryable, retryAfterMs)
   - 优雅的关闭代码处理

7. **授权控制**
   - 方法级作用域检查
   - 事件订阅作用域过滤
   - 管理员拥有所有权限

8. **性能优化**
   - 最大负载 25 MB
   - 缓冲限制 10 MB
   - 请求超时保护

这个架构实现了灵活、安全、高性能的客户端-服务器通信,同时支持短命令行操作和长交互式会话。

---

**文档版本:** 1.0
**最后更新:** 2026-02-19
**相关文档:**
- [Agent Loop 分析](./agent-loop-analysis.md)
- [WhatsApp 集成分析](./whatsapp-integration-analysis.md)
- [消息平台对比](./messaging-platforms-comparison.md)
- [Gateway 和 Chat Channels 分析](./gateway-and-channels-analysis.md)
- [Gateway 与 Agent 交互分析](./gateway-agent-interaction-analysis.md)
- [Gateway 与 Node 交互分析](./gateway-node-interaction-analysis.md)
