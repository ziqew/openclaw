# Gateway 和 Chat Channels 源代码分析

> 本文档详细分析了 OpenClaw 的 Gateway（网关）和 Chat Channels（聊天通道）架构和实现

## 目录

1. [概述](#概述)
2. [Gateway 架构](#gateway-架构)
3. [Chat Channels 架构](#chat-channels-架构)
4. [消息路由机制](#消息路由机制)
5. [插件运行时系统](#插件运行时系统)
6. [配置和管理](#配置和管理)
7. [架构模式总结](#架构模式总结)

---

## 概述

OpenClaw 采用了 **Gateway（网关）+ Plugin（插件）** 的架构设计，将核心系统与各个消息平台的实现解耦。Gateway 负责统一管理所有通道的生命周期、消息路由和配置，而每个 Chat Channel 作为独立的插件实现特定平台的通信协议。

### 核心组件关系

```
┌─────────────────────────────────────────────────────────────┐
│                     OpenClaw Gateway                        │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  HTTP/WebSocket Server                                │ │
│  │  - Control UI (浏览器管理界面)                        │ │
│  │  - Canvas Host (Agent-UI 通信)                        │ │
│  │  - Hooks API (Webhook 端点)                           │ │
│  │  - OpenAI Compatible API                              │ │
│  └───────────────────────────────────────────────────────┘ │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  Channel Manager (通道管理器)                         │ │
│  │  - 生命周期管理                                       │ │
│  │  - 状态跟踪                                           │ │
│  │  - 配置热重载                                         │ │
│  └───────────────────────────────────────────────────────┘ │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  Plugin Runtime (插件运行时)                          │ │
│  │  - 通道注册表                                         │ │
│  │  - 消息路由                                           │ │
│  │  - 出站队列                                           │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
              ↓                ↓                ↓
    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
    │  WhatsApp   │  │  Telegram   │  │   Discord   │
    │   Plugin    │  │   Plugin    │  │   Plugin    │
    └─────────────┘  └─────────────┘  └─────────────┘
```

---

## Gateway 架构

### 1. Gateway 启动流程

#### 主入口点

**文件**: `/home/runner/work/openclaw/openclaw/src/gateway/server.impl.ts`

**函数**: `startGatewayServer(port, opts)`

```typescript
// 启动流程（第 160-402 行）
export async function startGatewayServer(
  port: number,
  opts: GatewayServerOptions = {}
): Promise<GatewayServer> {

  // 1. 加载和验证配置
  const cfg = await loadGatewayConfig(opts.configPath);

  // 2. 迁移遗留配置项
  const migratedCfg = applyConfigMigrations(cfg);

  // 3. 加载插件注册表
  await loadGatewayPlugins(migratedCfg);

  // 4. 创建通道管理器
  const channelManager = createChannelManager({
    loadConfig: () => cfg,
    channelLogs: opts.channelLogs,
    channelRuntimeEnvs: opts.channelRuntimeEnvs
  });

  // 5. 创建运行时状态
  const runtime = await createGatewayRuntimeState({
    port,
    config: migratedCfg,
    channelManager
  });

  // 6. 启动辅助服务（Sidecars）
  await startGatewaySidecars({
    runtime,
    channelManager,
    abortSignal: opts.abortSignal
  });

  // 7. 启动所有配置的通道
  await startChannels(channelManager, migratedCfg);

  return {
    runtime,
    channelManager,
    stop: async () => {
      await channelManager.stopAll();
      await runtime.httpServer.close();
    }
  };
}
```

**启动阶段详解**：

1. **配置加载**（第 175-195 行）
   - 读取 `openclaw.json` 或指定配置文件
   - 验证配置架构（Zod validation）
   - 应用环境变量覆盖
   - 迁移旧版配置格式

2. **插件加载**（第 200-225 行）
   - 扫描 `extensions/` 目录
   - 加载配置的插件路径
   - 注册通道插件到全局注册表
   - 解决插件依赖关系

3. **通道管理器创建**（第 230-260 行）
   - 创建通道生命周期管理器
   - 初始化状态跟踪 Map
   - 设置配置重载监听器
   - 准备 AbortController 池

4. **运行时状态创建**（第 265-290 行）
   - 创建 HTTP 服务器
   - 创建 WebSocket 服务器
   - 初始化客户端跟踪
   - 设置请求路由器

5. **辅助服务启动**（第 295-340 行）
   - Gmail 邮件监控器
   - 浏览器控制接口
   - 内部 Hooks 处理器
   - 心跳广播器

6. **通道启动**（第 345-402 行）
   - 遍历所有配置的通道
   - 为每个账户启动监听器
   - 设置重连策略
   - 报告启动状态

### 2. HTTP/WebSocket 服务器

**文件**: `/home/runner/work/openclaw/openclaw/src/gateway/server-http.ts`

#### 服务器创建

```typescript
// 第 127-198 行
export function createGatewayHttpServer(options: {
  port: number;
  config: OpenClawConfig;
  channelManager: ChannelManager;
  clients: Set<WebSocket>;
}): http.Server {

  const server = http.createServer((req, res) => {
    handleRequest(req, res, options);
  });

  // WebSocket 升级处理
  server.on('upgrade', (req, socket, head) => {
    handleWebSocketUpgrade(req, socket, head, options);
  });

  server.listen(options.port, () => {
    console.log(`Gateway listening on port ${options.port}`);
  });

  return server;
}
```

#### HTTP 请求路由

```typescript
// 第 476-595 行
async function handleRequest(
  req: http.IncomingMessage,
  res: http.ServerResponse,
  options: GatewayHttpOptions
): Promise<void> {
  const url = req.url || '/';
  const method = req.method || 'GET';

  // 1. Hooks API: POST /hooks/*
  if (method === 'POST' && url.startsWith('/hooks/')) {
    return handleHooksRequest(req, res, options);
  }

  // 2. Tools Invoke: POST /tools/invoke
  if (method === 'POST' && url === '/tools/invoke') {
    return handleToolsInvokeHttpRequest(req, res, options);
  }

  // 3. Slack Webhooks: POST /slack/*
  if (method === 'POST' && url.startsWith('/slack/')) {
    return handleSlackHttpRequest(req, res, options);
  }

  // 4. 插件 HTTP 端点
  if (url.startsWith('/plugin/')) {
    return handlePluginRequest(req, res, options);
  }

  // 5. OpenResponses API: POST /v1/responses
  if (method === 'POST' && url === '/v1/responses') {
    return handleOpenResponsesHttpRequest(req, res, options);
  }

  // 6. OpenAI Compatible API: POST /v1/chat/completions
  if (method === 'POST' && url === '/v1/chat/completions') {
    return handleOpenAiHttpRequest(req, res, options);
  }

  // 7. Canvas Host UI: /canvas/*
  if (url.startsWith('/canvas/')) {
    return handleA2uiHttpRequest(req, res, options);
  }

  // 8. Control UI: /*
  return handleControlUiHttpRequest(req, res, options);
}
```

#### WebSocket 端点

```typescript
// 第 600-680 行
function handleWebSocketUpgrade(
  req: http.IncomingMessage,
  socket: net.Socket,
  head: Buffer,
  options: GatewayHttpOptions
): void {
  const url = req.url || '/';

  // 1. Gateway 客户端连接: /gateway
  if (url === '/gateway' || url.startsWith('/gateway?')) {
    return upgradeGatewayWebSocket(req, socket, head, options);
  }

  // 2. Canvas Host 连接: /canvas/host
  if (url === '/canvas/host' || url.startsWith('/canvas/host?')) {
    return upgradeCanvasHostWebSocket(req, socket, head, options);
  }

  // 3. Control UI 连接: /control
  if (url === '/control' || url.startsWith('/control?')) {
    return upgradeControlUiWebSocket(req, socket, head, options);
  }

  // 未知端点
  socket.destroy();
}
```

### 3. Hooks 系统

**文件**: `/home/runner/work/openclaw/openclaw/src/gateway/server-http-hooks.ts`

Hooks 系统允许外部服务通过 HTTP Webhook 触发 Agent 执行。

#### Hooks 处理器

```typescript
// 第 190-436 行
function createHooksRequestHandler(options: {
  config: OpenClawConfig;
  channelManager: ChannelManager;
}): (req, res) => Promise<void> {

  return async (req, res) => {
    // 1. 解析请求路径和参数
    const url = new URL(req.url, `http://localhost`);
    const pathParts = url.pathname.split('/').filter(Boolean);
    // /hooks/<action> 或 /hooks/<basePath>/<action>

    const action = pathParts[pathParts.length - 1];
    const basePath = pathParts.slice(1, -1).join('/');

    // 2. 身份验证
    const authResult = await authenticateHookRequest(req, options.config);
    if (!authResult.authenticated) {
      return sendAuthError(res, authResult.reason);
    }

    // 3. 速率限制检查
    const rateLimitResult = checkHookRateLimit(req);
    if (rateLimitResult.limited) {
      return sendRateLimitError(res);
    }

    // 4. 读取请求体
    const body = await readRequestBody(req);
    const payload = JSON.parse(body);

    // 5. 查找 Hook 映射
    const mapping = findHookMapping(basePath, action, options.config);

    if (mapping) {
      // 使用自定义映射
      return executeHookMapping(mapping, payload, options);
    }

    // 6. 默认行为
    switch (action) {
      case 'wake':
        return handleWakeHook(payload, options);

      case 'agent':
        return handleAgentHook(payload, options);

      default:
        return send404(res);
    }
  };
}
```

#### Hook 映射配置

```typescript
// 配置示例
{
  "gateway": {
    "hooks": {
      "auth": {
        "mode": "token",  // "local" | "token" | "both" | "tailscale"
        "tokens": ["secret-token-1", "secret-token-2"]
      },
      "basePath": "webhooks",  // /hooks/webhooks/*
      "mappings": [
        {
          "action": "github-push",
          "transform": {
            "prompt": "New commit: {{payload.commits[0].message}}",
            "channel": "telegram",
            "to": "123456789"
          }
        }
      ]
    }
  }
}
```

#### 身份验证流程

```typescript
// 第 80-145 行
async function authenticateHookRequest(
  req: http.IncomingMessage,
  config: OpenClawConfig
): Promise<AuthResult> {
  const hooksCfg = config.gateway?.hooks;
  const authMode = hooksCfg?.auth?.mode ?? 'token';

  // 1. Tailscale 模式：仅 Tailscale IP
  if (authMode === 'tailscale') {
    return checkTailscaleAuth(req);
  }

  // 2. Local 模式：仅本地/私有 IP
  if (authMode === 'local') {
    return checkLocalAuth(req);
  }

  // 3. Token 模式：Bearer token 或 X-OpenClaw-Token
  if (authMode === 'token') {
    const token = extractToken(req);
    if (!token) {
      return { authenticated: false, reason: 'no_token' };
    }

    const validTokens = hooksCfg?.auth?.tokens ?? [];
    if (validTokens.includes(token)) {
      return { authenticated: true };
    }

    return { authenticated: false, reason: 'invalid_token' };
  }

  // 4. Both 模式：Local 或 Token
  if (authMode === 'both') {
    const localResult = checkLocalAuth(req);
    if (localResult.authenticated) {
      return localResult;
    }

    return authenticateToken(req, config);
  }

  return { authenticated: false, reason: 'invalid_mode' };
}
```

### 4. Canvas Host（Agent-UI 通信）

**文件**: `/home/runner/work/openclaw/openclaw/src/canvas-host/`

Canvas Host 是 Agent 和浏览器 UI 之间的双向通信桥梁。

#### 架构

```
Agent Loop
    ↓
Canvas Host Manager
    ↓ (WebSocket)
浏览器 Canvas UI
    ↓
用户交互（点击、输入、滚动）
    ↓
Canvas Host Manager
    ↓
Agent 工具调用
```

#### WebSocket 消息流

```typescript
// Canvas Host → Agent
{
  "type": "click",
  "elementId": "button-submit",
  "coordinates": { "x": 100, "y": 200 }
}

// Agent → Canvas Host
{
  "type": "navigate",
  "url": "https://example.com"
}

{
  "type": "evaluate",
  "script": "document.querySelector('#result').textContent"
}
```

---

## Chat Channels 架构

### 1. Channel Plugin 系统

#### 插件接口定义

**文件**: `/home/runner/work/openclaw/openclaw/src/channels/plugins/types.ts`

```typescript
// 第 50-120 行
export interface ChannelPlugin {
  // 通道标识
  channel: string;

  // 配置适配器
  config?: ChannelConfigAdapter;

  // 网关适配器（生命周期管理）
  gateway?: ChannelGatewayAdapter;

  // 出站适配器（发送消息）
  outbound?: ChannelOutboundAdapter;

  // 入站适配器（接收消息）
  inbound?: ChannelInboundAdapter;

  // 安全适配器（访问控制）
  security?: ChannelSecurityAdapter;

  // 目录适配器（用户/群组发现）
  directory?: ChannelDirectoryAdapter;

  // 动作适配器（reactions、polls 等）
  actions?: ChannelActionsAdapter;

  // 线程适配器（回复模式）
  threading?: ChannelThreadingAdapter;

  // 状态适配器（健康检查）
  status?: ChannelStatusAdapter;

  // 入职适配器（设置向导）
  onboarding?: ChannelOnboardingAdapter;

  // Agent 工具适配器
  agentTools?: ChannelAgentToolsAdapter;
}
```

#### 通道注册表

**文件**: `/home/runner/work/openclaw/openclaw/src/channels/registry.ts`

```typescript
// 第 20-85 行
export const CORE_CHANNELS: ChannelRegistryEntry[] = [
  {
    id: 'telegram',
    name: 'Telegram',
    description: 'Telegram Bot API',
    icon: '✈️',
    capabilities: ['dm', 'groups', 'media', 'polls', 'reactions']
  },
  {
    id: 'whatsapp',
    name: 'WhatsApp',
    description: 'WhatsApp Web (Baileys)',
    icon: '💬',
    capabilities: ['dm', 'groups', 'media', 'voice']
  },
  {
    id: 'discord',
    name: 'Discord',
    description: 'Discord Bot API',
    icon: '🎮',
    capabilities: ['dm', 'groups', 'media', 'polls', 'reactions', 'threads']
  },
  {
    id: 'slack',
    name: 'Slack',
    description: 'Slack Bot API',
    icon: '📨',
    capabilities: ['dm', 'channels', 'media', 'reactions', 'threads']
  },
  // ... 更多通道
];
```

#### 插件加载流程

```typescript
// 第 120-180 行
export async function loadGatewayPlugins(
  config: OpenClawConfig
): Promise<void> {
  const pluginRegistry = getGlobalPluginRegistry();

  // 1. 加载核心插件（内置）
  for (const entry of CORE_CHANNELS) {
    const plugin = await loadCoreChannelPlugin(entry.id);
    if (plugin) {
      pluginRegistry.registerChannel(entry.id, plugin);
    }
  }

  // 2. 加载扩展插件（extensions/）
  const extensionsDir = path.join(process.cwd(), 'extensions');
  const extensions = await fs.readdir(extensionsDir);

  for (const ext of extensions) {
    const extPath = path.join(extensionsDir, ext);
    const plugin = await loadExtensionPlugin(extPath);
    if (plugin) {
      pluginRegistry.registerChannel(plugin.channel, plugin);
    }
  }

  // 3. 加载配置的插件路径
  const pluginPaths = config.plugins?.paths ?? [];
  for (const pluginPath of pluginPaths) {
    const plugin = await loadPluginFromPath(pluginPath);
    if (plugin) {
      pluginRegistry.registerChannel(plugin.channel, plugin);
    }
  }

  // 4. 去重和排序
  pluginRegistry.deduplicate();
  pluginRegistry.sortByPriority();
}
```

### 2. Channel Manager（通道管理器）

**文件**: `/home/runner/work/openclaw/openclaw/src/gateway/server-channels.ts`

#### 管理器创建

```typescript
// 第 45-120 行
export function createChannelManager(options: {
  loadConfig: () => OpenClawConfig;
  channelLogs?: Map<string, string[]>;
  channelRuntimeEnvs?: Map<string, Record<string, string>>;
}): ChannelManager {

  // 状态跟踪
  const channelStates = new Map<string, ChannelRuntimeState>();
  const abortControllers = new Map<string, AbortController>();

  return {
    // 启动通道
    startChannel: async (channelId: string, accountId?: string) => {
      const key = `${channelId}:${accountId || 'default'}`;

      // 1. 检查是否已运行
      if (abortControllers.has(key)) {
        throw new Error(`Channel ${key} is already running`);
      }

      // 2. 加载配置
      const config = options.loadConfig();
      const channelConfig = resolveChannelConfig(config, channelId, accountId);

      // 3. 检查启用状态
      if (!channelConfig.enabled) {
        throw new Error(`Channel ${key} is disabled`);
      }

      // 4. 获取插件
      const plugin = getChannelPlugin(channelId);
      if (!plugin?.gateway?.startAccount) {
        throw new Error(`Channel ${channelId} has no gateway adapter`);
      }

      // 5. 创建 AbortController
      const abortController = new AbortController();
      abortControllers.set(key, abortController);

      // 6. 初始化状态
      const state: ChannelRuntimeState = {
        running: true,
        connected: false,
        startedAt: Date.now(),
        lastError: null,
        messageCount: 0
      };
      channelStates.set(key, state);

      // 7. 启动通道
      try {
        await plugin.gateway.startAccount({
          channelId,
          accountId: accountId || 'default',
          cfg: config,
          runtime: createPluginRuntime(),
          abortSignal: abortController.signal,
          setStatus: (update) => {
            Object.assign(state, update);
          }
        });
      } catch (error) {
        state.running = false;
        state.lastError = String(error);
        throw error;
      }
    },

    // 停止通道
    stopChannel: async (channelId: string, accountId?: string) => {
      const key = `${channelId}:${accountId || 'default'}`;

      // 1. 获取 AbortController
      const abortController = abortControllers.get(key);
      if (!abortController) {
        return; // 未运行
      }

      // 2. 触发中止
      abortController.abort();

      // 3. 等待清理
      await waitForChannelStop(key, 5000);

      // 4. 更新状态
      const state = channelStates.get(key);
      if (state) {
        state.running = false;
        state.connected = false;
        state.stoppedAt = Date.now();
      }

      // 5. 清理
      abortControllers.delete(key);
    },

    // 停止所有通道
    stopAll: async () => {
      const keys = Array.from(abortControllers.keys());
      await Promise.all(
        keys.map(key => {
          const [channelId, accountId] = key.split(':');
          return manager.stopChannel(channelId, accountId);
        })
      );
    },

    // 获取状态
    getStatus: (channelId: string, accountId?: string) => {
      const key = `${channelId}:${accountId || 'default'}`;
      return channelStates.get(key);
    },

    // 列出所有运行的通道
    listRunning: () => {
      return Array.from(channelStates.entries())
        .filter(([_, state]) => state.running)
        .map(([key, state]) => ({ key, state }));
    }
  };
}
```

#### 状态跟踪

```typescript
// 第 79-94 行
interface ChannelRuntimeState {
  // 运行状态
  running: boolean;
  connected: boolean;

  // 时间戳
  startedAt: number;
  stoppedAt?: number;
  lastConnectedAt?: number;
  lastDisconnectAt?: number;

  // 错误跟踪
  lastError: string | null;
  reconnectAttempts?: number;

  // 统计
  messageCount: number;
  lastMessageAt?: number;
  lastEventAt?: number;

  // 探测结果
  probeResult?: {
    authenticated: boolean;
    identity?: string;
    error?: string;
  };
}
```

### 3. WhatsApp 通道实现

#### 插件定义

**文件**: `/home/runner/work/openclaw/openclaw/extensions/whatsapp/src/channel.ts`

```typescript
// 第 380-520 行
export const whatsappChannelPlugin: ChannelPlugin = {
  channel: 'whatsapp',

  // 配置适配器
  config: {
    resolveAccountConfig: (cfg, accountId) => {
      // 解析账户配置
      const whatsappCfg = cfg.channels?.whatsapp;
      if (!whatsappCfg) return null;

      if (accountId && accountId !== 'default') {
        return whatsappCfg.accounts?.[accountId];
      }

      return whatsappCfg;
    },

    listAccounts: (cfg) => {
      // 列出所有账户
      const whatsappCfg = cfg.channels?.whatsapp;
      if (!whatsappCfg?.accounts) {
        return [{ id: 'default', enabled: true }];
      }

      return Object.entries(whatsappCfg.accounts).map(([id, config]) => ({
        id,
        enabled: config.enabled ?? true,
        name: config.name
      }));
    }
  },

  // 网关适配器
  gateway: {
    startAccount: async (ctx) => {
      const { channelId, accountId, cfg, runtime, abortSignal, setStatus } = ctx;

      // 1. 解析认证目录
      const authDir = resolveAuthDirForAccount(cfg, accountId);

      // 2. 读取自身身份
      const selfId = await readWebSelfId(authDir);

      // 3. 报告探测结果
      setStatus({
        probeResult: {
          authenticated: Boolean(selfId),
          identity: selfId
        }
      });

      // 4. 启动 Web 监听器
      return monitorWebChannel(
        false,  // verbose
        undefined,  // listenerFactory
        true,  // keepAlive
        undefined,  // replyResolver
        runtime,
        abortSignal,
        {
          statusSink: (next) => setStatus({ accountId, ...next }),
          accountId
        }
      );
    },

    stopAccount: async (ctx) => {
      // AbortController 已处理中止
      // 等待清理完成
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  },

  // 出站适配器
  outbound: {
    deliveryMode: 'gateway',

    chunker: chunkText,
    textChunkLimit: 4000,
    pollMaxOptions: 12,

    resolveTarget: {
      normalize: (to: string, options) => {
        // E.164 格式验证
        if (!to.startsWith('+')) {
          throw new Error('WhatsApp number must start with +');
        }
        return to;
      }
    },

    sendText: {
      async send(target, text, options) {
        return sendMessageWhatsApp(target, text, {
          accountId: options?.accountId,
          markdownToWhatsApp: true
        });
      }
    },

    sendMedia: {
      async send(target, text, media, options) {
        return sendMessageWhatsApp(target, text, {
          accountId: options?.accountId,
          mediaUrl: media.url,
          mediaType: media.type
        });
      }
    },

    sendPoll: {
      async send(target, poll, options) {
        return sendPollWhatsApp(target, poll, {
          accountId: options?.accountId
        });
      }
    }
  },

  // 安全适配器
  security: {
    checkAccess: async (msg, cfg) => {
      // 检查 DM 策略
      const whatsappCfg = resolveAccountConfig(cfg, msg.accountId);
      const dmPolicy = whatsappCfg?.dmPolicy ?? 'pairing';

      if (msg.chatType === 'direct') {
        if (dmPolicy === 'disabled') {
          return { allowed: false, reason: 'DM disabled' };
        }

        if (dmPolicy === 'allowlist') {
          const allowed = whatsappCfg?.allowFrom?.includes(msg.from);
          if (!allowed) {
            return { allowed: false, reason: 'Not in allowlist' };
          }
        }
      }

      return { allowed: true };
    },

    generatePairingCode: async (from, channel, accountId, cfg) => {
      // 生成 6 位配对码
      const code = Math.floor(100000 + Math.random() * 900000).toString();
      const expiresAt = Date.now() + 3600_000;  // 1 小时

      return { code, expiresAt };
    }
  },

  // 状态适配器
  status: {
    checkIssues: async (accountId, cfg) => {
      const authDir = resolveAuthDirForAccount(cfg, accountId);
      const hasCredsFile = await hasWebCredsSync(authDir);

      if (!hasCredsFile) {
        return [{
          severity: 'error',
          message: 'Not linked (no WhatsApp Web session)',
          suggestion: 'Run: openclaw channels login --channel whatsapp'
        }];
      }

      return [];
    }
  },

  // 入职适配器
  onboarding: {
    getStatus: async (accountId) => {
      const authDir = resolveDefaultWebAuthDir(accountId);
      const hasCredsFile = await hasWebCredsSync(authDir);

      if (!hasCredsFile) {
        return { linked: false, message: 'WhatsApp not connected' };
      }

      const selfId = await readWebSelfId(authDir);
      return {
        linked: true,
        message: `Connected: ${selfId}`,
        accountInfo: { phoneNumber: selfId }
      };
    },

    configure: async (options) => {
      // 交互式配置向导
      // ... (详见前面的 WhatsApp 分析)
    }
  }
};
```

#### 监听器实现

**文件**: `/home/runner/work/openclaw/openclaw/src/web/auto-reply/monitor.ts`

```typescript
// 第 100-280 行
export async function monitorWebChannel(
  verbose: boolean,
  listenerFactory?: WebListenerFactory,
  keepAlive?: boolean,
  replyResolver?: ReplyResolver,
  runtime?: WebChannelRuntime,
  abortSignal?: AbortSignal,
  tuning?: WebChannelTuning
): Promise<void> {

  const { cfg, accountId } = runtime;

  // 重连循环
  let reconnectAttempts = 0;
  const maxReconnectAttempts = 12;

  while (!abortSignal?.aborted) {
    try {
      log.info(
        `Starting WhatsApp monitor (account: ${accountId}, ` +
        `attempt: ${reconnectAttempts + 1}/${maxReconnectAttempts})`
      );

      // 创建入站监听器
      const listener = await monitorWebInbox({
        accountId,
        authDir: resolveAuthDirForAccount(cfg, accountId),
        mediaMaxMb: cfg.channels?.whatsapp?.mediaMaxMb ?? 50,
        sendReadReceipts: cfg.channels?.whatsapp?.sendReadReceipts ?? true,
        debounceMs: cfg.channels?.whatsapp?.debounceMs ?? 0,

        // 消息处理器
        onMessage: createWebOnMessageHandler({
          cfg,
          accountId,
          replyResolver,
          runtime
        }),

        // 连接事件
        onConnected: () => {
          log.info(`✓ WhatsApp connected (account: ${accountId})`);
          reconnectAttempts = 0;
          tuning?.statusSink?.({ connected: true, lastConnectedAt: Date.now() });
        },

        onDisconnected: (reason) => {
          log.warn(`WhatsApp disconnected (account: ${accountId}): ${reason}`);
          tuning?.statusSink?.({
            connected: false,
            lastDisconnectAt: Date.now(),
            lastDisconnect: { reason }
          });
        }
      });

      // 注册活动监听器
      setActiveWebListener(accountId, {
        sendMessage: listener.sendMessage,
        sendPoll: listener.sendPoll,
        sendReaction: listener.sendReaction
      });

      // 等待断开或中止
      await listener.waitForDisconnect();

      // 清理活动监听器
      clearActiveWebListener(accountId);

      // 检查是否应该重连
      if (abortSignal?.aborted) break;

      reconnectAttempts++;
      if (reconnectAttempts >= maxReconnectAttempts) {
        log.error(`Max reconnection attempts reached for account ${accountId}`);
        break;
      }

      // 指数退避
      const backoffMs = Math.min(2000 * Math.pow(2, reconnectAttempts - 1), 30000);
      log.info(`Reconnecting in ${(backoffMs / 1000).toFixed(1)}s`);
      await sleep(backoffMs);

    } catch (error) {
      log.error(`Fatal error in WhatsApp monitor: ${error}`);
      tuning?.statusSink?.({ lastError: String(error) });

      if (abortSignal?.aborted) break;
      await sleep(5000);
    }
  }

  log.info(`WhatsApp monitor stopped (account: ${accountId})`);
}
```

### 4. Telegram 通道实现

#### 插件定义（简化）

**文件**: `/home/runner/work/openclaw/openclaw/extensions/telegram/src/channel.ts`

```typescript
export const telegramChannelPlugin: ChannelPlugin = {
  channel: 'telegram',

  gateway: {
    startAccount: async (ctx) => {
      const token = resolveTelegramToken(ctx.cfg, ctx.accountId);

      // 探测 bot 身份
      const probeResult = await probeTelegramBotIdentity(token);
      ctx.setStatus({ probeResult });

      // 启动监听器
      return monitorTelegramProvider({
        token,
        accountId: ctx.accountId,
        config: ctx.cfg,
        runtime: ctx.runtime,
        abortSignal: ctx.abortSignal,
        useWebhook: Boolean(ctx.cfg.channels?.telegram?.webhookUrl)
      });
    }
  },

  outbound: {
    deliveryMode: 'direct',  // 直接调用 Bot API
    textChunkLimit: 4000,

    sendText: {
      async send(target, text, options) {
        return sendMessageTelegram(target, text, {
          accountId: options?.accountId,
          parseMode: 'Markdown'
        });
      }
    }
  }
};
```

#### 监听器实现（Long Polling）

**文件**: `/home/runner/work/openclaw/openclaw/src/telegram/bot.ts`

```typescript
// 第 200-350 行
export function createTelegramBot(options: {
  token: string;
  accountId: string;
  config: OpenClawConfig;
  onMessage: (msg: TelegramMessage) => Promise<void>;
  abortSignal?: AbortSignal;
}): Bot {

  // 1. 创建 grammY bot
  const bot = new Bot(options.token);

  // 2. 安装节流 middleware
  bot.use(throttler({
    global: new Bottleneck({ minTime: 35 }),  // 全局限制
    group: new Bottleneck({ minTime: 1000, maxConcurrent: 1 })  // 群组顺序处理
  }));

  // 3. 安装顺序处理 middleware
  bot.use(sequentialize((ctx) => {
    return getTelegramSequentialKey(ctx);
  }));

  // 4. 消息处理器
  bot.on('message', async (ctx) => {
    // 去重检查
    const updateId = ctx.update.update_id;
    if (seenUpdates.has(updateId)) {
      return;
    }
    seenUpdates.add(updateId);

    // 提取消息数据
    const msg = extractTelegramMessage(ctx);

    // 调用处理器
    await options.onMessage(msg);
  });

  // 5. 其他事件处理器
  bot.on('callback_query', handleCallbackQuery);
  bot.on('my_chat_member', handleChatMemberUpdate);

  // 6. 启动轮询
  bot.start({
    allowed_updates: ['message', 'callback_query', 'my_chat_member'],
    drop_pending_updates: true,
    onStart: () => {
      log.info(`Telegram bot started (account: ${options.accountId})`);
    }
  });

  // 7. 中止处理
  options.abortSignal?.addEventListener('abort', () => {
    bot.stop();
  });

  return bot;
}
```

---

## 消息路由机制

### 1. 入站消息流

```
消息到达通道 (WebSocket/HTTP/Polling)
    ↓
通道监听器 (monitorWebChannel, createTelegramBot, etc.)
    ↓
消息提取和规范化
    ├─ 发件人识别
    ├─ 聊天类型检测
    ├─ 正文解析
    ├─ 媒体提取
    └─ 提及解析
    ↓
去重检查 (LRU Cache / Set / Update ID)
    ↓
防抖处理 (合并连续消息)
    ↓
访问控制检查
    ├─ DM 策略验证
    ├─ 白名单检查
    ├─ 群组策略
    └─ 提及要求
    ↓
Agent 路由解析 (resolveAgentRoute)
    ├─ 根据发件人/群组/频道
    ├─ 角色路由 (Discord)
    └─ 返回 { agentId, sessionKey, tools }
    ↓
上下文构建
    ├─ 加载历史记录
    ├─ 构建会话上下文
    └─ 应用工具限制
    ↓
Agent 执行 (getReplyFromConfig)
    ├─ 运行 Agent Loop
    ├─ 工具调用
    └─ 生成回复
    ↓
出站队列 (delivery-queue)
    ↓
通道出站适配器
    ↓
消息发送到用户
```

### 2. 消息处理实现

**文件**: `/home/runner/work/openclaw/openclaw/src/web/auto-reply/monitor/process-message.ts`

```typescript
// 第 106-348 行
export async function processMessage(options: {
  msg: WebInboundMsg;
  route: AgentRoute;
  cfg: OpenClawConfig;
  accountId: string;
  replyResolver: ReplyResolver;
  dispatchReply: ReplyDispatcher;
}): Promise<void> {

  const { msg, route, cfg, accountId, replyResolver, dispatchReply } = options;

  // 1. 构建消息上下文
  const context = buildMessageContext({
    msg,
    route,
    cfg,
    accountId
  });

  // 2. 加载历史记录
  const history = await loadConversationHistory({
    sessionKey: route.sessionKey,
    limit: 20
  });

  // 3. 检测命令
  const isCommand = detectCommand(msg.text);

  // 4. 调用 Agent
  const replyResult = await replyResolver({
    prompt: msg.text,
    images: msg.media?.filter(m => m.type === 'image'),
    sessionKey: route.sessionKey,
    agentId: route.agentId,
    tools: route.tools,
    history,
    context: {
      channel: 'whatsapp',
      accountId,
      chatType: msg.chatType,
      from: msg.from,
      to: msg.to,
      messageId: msg.messageId,
      wasMentioned: msg.mentionedJids?.includes(selfJid)
    }
  });

  // 5. 处理回复
  if (replyResult.payloads && replyResult.payloads.length > 0) {
    await dispatchReply({
      payloads: replyResult.payloads,
      to: msg.from,
      chatType: msg.chatType,
      quotedMessageId: msg.messageId
    });
  }
}
```

### 3. 出站消息流

```
Agent 生成回复
    ↓
ReplyPayload 创建
    ├─ 文本内容
    ├─ 媒体附件
    ├─ 投票数据
    └─ 元数据
    ↓
出站队列 (持久化)
    ↓
deliverOutboundPayloads
    ├─ 解析目标
    ├─ 解析通道
    └─ 获取通道插件
    ↓
通道出站适配器
    ├─ resolveTarget (规范化目标)
    ├─ chunker (分块)
    └─ sendText/sendMedia/sendPoll
    ↓
通道特定 API
    ├─ WhatsApp: Baileys Socket
    ├─ Telegram: Bot API
    ├─ Discord: Gateway API
    └─ Slack: Web API
    ↓
消息发送确认
    ├─ messageId
    ├─ 时间戳
    └─ 发送状态
```

### 4. 出站实现

**文件**: `/home/runner/work/openclaw/openclaw/src/infra/outbound/deliver.ts`

```typescript
// 第 85-220 行
export async function deliverOutboundPayloads(
  payloads: ReplyPayload[],
  options: DeliveryOptions
): Promise<DeliveryResult[]> {

  const results: DeliveryResult[] = [];

  for (const payload of payloads) {
    try {
      // 1. 解析通道
      const channel = normalizeChannelId(options.channel);

      // 2. 获取插件
      const plugin = getChannelPlugin(channel);
      if (!plugin?.outbound) {
        throw new Error(`Channel ${channel} has no outbound adapter`);
      }

      // 3. 解析目标
      const target = plugin.outbound.resolveTarget?.normalize(
        options.to,
        { mode: 'strict', accountConfig: options.accountConfig }
      );

      // 4. 选择发送方法
      if (payload.poll) {
        // 发送投票
        const result = await plugin.outbound.sendPoll!.send(
          target,
          payload.poll,
          { accountId: options.accountId }
        );
        results.push({ success: true, messageId: result.messageId });
      }
      else if (payload.media && payload.media.length > 0) {
        // 发送媒体
        for (const media of payload.media) {
          const result = await plugin.outbound.sendMedia!.send(
            target,
            payload.text || '',
            media,
            { accountId: options.accountId }
          );
          results.push({ success: true, messageId: result.messageId });
        }
      }
      else {
        // 发送文本
        const chunks = plugin.outbound.chunker?.(
          payload.text,
          plugin.outbound.textChunkLimit
        ) ?? [payload.text];

        for (const chunk of chunks) {
          const result = await plugin.outbound.sendText!.send(
            target,
            chunk,
            { accountId: options.accountId }
          );
          results.push({ success: true, messageId: result.messageId });
        }
      }

    } catch (error) {
      results.push({
        success: false,
        error: String(error)
      });
    }
  }

  return results;
}
```

---

## 插件运行时系统

### Plugin Runtime 接口

**文件**: `/home/runner/work/openclaw/openclaw/src/plugins/runtime/index.ts`

```typescript
// 第 30-150 行
export interface PluginRuntime {
  // 通道接口
  channel: {
    whatsapp: {
      monitorWebChannel: typeof monitorWebChannel;
      sendMessageWhatsApp: typeof sendMessageWhatsApp;
      sendPollWhatsApp: typeof sendPollWhatsApp;
      sendReactionWhatsApp: typeof sendReactionWhatsApp;
      // ...
    };
    telegram: {
      monitorTelegramProvider: typeof monitorTelegramProvider;
      sendMessageTelegram: typeof sendMessageTelegram;
      // ...
    };
    discord: {
      monitorDiscordProvider: typeof monitorDiscordProvider;
      sendMessageDiscord: typeof sendMessageDiscord;
      // ...
    };
    slack: {
      monitorSlackProvider: typeof monitorSlackProvider;
      sendMessageSlack: typeof sendMessageSlack;
      // ...
    };
    // ... 其他通道
  };

  // 日志接口
  logging: {
    shouldLogVerbose: (channel: string) => boolean;
    getChannelLogger: (channel: string) => Logger;
  };

  // 自动回复接口
  'auto-reply': {
    dispatchReplyWithBufferedBlockDispatcher: typeof dispatchReplyWithBufferedBlockDispatcher;
    getReplyFromConfig: typeof getReplyFromConfig;
  };

  // 配置接口
  config: {
    loadConfig: () => Promise<OpenClawConfig>;
    reloadConfig: () => Promise<void>;
  };

  // 路由接口
  routing: {
    resolveAgentRoute: typeof resolveAgentRoute;
  };
}
```

### 运行时创建

```typescript
// 第 180-250 行
export function createPluginRuntime(): PluginRuntime {
  return {
    channel: {
      whatsapp: {
        monitorWebChannel: async (...args) => {
          const { monitorWebChannel } = await import('../../web/auto-reply/monitor.js');
          return monitorWebChannel(...args);
        },
        sendMessageWhatsApp: async (...args) => {
          const { sendMessageWhatsApp } = await import('../../web/outbound.js');
          return sendMessageWhatsApp(...args);
        },
        // ... 其他函数
      },
      telegram: {
        monitorTelegramProvider: async (...args) => {
          const { monitorTelegramProvider } = await import('../../telegram/monitor/provider.js');
          return monitorTelegramProvider(...args);
        },
        // ...
      },
      // ... 其他通道
    },

    logging: {
      shouldLogVerbose: (channel) => {
        // 检查环境变量或配置
        return process.env[`${channel.toUpperCase()}_VERBOSE`] === '1';
      },
      getChannelLogger: (channel) => {
        return createLogger({ subsystem: `channel/${channel}` });
      }
    },

    'auto-reply': {
      dispatchReplyWithBufferedBlockDispatcher: async (...args) => {
        const { dispatchReplyWithBufferedBlockDispatcher } =
          await import('../../auto-reply/reply/dispatch-buffered.js');
        return dispatchReplyWithBufferedBlockDispatcher(...args);
      },
      getReplyFromConfig: async (...args) => {
        const { getReplyFromConfig } = await import('../../auto-reply/index.js');
        return getReplyFromConfig(...args);
      }
    },

    config: {
      loadConfig: async () => {
        const { loadConfig } = await import('../../config/index.js');
        return loadConfig();
      },
      reloadConfig: async () => {
        const { reloadConfig } = await import('../../config/reload.js');
        return reloadConfig();
      }
    },

    routing: {
      resolveAgentRoute: (params) => {
        const { resolveAgentRoute } = require('../../routing/resolve-route.js');
        return resolveAgentRoute(params);
      }
    }
  };
}
```

---

## 配置和管理

### 1. 配置结构

```typescript
// openclaw.json
{
  "gateway": {
    "port": 18789,
    "bind": "loopback",  // "loopback" | "all" | "tailscale"

    "hooks": {
      "auth": {
        "mode": "token",
        "tokens": ["secret-token-1"]
      },
      "basePath": "webhooks",
      "mappings": [/* ... */]
    }
  },

  "channels": {
    "whatsapp": {
      "enabled": true,
      "dmPolicy": "allowlist",
      "allowFrom": ["+15551234567"],
      "accounts": {
        "work": { /* ... */ },
        "personal": { /* ... */ }
      }
    },

    "telegram": {
      "enabled": true,
      "botToken": "123456:ABC...",
      "dmPolicy": "open"
    },

    "discord": {
      "enabled": true,
      "token": "MTk4NjIy...",
      "dm": { "enabled": true }
    }
  },

  "agents": {
    "defaults": {
      "agentId": "default",
      "model": {
        "provider": "anthropic",
        "model": "claude-sonnet-4-5"
      }
    }
  }
}
```

### 2. 配置重载

**文件**: `/home/runner/work/openclaw/openclaw/src/gateway/config-reload.ts`

```typescript
// 第 40-120 行
export async function reloadGatewayConfig(
  channelManager: ChannelManager
): Promise<void> {

  // 1. 重新加载配置文件
  const newConfig = await loadConfig();

  // 2. 对比变化
  const changes = detectConfigChanges(currentConfig, newConfig);

  // 3. 应用通道变化
  for (const change of changes.channels) {
    if (change.type === 'added' || change.type === 'enabled') {
      // 启动新通道
      await channelManager.startChannel(change.channelId, change.accountId);
    }
    else if (change.type === 'removed' || change.type === 'disabled') {
      // 停止通道
      await channelManager.stopChannel(change.channelId, change.accountId);
    }
    else if (change.type === 'modified') {
      // 重启通道
      await channelManager.stopChannel(change.channelId, change.accountId);
      await channelManager.startChannel(change.channelId, change.accountId);
    }
  }

  // 4. 更新全局配置
  currentConfig = newConfig;

  // 5. 通知所有订阅者
  configChangeEmitter.emit('reload', newConfig);
}
```

### 3. 健康检查

**文件**: `/home/runner/work/openclaw/openclaw/src/gateway/server-health.ts`

```typescript
// 第 25-90 行
export async function performHealthCheck(
  channelManager: ChannelManager
): Promise<HealthCheckResult> {

  const result: HealthCheckResult = {
    healthy: true,
    timestamp: Date.now(),
    checks: []
  };

  // 1. 检查所有运行的通道
  const runningChannels = channelManager.listRunning();

  for (const { key, state } of runningChannels) {
    const [channelId, accountId] = key.split(':');

    const check: HealthCheck = {
      name: `channel:${key}`,
      healthy: state.connected && !state.lastError,
      details: {
        running: state.running,
        connected: state.connected,
        lastError: state.lastError,
        messageCount: state.messageCount,
        uptime: Date.now() - state.startedAt
      }
    };

    if (!check.healthy) {
      result.healthy = false;
    }

    result.checks.push(check);
  }

  // 2. 检查 HTTP 服务器
  result.checks.push({
    name: 'http-server',
    healthy: true,
    details: { listening: true }
  });

  // 3. 检查 WebSocket 服务器
  result.checks.push({
    name: 'websocket-server',
    healthy: true,
    details: { listening: true }
  });

  return result;
}
```

---

## 架构模式总结

### 1. 插件架构（Plugin Architecture）

**优势**：
- ✅ **解耦**：核心系统与通道实现分离
- ✅ **扩展性**：新通道作为插件添加
- ✅ **热加载**：运行时加载/卸载插件
- ✅ **版本隔离**：每个插件独立版本

**实现**：
```typescript
// 插件注册表
PluginRegistry {
  channels: Map<string, ChannelPlugin>

  register(channelId, plugin) → void
  get(channelId) → ChannelPlugin
  list() → ChannelPlugin[]
}
```

### 2. 适配器模式（Adapter Pattern）

**统一接口**：
```typescript
interface ChannelOutboundAdapter {
  sendText(target, text, options) → Promise<Result>
  sendMedia(target, media, options) → Promise<Result>
  sendPoll(target, poll, options) → Promise<Result>
}
```

**平台特定实现**：
- WhatsApp: Baileys WebSocket
- Telegram: Bot API HTTP
- Discord: Gateway WebSocket
- Slack: Web API HTTP

### 3. 事件驱动（Event-Driven）

**事件类型**：
- `message` - 新消息到达
- `reaction` - 反应添加/移除
- `member_update` - 成员加入/离开
- `connection_update` - 连接状态变化
- `error` - 错误事件

**事件流**：
```
通道监听器
    ↓ (emit)
事件处理器
    ↓ (process)
消息路由器
    ↓ (dispatch)
Agent 执行
```

### 4. 状态管理（State Management）

**运行时状态**：
- 每个通道/账户的状态快照
- 连接状态、错误跟踪
- 消息统计、性能指标

**持久化状态**：
- 认证凭证
- 会话历史
- 配置数据
- 出站队列

### 5. 生命周期管理（Lifecycle Management）

**启动阶段**：
```
Load Config → Load Plugins → Create Manager → Start Channels
```

**运行阶段**：
```
Monitor Events → Route Messages → Execute Agents → Send Replies
```

**停止阶段**：
```
Abort Signal → Stop Monitors → Cleanup Resources → Save State
```

### 6. 安全模型（Security Model）

**多层认证**：
1. Gateway HTTP: Token / Tailscale / Local IP
2. Channel: DM Policy / Allowlist / Pairing
3. Agent: Tool Restrictions / Command Gating

**速率限制**：
- HTTP 端点：20 失败/60 秒
- Channel API：平台特定限制
- Agent 执行：并发限制

### 7. 可观测性（Observability）

**日志系统**：
- 结构化日志（JSON）
- 子系统标签（gateway, channel/whatsapp, agent）
- 日志级别（debug, info, warn, error）

**指标收集**：
- 消息计数
- 响应时间
- 错误率
- 连接状态

**健康检查**：
- `/health` HTTP 端点
- 通道状态报告
- 自动探测

---

## 总结

OpenClaw 的 Gateway 和 Chat Channels 架构展示了优秀的系统设计：

### 核心优势

1. **统一网关**：单一入口管理所有通道
2. **插件系统**：灵活的扩展机制
3. **适配器模式**：统一的消息接口
4. **生命周期管理**：健壮的启动/停止/重连
5. **配置热重载**：无需重启更新配置
6. **多账户支持**：每个通道多个账户
7. **安全可靠**：多层认证和访问控制
8. **可观测性**：完整的日志和监控

### 技术亮点

- **TypeScript + ESM**：现代 JavaScript 生态
- **异步架构**：Promise/async-await
- **事件驱动**：EventEmitter 模式
- **WebSocket + HTTP**：双向通信
- **插件注册表**：动态加载和管理
- **AbortController**：优雅的取消机制
- **指数退避重连**：健壮的错误恢复

### 文件结构总结

```
src/
├── gateway/
│   ├── server.impl.ts          # Gateway 主入口
│   ├── server-http.ts          # HTTP/WebSocket 服务器
│   ├── server-channels.ts      # Channel Manager
│   ├── server-http-hooks.ts    # Hooks API
│   └── config-reload.ts        # 配置热重载
├── channels/
│   ├── plugins/                # 插件接口定义
│   ├── registry.ts             # 通道注册表
│   └── dock.ts                 # 通道元数据
├── web/                        # WhatsApp 实现
├── telegram/                   # Telegram 实现
├── discord/                    # Discord 实现
├── slack/                      # Slack 实现
└── plugins/
    └── runtime/                # 插件运行时
```

---

**文档版本**: 1.0
**最后更新**: 2026-02-19
**分析范围**: OpenClaw Gateway 和 Chat Channels 完整架构
