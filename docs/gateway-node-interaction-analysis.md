# Gateway 与 Node 交互源码分析

本文档详细分析 OpenClaw 中 Gateway (网关) 与 Node (执行节点) 之间的交互机制,包括节点注册、通信协议、命令调用、认证授权等核心功能。

## 目录

1. [Node 概念说明](#1-node-概念说明)
2. [节点注册与连接](#2-节点注册与连接)
3. [通信协议](#3-通信协议)
4. [消息类型与数据结构](#4-消息类型与数据结构)
5. [认证与授权](#5-认证与授权)
6. [生命周期管理](#6-生命周期管理)
7. [命令调用:核心交互流程](#7-命令调用核心交互流程)
8. [节点事件处理](#8-节点事件处理)
9. [负载均衡与节点选择](#9-负载均衡与节点选择)
10. [错误处理与降级](#10-错误处理与降级)
11. [完整数据流图](#11-完整数据流图)
12. [关键代码示例](#12-关键代码示例)
13. [配置与常量](#13-配置与常量)

---

## 1. Node 概念说明

在 OpenClaw 的上下文中,**"Node" (节点) 指的是执行主机/工作节点**,通常是:

- **iOS 应用** (iPhone/iPad)
- **Android 应用**
- **macOS 伴侣应用**
- **无头节点主机** (运行 `openclaw node-host` 的服务器)

这些节点通过 WebSocket 连接到 Gateway,提供执行能力(如系统命令、浏览器代理等)。

**重要区分:**
- Node **不是** Node.js 运行时
- Node **不是** 网络对等节点
- Node **是** 提供特定能力的执行主机

---

## 2. 节点注册与连接

### 2.1 WebSocket 连接入口

**文件位置:** `src/gateway/server/ws-connection/message-handler.ts`

节点通过 WebSocket 连接到网关,默认地址为 `ws://gateway-host:18789`。

**连接握手帧结构:**

```typescript
{
  type: "req",
  method: "connect",
  id: "uuid",
  params: {
    minProtocol: number,              // 最小协议版本
    maxProtocol: number,              // 最大协议版本
    role: "node" | "operator",       // 角色标识
    client: {
      id: string,                    // 客户端 ID
      displayName: string,           // 显示名称
      mode: "node",                  // 模式
      version: string,               // 版本号
      instanceId?: string,           // 实例 ID
      platform?: string              // 平台 (iOS/Android/macOS)
    },
    device?: {
      id: string,                    // 设备 ID
      publicKey?: string             // 公钥(用于加密)
    },
    caps: string[],                  // 能力列表,如 ["system", "browser"]
    commands: string[],              // 支持的命令,如 ["system.run", "system.which"]
    auth?: {
      token?: string,                // 认证令牌
      password?: string,             // 密码
      deviceSignature?: string       // 设备签名
    }
  }
}
```

### 2.2 注册流程

**文件位置:** `src/gateway/node-registry.ts` (行 43-79)

**NodeSession 数据结构:**

```typescript
export type NodeSession = {
  nodeId: string;                    // 节点 ID
  connId: string;                    // 连接 ID
  client: GatewayWsClient;           // WebSocket 客户端对象
  displayName?: string;              // 显示名称
  platform?: string;                 // 平台
  version?: string;                  // 版本
  coreVersion?: string;              // 核心版本
  uiVersion?: string;                // UI 版本
  deviceFamily?: string;             // 设备系列
  modelIdentifier?: string;          // 型号标识
  remoteIp?: string;                 // 远程 IP
  caps: string[];                    // 能力列表
  commands: string[];                // 命令列表
  permissions?: Record<string, boolean>; // 权限映射
  pathEnv?: string;                  // PATH 环境变量
  connectedAtMs: number;             // 连接时间戳
};
```

**NodeRegistry 类:**

```typescript
export class NodeRegistry {
  private nodesById = new Map<string, NodeSession>();
  private nodesByConn = new Map<string, string>();

  register(client: GatewayWsClient, opts: { remoteIp?: string }): NodeSession {
    const connect = client.connect!;
    const nodeId = connect.device?.id ?? connect.client.id;

    const session: NodeSession = {
      nodeId,
      connId: client.connId,
      client,
      displayName: connect.client.displayName,
      platform: connect.client.platform,
      caps: Array.isArray(connect.caps) ? connect.caps : [],
      commands: Array.isArray(connect.commands) ? connect.commands : [],
      connectedAtMs: Date.now(),
      // ... 更多字段
    };

    this.nodesById.set(nodeId, session);
    this.nodesByConn.set(client.connId, nodeId);

    return session;
  }
}
```

**消息处理器中的注册:**

```typescript
// src/gateway/server/ws-connection/message-handler.ts (约第 800 行)
if (roleRaw === "node") {
  const nodeSession = context.nodeRegistry.register(nextClient, {
    remoteIp: reportedClientIp,
  });

  // 在持久化存储中记录最后连接时间
  void updatePairedNodeMetadata(nodeId, {
    lastConnectedAtMs: nodeSession.connectedAtMs,
  });
}
```

---

## 3. 通信协议

### 3.1 协议细节

**文件位置:** `src/gateway/protocol/schema/nodes.ts`

**传输层:**
- **协议:** WebSocket (`ws://` 或 `wss://`)
- **数据格式:** JSON (每条 WebSocket 消息一个 JSON 对象)
- **默认端口:** 18789
- **最大负载:** 25 MB (针对节点连接)
- **消息格式:** `{ type: "req"|"res"|"event", ... }`

### 3.2 消息帧类型

**请求帧 (客户端发送):**

```typescript
{
  type: "req",
  id: "uuid",                        // 请求 ID
  method: "node.invoke",             // 方法名
  params: {
    nodeId: string,
    command: string,
    params?: unknown,
    timeoutMs?: number,
    idempotencyKey: string
  }
}
```

**响应帧 (服务器返回):**

```typescript
{
  type: "res",
  id: "uuid",                        // 对应请求 ID
  ok: boolean,                       // 是否成功
  payload?: unknown,                 // 成功时的数据
  error?: {
    code: string,                    // 错误码
    message: string                  // 错误消息
  }
}
```

**事件广播帧 (服务器推送):**

```typescript
{
  type: "event",
  event: string,                     // 事件名称
  payload: unknown                   // 事件数据
}
```

---

## 4. 消息类型与数据结构

### 4.1 节点配对消息

**文件位置:** `src/gateway/protocol/schema/nodes.ts` (行 4-102)

**配对请求 (Node → Gateway):**

```typescript
export const NodePairRequestParamsSchema = Type.Object({
  nodeId: NonEmptyString,
  displayName?: NonEmptyString,
  platform?: NonEmptyString,         // "iOS", "Android", "macOS" 等
  version?: NonEmptyString,
  coreVersion?: NonEmptyString,
  uiVersion?: NonEmptyString,
  deviceFamily?: NonEmptyString,     // 例: "iPhone", "iPad"
  modelIdentifier?: NonEmptyString,  // 例: "iPhone14,2"
  caps?: Type.Array(NonEmptyString), // ["system", "browser", ...]
  commands?: Type.Array(NonEmptyString), // ["system.run", "system.which", ...]
  remoteIp?: NonEmptyString,
  silent?: Type.Boolean(),           // 是否静默配对(不通知审批)
});
```

**配对批准 (Operator → Gateway):**

```typescript
export const NodePairApproveParamsSchema = Type.Object({
  requestId: NonEmptyString,
});
```

**令牌验证 (Node → Gateway):**

```typescript
export const NodePairVerifyParamsSchema = Type.Object({
  nodeId: NonEmptyString,
  token: NonEmptyString,
});
```

### 4.2 命令调用消息

**调用请求 (Gateway → Node):**

```typescript
export const NodeInvokeParamsSchema = Type.Object({
  nodeId: NonEmptyString,            // 目标节点 ID
  command: NonEmptyString,           // 命令名称
  params: Type.Optional(Type.Unknown()), // 命令参数
  timeoutMs: Type.Optional(Type.Integer({ minimum: 0 })), // 超时(毫秒)
  idempotencyKey: NonEmptyString,    // 幂等性密钥
});
```

**调用请求事件 (Gateway 发送给 Node):**

```typescript
export const NodeInvokeRequestEventSchema = Type.Object({
  id: NonEmptyString,                // 请求 ID
  nodeId: NonEmptyString,            // 节点 ID
  command: NonEmptyString,           // 命令名称
  paramsJSON?: Type.String(),        // 参数的 JSON 字符串
  timeoutMs?: Type.Integer({ minimum: 0 }),
  idempotencyKey?: NonEmptyString,
});
```

**调用结果 (Node → Gateway):**

```typescript
export const NodeInvokeResultParamsSchema = Type.Object({
  id: NonEmptyString,                // 对应请求 ID
  nodeId: NonEmptyString,            // 节点 ID
  ok: Type.Boolean(),                // 是否成功
  payload?: Type.Unknown(),          // 成功时的结果
  payloadJSON?: Type.String(),       // 结果的 JSON 字符串
  error?: {
    code?: NonEmptyString,           // 错误码
    message?: NonEmptyString,        // 错误消息
  },
});
```

### 4.3 节点事件消息

**事件发送 (Node → Gateway):**

```typescript
export const NodeEventParamsSchema = Type.Object({
  event: NonEmptyString,             // 事件名称
  payload?: Type.Unknown(),          // 事件数据
  payloadJSON?: Type.String(),       // 数据的 JSON 字符串
});
```

**常见节点事件:**
- `voice.transcript`: 来自节点的语音输入
- `agent.request`: 来自节点的 Agent 请求(深度链接)
- `chat.subscribe`: 节点订阅聊天更新
- `chat.unsubscribe`: 节点取消订阅
- `exec.started`: 执行开始事件
- `exec.finished`: 执行完成事件
- `exec.denied`: 执行被拒绝事件

---

## 5. 认证与授权

### 5.1 设备配对 (持久化认证)

**文件位置:** `src/infra/node-pairing.ts`

节点通过配对流程获得持久化的认证令牌。

**已配对节点数据结构:**

```typescript
export type NodePairingPairedNode = {
  nodeId: string;                    // 节点 ID
  token: string;                     // 认证令牌
  displayName?: string;              // 显示名称
  platform?: string;                 // 平台
  version?: string;                  // 版本
  caps?: string[];                   // 能力列表
  commands?: string[];               // 命令列表
  permissions?: Record<string, boolean>; // 权限映射
  remoteIp?: string;                 // 远程 IP
  createdAtMs: number;               // 创建时间戳
  approvedAtMs: number;              // 批准时间戳
  lastConnectedAtMs?: number;        // 最后连接时间戳
};
```

**存储位置:** `~/.openclaw/nodes/paired.json`

### 5.2 两阶段认证流程

**阶段 1: 初始配对请求 → 创建待审批请求**

```typescript
export async function requestNodePairing(
  req: Omit<NodePairingPendingRequest, "requestId" | "ts">
): Promise<{ status: "pending"; request: NodePairingPendingRequest }>
```

**阶段 2: 审批 → 生成认证令牌并存储到 paired.json**

```typescript
export async function approveNodePairing(
  requestId: string
): Promise<{ requestId: string; node: NodePairingPairedNode }>
```

**阶段 3: 后续连接时验证令牌**

```typescript
export async function verifyNodeToken(
  nodeId: string,
  token: string
): Promise<{ ok: boolean; node?: NodePairingPairedNode }>
```

**待审批请求的 TTL:** 5 分钟 (PENDING_TTL_MS = 5 * 60 * 1000)

**配对请求数据结构:**

```typescript
export type NodePairingPendingRequest = {
  requestId: string;                 // 请求 ID
  nodeId: string;                    // 节点 ID
  displayName?: string;              // 显示名称
  platform?: string;                 // 平台
  version?: string;                  // 版本
  caps?: string[];                   // 能力列表
  commands?: string[];               // 命令列表
  remoteIp?: string;                 // 远程 IP
  ts: number;                        // 请求时间戳
};
```

**存储位置:** `~/.openclaw/nodes/pending.json`

### 5.3 基于角色的访问控制

**文件位置:** `src/gateway/node-command-policy.ts`

节点只能调用明确允许的命令。

**配置示例 (config.yaml):**

```yaml
tools:
  exec:
    node:
      allowlist:
        - "system.run"
        - "system.which"
        - "browser.proxy"
```

**授权检查函数:**

```typescript
export function isNodeCommandAllowed(params: {
  command: string;
  declaredCommands: string[];
  allowlist: string[];
}): { ok: boolean; reason?: string } {
  const { command, declaredCommands, allowlist } = params;

  // 命令必须在节点声明的命令列表中
  if (!declaredCommands.includes(command)) {
    return { ok: false, reason: "not_declared" };
  }

  // 命令必须在允许列表中
  if (!allowlist.includes(command)) {
    return { ok: false, reason: "not_allowed" };
  }

  return { ok: true };
}
```

**解析允许列表:**

```typescript
export function resolveNodeCommandAllowlist(
  cfg: OpenClawConfig,
  nodeSession: NodeSession
): string[] {
  const allowlist = cfg?.tools?.exec?.node?.allowlist;

  if (!Array.isArray(allowlist)) {
    return [];
  }

  return allowlist.filter((cmd) => typeof cmd === "string" && cmd.length > 0);
}
```

---

## 6. 生命周期管理

### 6.1 连接初始化

**文件位置:** `src/node-host/runner.ts` (行 109-162)

**节点连接示例:**

```typescript
const client = new GatewayClient({
  url: `${scheme}://${host}:${port}`,
  token: authToken,                  // 认证令牌
  password: authPassword,            // 密码 (备选)
  instanceId: nodeId,                // 实例 ID
  clientName: GATEWAY_CLIENT_NAMES.NODE_HOST,
  clientDisplayName: displayName,
  role: "node",                      // 角色标识
  scopes: [],
  caps: ["system", ...(browserProxyEnabled ? ["browser"] : [])],
  commands: [
    "system.run",
    "system.which",
    "system.execApprovals.get",
    "system.execApprovals.set",
    ...(browserProxyEnabled ? ["browser.proxy"] : []),
  ],
  onEvent: (evt) => {
    if (evt.event === "node.invoke.request") {
      const payload = coerceNodeInvokePayload(evt.payload);
      void handleInvoke(payload, client, skillBins);
    }
  },
  onConnectError: (err) => {
    console.error(`node host gateway connect failed: ${err.message}`);
  },
  onClose: (code, reason) => {
    console.error(`node host gateway closed (${code}): ${reason}`);
  },
});

client.start();
```

### 6.2 连接生命周期

**文件位置:** `src/gateway/client.ts`

**GatewayClient 类:**

```typescript
export class GatewayClient {
  private ws: WebSocket | null = null;
  private backoffMs = 1000;          // 退避时间
  private closed = false;            // 是否已关闭

  start() {
    this.ws = new WebSocket(url, {
      maxPayload: 25 * 1024 * 1024   // 25 MB 最大负载
    });

    this.ws.on("open", () => {
      // 验证 TLS 指纹(如果配置)
      // 发送连接握手
      this.sendConnect();
    });

    this.ws.on("message", (data) => {
      // 解析 JSON 并处理响应/事件
      const frame = JSON.parse(data.toString());
      this.handleFrame(frame);
    });

    this.ws.on("close", (code, reason) => {
      // 触发重连(指数退避)
      this.reconnect();
    });

    this.ws.on("error", (err) => {
      this.opts.onConnectError?.(err);
    });
  }

  async request<T>(method: string, params: unknown): Promise<T> {
    const requestId = randomUUID();
    return new Promise((resolve, reject) => {
      this.pending.set(requestId, {
        resolve: (value) => resolve(value as T),
        reject,
        expectFinal: true,
      });

      this.ws?.send(JSON.stringify({
        type: "req",
        id: requestId,
        method,
        params,
      }));
    });
  }

  private reconnect() {
    if (this.closed) return;

    setTimeout(() => {
      this.start();
    }, this.backoffMs);

    // 指数退避,最大 30 秒
    this.backoffMs = Math.min(this.backoffMs * 1.5, 30_000);
  }
}
```

### 6.3 断开连接与清理

**文件位置:** `src/gateway/server/ws-connection.ts` (行 242-248)

**断开连接处理:**

```typescript
socket.once("close", (code, reason) => {
  if (client?.connect?.role === "node") {
    const context = buildRequestContext();
    const nodeId = context.nodeRegistry.unregister(connId);

    if (nodeId) {
      // 取消所有订阅
      context.nodeUnsubscribeAll(nodeId);
    }
  }

  // 记录日志
  logger.info(`WebSocket connection closed: connId=${connId}, code=${code}, reason=${reason}`);
});
```

**NodeRegistry 清理:**

```typescript
// src/gateway/node-registry.ts (行 81-97)
unregister(connId: string): string | null {
  const nodeId = this.nodesByConn.get(connId);
  if (!nodeId) {
    return null;
  }

  this.nodesByConn.delete(connId);
  this.nodesById.delete(nodeId);

  // 拒绝所有来自该节点的待处理调用
  for (const [id, pending] of this.pendingInvokes.entries()) {
    if (pending.nodeId === nodeId) {
      clearTimeout(pending.timer);
      pending.reject(new Error(`node disconnected (${pending.command})`));
      this.pendingInvokes.delete(id);
    }
  }

  return nodeId;
}
```

---

## 7. 命令调用:核心交互流程

### 7.1 步骤 1: Gateway 接收调用请求

**文件位置:** `src/gateway/server-methods/nodes.ts` (行 364-450)

**调用处理器:**

```typescript
"node.invoke": async ({ params, respond, context }) => {
  const p = params as {
    nodeId: string;
    command: string;
    params?: unknown;
    timeoutMs?: number;
    idempotencyKey: string;
  };

  // 验证节点是否已连接
  const nodeSession = context.nodeRegistry.get(p.nodeId);
  if (!nodeSession) {
    respond(false, undefined, {
      code: ErrorCodes.UNAVAILABLE,
      message: "node not connected",
    });
    return;
  }

  // 检查命令授权
  const allowlist = resolveNodeCommandAllowlist(cfg, nodeSession);
  const allowed = isNodeCommandAllowed({
    command: p.command,
    declaredCommands: nodeSession.commands,
    allowlist,
  });

  if (!allowed.ok) {
    respond(false, undefined, {
      code: ErrorCodes.INVALID_REQUEST,
      message: "node command not allowed",
      details: { reason: allowed.reason, command: p.command },
    });
    return;
  }

  // 调用并获取结果
  const res = await context.nodeRegistry.invoke({
    nodeId: p.nodeId,
    command: p.command,
    params: p.params,
    timeoutMs: p.timeoutMs,
    idempotencyKey: p.idempotencyKey,
  });

  if (!res.ok) {
    respond(false, undefined, {
      code: ErrorCodes.UNAVAILABLE,
      message: res.error?.message || "invoke failed",
    });
    return;
  }

  respond(true, { ok: true, payload: res.payload }, undefined);
}
```

### 7.2 步骤 2: Gateway 发送调用请求事件

**文件位置:** `src/gateway/node-registry.ts` (行 107-155)

**invoke 方法:**

```typescript
async invoke(params: {
  nodeId: string;
  command: string;
  params?: unknown;
  timeoutMs?: number;
  idempotencyKey?: string;
}): Promise<NodeInvokeResult> {
  const node = this.nodesById.get(params.nodeId);
  if (!node) {
    return {
      ok: false,
      error: { code: "NOT_CONNECTED", message: "node not connected" },
    };
  }

  const requestId = randomUUID();
  const payload = {
    id: requestId,
    nodeId: params.nodeId,
    command: params.command,
    paramsJSON: params.params !== undefined
      ? JSON.stringify(params.params)
      : null,
    timeoutMs: params.timeoutMs,
    idempotencyKey: params.idempotencyKey,
  };

  // 发送事件到节点
  const ok = this.sendEventToSession(node, "node.invoke.request", payload);
  if (!ok) {
    return {
      ok: false,
      error: { code: "UNAVAILABLE", message: "failed to send invoke to node" },
    };
  }

  // 等待结果(带超时)
  const timeoutMs = params.timeoutMs ?? 30_000;
  return await new Promise<NodeInvokeResult>((resolve, reject) => {
    const timer = setTimeout(() => {
      this.pendingInvokes.delete(requestId);
      resolve({
        ok: false,
        error: { code: "TIMEOUT", message: "node invoke timed out" },
      });
    }, timeoutMs);

    this.pendingInvokes.set(requestId, {
      nodeId: params.nodeId,
      command: params.command,
      resolve,
      reject,
      timer,
    });
  });
}

private sendEventToSession(
  node: NodeSession,
  event: string,
  payload: unknown
): boolean {
  try {
    node.client.socket.send(JSON.stringify({
      type: "event",
      event,
      payload,
    }));
    return true;
  } catch {
    return false;
  }
}
```

### 7.3 步骤 3: Node 处理命令

**文件位置:** `src/node-host/runner.ts` (行 133-142)

**事件监听器:**

```typescript
onEvent: (evt) => {
  if (evt.event !== "node.invoke.request") {
    return;
  }

  const payload = coerceNodeInvokePayload(evt.payload);
  if (!payload) {
    return;
  }

  void handleInvoke(payload, client, skillBins);
}
```

**文件位置:** `src/node-host/invoke.ts` (行 227-308)

**命令处理函数:**

```typescript
async function handleInvoke(
  payload: NodeInvokeRequestPayload,
  client: GatewayClient,
  skillBins: SkillBinsProvider
) {
  const { id, nodeId, command, paramsJSON, timeoutMs, idempotencyKey } = payload;

  try {
    // 解析参数
    const params = paramsJSON ? JSON.parse(paramsJSON) : {};

    let result: unknown;

    // 根据命令类型执行
    if (command === "system.run") {
      const systemParams = params as SystemRunParams;
      result = await handleSystemRun(systemParams);
    } else if (command === "system.which") {
      const systemParams = params as SystemWhichParams;
      result = await handleSystemWhich(systemParams);
    } else if (command === "browser.proxy") {
      const browserParams = params as BrowserProxyParams;
      result = await handleBrowserProxy(browserParams);
    }
    // ... 其他命令处理器

    // 发送结果回 Gateway
    await client.request("node.invoke.result", {
      id,
      nodeId,
      ok: true,
      payload: result,
      payloadJSON: JSON.stringify(result),
    });
  } catch (err) {
    // 发送错误回 Gateway
    await client.request("node.invoke.result", {
      id,
      nodeId,
      ok: false,
      error: {
        code: "COMMAND_FAILED",
        message: err.message || "unknown error",
      },
    });
  }
}
```

**system.run 命令处理示例:**

```typescript
async function handleSystemRun(params: SystemRunParams): Promise<SystemRunResult> {
  const { command, args, cwd, env, timeoutMs } = params;

  // 执行系统命令
  const proc = spawn(command, args || [], {
    cwd: cwd || process.cwd(),
    env: { ...process.env, ...env },
    timeout: timeoutMs || 30_000,
  });

  let stdout = "";
  let stderr = "";

  proc.stdout?.on("data", (data) => {
    stdout += data.toString();
  });

  proc.stderr?.on("data", (data) => {
    stderr += data.toString();
  });

  return new Promise((resolve, reject) => {
    proc.on("exit", (code) => {
      resolve({
        exitCode: code ?? -1,
        stdout,
        stderr,
      });
    });

    proc.on("error", (err) => {
      reject(err);
    });
  });
}
```

### 7.4 步骤 4: Node 返回结果

```typescript
await client.request("node.invoke.result", {
  id: requestId,
  nodeId,
  ok: true,
  payload: result,
  payloadJSON: JSON.stringify(result),
});
```

### 7.5 步骤 5: Gateway 处理结果

**文件位置:** `src/gateway/server-methods/nodes.ts` (行 451-489)

**结果处理器:**

```typescript
"node.invoke.result": async ({ params, respond, context, client }) => {
  const p = params as {
    id: string;
    nodeId: string;
    ok: boolean;
    payload?: unknown;
    payloadJSON?: string | null;
    error?: { code?: string; message?: string } | null;
  };

  // 验证调用者是结果中提到的节点
  const callerNodeId = client?.connect?.device?.id ?? client?.connect?.client?.id;
  if (callerNodeId && callerNodeId !== p.nodeId) {
    respond(false, undefined, {
      code: ErrorCodes.INVALID_REQUEST,
      message: "nodeId mismatch",
    });
    return;
  }

  // 解析待处理的调用 Promise
  const ok = context.nodeRegistry.handleInvokeResult({
    id: p.id,
    nodeId: p.nodeId,
    ok: p.ok,
    payload: p.payload,
    payloadJSON: p.payloadJSON ?? null,
    error: p.error ?? null,
  });

  if (!ok) {
    // 延迟到达的结果(超时后) - 无害
    respond(true, { ok: true, ignored: true }, undefined);
    return;
  }

  respond(true, { ok: true }, undefined);
}
```

**NodeRegistry 结果处理:**

```typescript
// src/gateway/node-registry.ts (行 157-181)
handleInvokeResult(params: {
  id: string;
  nodeId: string;
  ok: boolean;
  payload?: unknown;
  payloadJSON?: string | null;
  error?: { code?: string; message?: string } | null;
}): boolean {
  const pending = this.pendingInvokes.get(params.id);
  if (!pending) {
    return false;  // 结果已超时或无效
  }

  if (pending.nodeId !== params.nodeId) {
    return false;  // 节点 ID 不匹配
  }

  clearTimeout(pending.timer);
  this.pendingInvokes.delete(params.id);

  // 解析 Promise
  pending.resolve({
    ok: params.ok,
    payload: params.payload,
    payloadJSON: params.payloadJSON ?? null,
    error: params.error ?? null,
  });

  return true;
}
```

---

## 8. 节点事件处理

**文件位置:** `src/gateway/server-node-events.ts`

节点可以随时向 Gateway 发出事件。

**事件处理函数:**

```typescript
export const handleNodeEvent = async (
  ctx: NodeEventContext,
  nodeId: string,
  evt: NodeEvent
) => {
  switch (evt.event) {
    case "voice.transcript": {
      // 来自节点的语音输入 → 触发 Agent
      const payload = JSON.parse(evt.payloadJSON);
      await agentCommand({
        message: payload.text,
        sessionId: payload.sessionId,
        messageChannel: "node",
        deliver: true,
      }, runtime, ctx.deps);
      break;
    }

    case "agent.request": {
      // 来自节点的 Agent 请求(深度链接)
      const link = JSON.parse(evt.payloadJSON);
      await agentCommand({
        message: link.message,
        sessionId: link.sessionId,
        sessionKey: link.sessionKey,
        deliver: link.deliver,
      }, runtime, ctx.deps);
      break;
    }

    case "chat.subscribe": {
      // 节点订阅聊天更新
      const payload = JSON.parse(evt.payloadJSON);
      ctx.nodeSubscribe(nodeId, payload.sessionKey);
      break;
    }

    case "chat.unsubscribe": {
      // 节点取消订阅
      const payload = JSON.parse(evt.payloadJSON);
      ctx.nodeUnsubscribe(nodeId, payload.sessionKey);
      break;
    }

    case "exec.started":
    case "exec.finished":
    case "exec.denied": {
      // 执行生命周期事件
      const payload = JSON.parse(evt.payloadJSON);
      enqueueSystemEvent(payload.text, {
        sessionKey: payload.sessionKey,
        contextKey: `exec:${payload.runId}`
      });
      break;
    }
  }
};
```

**订阅管理:**

```typescript
// src/gateway/server-chat.ts
export function createNodeSubscriptionManager() {
  // nodeId → Set<sessionKey>
  const subscriptionsByNode = new Map<string, Set<string>>();

  // sessionKey → Set<nodeId>
  const subscriptionsBySession = new Map<string, Set<string>>();

  return {
    subscribe: (nodeId: string, sessionKey: string) => {
      if (!subscriptionsByNode.has(nodeId)) {
        subscriptionsByNode.set(nodeId, new Set());
      }
      subscriptionsByNode.get(nodeId)!.add(sessionKey);

      if (!subscriptionsBySession.has(sessionKey)) {
        subscriptionsBySession.set(sessionKey, new Set());
      }
      subscriptionsBySession.get(sessionKey)!.add(nodeId);
    },

    unsubscribe: (nodeId: string, sessionKey: string) => {
      subscriptionsByNode.get(nodeId)?.delete(sessionKey);
      subscriptionsBySession.get(sessionKey)?.delete(nodeId);
    },

    unsubscribeAll: (nodeId: string) => {
      const sessions = subscriptionsByNode.get(nodeId);
      if (sessions) {
        for (const sessionKey of sessions) {
          subscriptionsBySession.get(sessionKey)?.delete(nodeId);
        }
      }
      subscriptionsByNode.delete(nodeId);
    },

    getSubscribedNodes: (sessionKey: string): string[] => {
      return Array.from(subscriptionsBySession.get(sessionKey) || []);
    },
  };
}
```

---

## 9. 负载均衡与节点选择

### 9.1 节点发现与状态

**文件位置:** `src/gateway/server-methods/nodes.ts` (行 229-309)

**节点列表方法:**

```typescript
"node.list": async ({ params, respond, context }) => {
  await respondUnavailableOnThrow(respond, async () => {
    // 获取已配对的节点
    const list = await listDevicePairing();
    const pairedById = new Map(list.paired
      .filter((entry) => isNodeEntry(entry))
      .map((entry) => [entry.deviceId, entry]));

    // 获取当前已连接的节点
    const connected = context.nodeRegistry.listConnected();
    const connectedById = new Map(connected.map((n) => [n.nodeId, n]));

    // 合并配对和已连接的节点
    const nodeIds = new Set([...pairedById.keys(), ...connectedById.keys()]);

    const nodes = [...nodeIds].map((nodeId) => {
      const paired = pairedById.get(nodeId);
      const live = connectedById.get(nodeId);

      return {
        nodeId,
        displayName: live?.displayName ?? paired?.displayName,
        platform: live?.platform ?? paired?.platform,
        version: live?.version ?? paired?.version,
        connected: Boolean(live),
        paired: Boolean(paired),
        connectedAtMs: live?.connectedAtMs,
        lastConnectedAtMs: paired?.lastConnectedAtMs,
        caps: live?.caps ?? paired?.caps ?? [],
        commands: live?.commands ?? paired?.commands ?? [],
        remoteIp: live?.remoteIp ?? paired?.remoteIp,
        // ... 更多元数据
      };
    });

    // 排序:已连接的优先,然后按名称排序
    nodes.sort((a, b) => {
      if (a.connected !== b.connected) {
        return a.connected ? -1 : 1;
      }
      return (a.displayName ?? "").localeCompare(b.displayName ?? "");
    });

    respond(true, { ts: Date.now(), nodes }, undefined);
  });
}
```

**列出已连接节点:**

```typescript
// src/gateway/node-registry.ts
listConnected(): NodeSession[] {
  return Array.from(this.nodesById.values());
}

get(nodeId: string): NodeSession | undefined {
  return this.nodesById.get(nodeId);
}
```

### 9.2 节点选择

**执行前的节点可用性验证:**

```typescript
const nodeSession = context.nodeRegistry.get(nodeId);
if (!nodeSession) {
  // 节点未连接
  return {
    ok: false,
    error: { code: "NOT_CONNECTED", message: "node not connected" }
  };
}
```

**能力检查:**

```typescript
function nodeHasCapability(node: NodeSession, capability: string): boolean {
  return node.caps.includes(capability);
}

// 使用示例
if (!nodeHasCapability(nodeSession, "browser")) {
  return {
    ok: false,
    error: { code: "CAPABILITY_MISSING", message: "node lacks browser capability" }
  };
}
```

---

## 10. 错误处理与降级

### 10.1 超时管理

**默认超时:**

```typescript
// 节点调用的默认超时为 30 秒
const timeoutMs = typeof params.timeoutMs === "number"
  ? params.timeoutMs
  : 30_000;
```

**超时处理:**

```typescript
const timer = setTimeout(() => {
  this.pendingInvokes.delete(requestId);
  resolve({
    ok: false,
    error: { code: "TIMEOUT", message: "node invoke timed out" },
  });
}, timeoutMs);
```

### 10.2 连接错误处理

**指数退避重连:**

```typescript
// src/gateway/client.ts
private reconnect() {
  if (this.closed) return;

  setTimeout(() => {
    this.start();
  }, this.backoffMs);

  // 指数退避,最大 30 秒
  this.backoffMs = Math.min(this.backoffMs * 1.5, 30_000);
}
```

**连接错误回调:**

```typescript
onConnectError: (err) => {
  logger.error(`node host gateway connect failed: ${err.message}`);
  // 触发重连机制
},

onClose: (code, reason) => {
  logger.error(`node host gateway closed (${code}): ${reason}`);
  // 触发重连机制
}
```

### 10.3 命令授权失败

```typescript
const allowed = isNodeCommandAllowed({
  command,
  declaredCommands: nodeSession.commands,
  allowlist,
});

if (!allowed.ok) {
  respond(false, undefined, {
    code: ErrorCodes.INVALID_REQUEST,
    message: "node command not allowed",
    details: {
      reason: allowed.reason,  // "not_declared" | "not_allowed"
      command
    },
  });
}
```

### 10.4 待审批请求过期

**自动清理过期的配对请求:**

```typescript
const PENDING_TTL_MS = 5 * 60 * 1000;  // 5 分钟

function pruneExpiredPending(
  pendingById: Record<string, NodePairingPendingRequest>,
  nowMs: number,
) {
  for (const [id, req] of Object.entries(pendingById)) {
    if (nowMs - req.ts > PENDING_TTL_MS) {
      delete pendingById[id];
    }
  }
}
```

### 10.5 命令执行失败

**Node 端错误处理:**

```typescript
try {
  const result = await handleSystemRun(params);
  await client.request("node.invoke.result", {
    id,
    nodeId,
    ok: true,
    payload: result,
  });
} catch (err) {
  await client.request("node.invoke.result", {
    id,
    nodeId,
    ok: false,
    error: {
      code: "COMMAND_FAILED",
      message: err.message || "unknown error",
      stack: err.stack,  // 调试信息
    },
  });
}
```

---

## 11. 完整数据流图

```
NODE (iOS/Android/macOS/Headless)          GATEWAY
    |                                           |
    |-- WebSocket Connect (WS) ----------------->|
    |    {                                       |
    |      role: "node",                         |
    |      caps: ["system", "browser"],          |
    |      commands: ["system.run", ...]         |
    |      auth: { token: "..." }                |
    |    }                                       |
    |                                            | NodeRegistry.register()
    |                                            | 验证 token
    |                                            | 保存会话
    |<-- Connect Response (authenticated) ------|
    |    { ok: true, connId: "..." }             |
    |                                            |
    |                          [Node 等待命令]     |
    |                                            |
    | [Gateway 收到调用请求]                        |
    |                                            | 检查节点是否连接
    |                                            | 检查命令授权
    |<-- node.invoke.request event -------------|
    |    {                                       |
    |      id: "req-123",                        |
    |      nodeId: "node-xyz",                   |
    |      command: "system.run",                |
    |      paramsJSON: "{...}"                   |
    |    }                                       |
    |                                            |
    | [Node 执行命令]                               |
    | system.run, system.which, browser.proxy... |
    |                                            |
    |-- node.invoke.result request ---------->|
    |    {                                       |
    |      id: "req-123",                        |
    |      nodeId: "node-xyz",                   |
    |      ok: true,                             |
    |      payloadJSON: "{...}"                  |
    |    }                                       |
    |<-- Response (ok: true) ---------------------|
    |                                            | NodeRegistry.handleInvokeResult()
    |                                            | 解析 pending Promise
    |                                            |
    | [Node 可随时发出事件]                          |
    |-- node.event request (可选) ------------>|
    |    {                                       |
    |      event: "voice.transcript",           |
    |      payloadJSON: "{...}"                  |
    |    }                                       |
    |<-- Response (ok: true) ---------------------|
    |                                            | 处理事件
    |                                            |
    |-- Close/Disconnect --------------------->|
    |                                            | NodeRegistry.unregister()
    |                                            | 取消待处理调用
    |                                            | 取消所有订阅
```

---

## 12. 关键代码示例

### 12.1 最小节点连接示例

**文件位置:** `src/node-host/runner.ts`

```typescript
import { GatewayClient } from "../gateway/client.js";
import { GATEWAY_CLIENT_NAMES } from "../gateway/constants.js";

const client = new GatewayClient({
  url: "ws://gateway:18789",
  token: authToken,
  role: "node",
  clientName: GATEWAY_CLIENT_NAMES.NODE_HOST,
  caps: ["system", "browser"],
  commands: ["system.run", "system.which", "browser.proxy"],
  onEvent: (evt) => {
    if (evt.event === "node.invoke.request") {
      handleInvoke(evt.payload, client);
    }
  },
  onConnectError: (err) => {
    console.error(`Gateway connect failed: ${err.message}`);
  },
  onClose: (code, reason) => {
    console.error(`Gateway closed (${code}): ${reason}`);
  },
});

client.start();
```

### 12.2 Gateway 调用处理器

**文件位置:** `src/gateway/server-methods/nodes.ts`

```typescript
"node.invoke": async ({ params, respond, context }) => {
  const { nodeId, command, params: cmdParams, timeoutMs, idempotencyKey } = params;

  // 验证节点连接
  const nodeSession = context.nodeRegistry.get(nodeId);
  if (!nodeSession) {
    respond(false, undefined, errorShape(ErrorCodes.UNAVAILABLE, "node not connected"));
    return;
  }

  // 检查命令授权
  const allowlist = resolveNodeCommandAllowlist(cfg, nodeSession);
  const allowed = isNodeCommandAllowed({
    command,
    declaredCommands: nodeSession.commands,
    allowlist,
  });

  if (!allowed.ok) {
    respond(false, undefined, errorShape(
      ErrorCodes.INVALID_REQUEST,
      "node command not allowed"
    ));
    return;
  }

  // 调用节点
  const res = await context.nodeRegistry.invoke({
    nodeId,
    command,
    params: cmdParams,
    timeoutMs,
    idempotencyKey,
  });

  if (!res.ok) {
    respond(false, undefined, errorShape(
      ErrorCodes.UNAVAILABLE,
      res.error?.message
    ));
    return;
  }

  respond(true, { ok: true, payload: res.payload }, undefined);
}
```

### 12.3 命令授权检查

**文件位置:** `src/gateway/node-command-policy.ts`

```typescript
export function isNodeCommandAllowed(params: {
  command: string;
  declaredCommands: string[];
  allowlist: string[];
}): { ok: boolean; reason?: string } {
  const { command, declaredCommands, allowlist } = params;

  // 命令必须在节点声明的命令列表中
  if (!declaredCommands.includes(command)) {
    return { ok: false, reason: "not_declared" };
  }

  // 命令必须在允许列表中
  if (!allowlist.includes(command)) {
    return { ok: false, reason: "not_allowed" };
  }

  return { ok: true };
}

export function resolveNodeCommandAllowlist(
  cfg: OpenClawConfig,
  nodeSession: NodeSession
): string[] {
  const allowlist = cfg?.tools?.exec?.node?.allowlist;

  if (!Array.isArray(allowlist)) {
    return [];
  }

  return allowlist.filter((cmd) => typeof cmd === "string" && cmd.length > 0);
}
```

### 12.4 Node 端命令处理

**文件位置:** `src/node-host/invoke.ts`

```typescript
async function handleInvoke(
  payload: NodeInvokeRequestPayload,
  client: GatewayClient,
  skillBins: SkillBinsProvider
) {
  const { id, nodeId, command, paramsJSON } = payload;

  try {
    const params = paramsJSON ? JSON.parse(paramsJSON) : {};
    let result: unknown;

    switch (command) {
      case "system.run": {
        const systemParams = params as SystemRunParams;
        result = await handleSystemRun(systemParams);
        break;
      }

      case "system.which": {
        const systemParams = params as SystemWhichParams;
        result = await handleSystemWhich(systemParams);
        break;
      }

      case "browser.proxy": {
        const browserParams = params as BrowserProxyParams;
        result = await handleBrowserProxy(browserParams);
        break;
      }

      default: {
        throw new Error(`Unknown command: ${command}`);
      }
    }

    // 发送成功结果
    await client.request("node.invoke.result", {
      id,
      nodeId,
      ok: true,
      payload: result,
      payloadJSON: JSON.stringify(result),
    });
  } catch (err) {
    // 发送错误结果
    await client.request("node.invoke.result", {
      id,
      nodeId,
      ok: false,
      error: {
        code: "COMMAND_FAILED",
        message: err.message || "unknown error",
      },
    });
  }
}
```

### 12.5 系统命令执行示例

**文件位置:** `src/node-host/invoke.ts`

```typescript
async function handleSystemRun(params: SystemRunParams): Promise<SystemRunResult> {
  const { command, args, cwd, env, timeoutMs } = params;

  return new Promise((resolve, reject) => {
    const proc = spawn(command, args || [], {
      cwd: cwd || process.cwd(),
      env: { ...process.env, ...env },
      timeout: timeoutMs || 30_000,
      shell: false,
    });

    let stdout = "";
    let stderr = "";

    proc.stdout?.on("data", (data) => {
      stdout += data.toString();
    });

    proc.stderr?.on("data", (data) => {
      stderr += data.toString();
    });

    proc.on("exit", (code) => {
      resolve({
        exitCode: code ?? -1,
        stdout,
        stderr,
      });
    });

    proc.on("error", (err) => {
      reject(err);
    });
  });
}
```

---

## 13. 配置与常量

### 13.1 服务器常量

**文件位置:** `src/gateway/server-constants.ts`

```typescript
// WebSocket 配置
export const MAX_BUFFERED_BYTES = 10 * 1024 * 1024;  // 10 MB
export const MAX_PAYLOAD_BYTES = 25 * 1024 * 1024;   // 25 MB (节点连接)
export const TICK_INTERVAL_MS = 30_000;              // 30 秒活跃性检查

// 超时配置
export const DEFAULT_INVOKE_TIMEOUT_MS = 30_000;     // 30 秒
export const MAX_INVOKE_TIMEOUT_MS = 300_000;        // 5 分钟

// 配对配置
export const PENDING_TTL_MS = 5 * 60 * 1000;         // 5 分钟
```

### 13.2 存储位置

**节点配对数据:**
- 已配对节点: `~/.openclaw/nodes/paired.json`
- 待审批请求: `~/.openclaw/nodes/pending.json`

**已配对节点文件结构:**

```json
{
  "paired": [
    {
      "nodeId": "device-abc123",
      "token": "secret-token-xyz",
      "displayName": "My iPhone",
      "platform": "iOS",
      "version": "1.0.0",
      "caps": ["system", "browser"],
      "commands": ["system.run", "system.which"],
      "createdAtMs": 1707926400000,
      "approvedAtMs": 1707926460000,
      "lastConnectedAtMs": 1707930000000
    }
  ]
}
```

**待审批请求文件结构:**

```json
{
  "pending": {
    "req-123": {
      "requestId": "req-123",
      "nodeId": "device-def456",
      "displayName": "My MacBook",
      "platform": "macOS",
      "version": "1.0.0",
      "caps": ["system"],
      "commands": ["system.run", "system.which"],
      "ts": 1707926400000
    }
  }
}
```

### 13.3 配置文件示例

**文件位置:** `~/.openclaw/config.yaml`

```yaml
gateway:
  mode: local
  bind: loopback
  port: 18789
  tls:
    enabled: false

tools:
  exec:
    node:
      allowlist:
        - "system.run"
        - "system.which"
        - "browser.proxy"

auth:
  # 节点认证通过配对流程管理
```

---

## 总结

### Gateway-Node 架构的核心特点

1. **Node 是执行主机**
   - iOS/Android/macOS 应用或无头节点主机
   - 通过 WebSocket 连接到 Gateway
   - 提供特定能力(系统命令、浏览器代理等)

2. **认证机制**
   - 两阶段设备配对流程
   - 持久化认证令牌(存储在 `paired.json`)
   - 5 分钟的待审批请求 TTL

3. **通信协议**
   - 双向 JSON-RPC over WebSocket
   - 三种帧类型: `req`, `res`, `event`
   - 每个命令默认 30 秒超时
   - 最大负载 25 MB

4. **授权控制**
   - 基于角色的命令允许列表
   - 节点必须声明支持的命令
   - Gateway 验证命令是否在允许列表中

5. **生命周期管理**
   - 自动重连(指数退避,最大 30 秒)
   - 断开连接时自动清理待处理调用
   - 30 秒心跳间隔

6. **命令调用流程**
   1. Gateway 接收调用请求
   2. 验证节点连接和命令授权
   3. 发送 `node.invoke.request` 事件到 Node
   4. Node 执行命令并返回结果
   5. Gateway 解析待处理的 Promise

7. **事件系统**
   - Node 可发出事件: `voice.transcript`, `agent.request`, `chat.subscribe` 等
   - 支持订阅/取消订阅机制
   - 执行生命周期事件: `exec.started`, `exec.finished`, `exec.denied`

8. **错误处理**
   - 超时自动返回错误
   - 指数退避重连
   - 命令授权失败保护
   - 待审批请求自动过期

这个架构实现了安全的远程命令执行,同时保持了细粒度的访问控制和适当的错误处理机制。

---

**文档版本:** 1.0
**最后更新:** 2026-02-19
**相关文档:**
- [Agent Loop 分析](./agent-loop-analysis.md)
- [WhatsApp 集成分析](./whatsapp-integration-analysis.md)
- [消息平台对比](./messaging-platforms-comparison.md)
- [Gateway 和 Chat Channels 分析](./gateway-and-channels-analysis.md)
- [Gateway 与 Agent 交互分析](./gateway-agent-interaction-analysis.md)
