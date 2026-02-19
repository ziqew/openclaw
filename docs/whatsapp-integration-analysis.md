# WhatsApp 接入 OpenClaw 源代码分析

> 本文档详细分析了 WhatsApp 如何接入 OpenClaw 系统的完整架构和实现

## 目录

1. [概述](#概述)
2. [整体架构](#整体架构)
3. [认证与会话管理](#认证与会话管理)
4. [消息接收流程](#消息接收流程)
5. [消息发送流程](#消息发送流程)
6. [配置与设置](#配置与设置)
7. [与 Agent Loop 集成](#与-agent-loop-集成)
8. [核心组件详解](#核心组件详解)
9. [完整数据流图](#完整数据流图)
10. [安全性与最佳实践](#安全性与最佳实践)

---

## 概述

WhatsApp 在 OpenClaw 中以 **Web 通道** 的形式实现，使用 **Baileys 库**（WhatsApp Web 自动化库）进行连接。这种实现方式类似于 WhatsApp Web 客户端，通过 WebSocket 与 WhatsApp 服务器通信。

### 关键特性

- ✅ **QR 码登录**：类似 WhatsApp Web 的配对方式
- ✅ **多账户支持**：可同时运行多个 WhatsApp 账户
- ✅ **完整媒体支持**：图片、视频、音频、文档、语音消息
- ✅ **群组功能**：支持群聊、提及、回复
- ✅ **访问控制**：DM 策略、白名单、群组策略
- ✅ **自动重连**：带指数退避的连接恢复
- ✅ **流式消息**：支持实时消息流和块回复

## 整体架构

### 高层连接架构

```
WhatsApp 服务器
    ↕ (WebSocket 连接)
Baileys 库 (src/web/session.ts)
    ↓
Web 会话管理器
    ↓
入站消息监听器 (src/web/inbound/monitor.ts)
    ↓
访问控制 & 防抖 (src/web/inbound/access-control.ts)
    ↓
消息处理器 (src/web/auto-reply/monitor.ts)
    ↓
Agent 路由解析 → Agent Loop 执行
    ↓
出站适配器 (src/channels/plugins/outbound/whatsapp.ts)
    ↓
通过 WebSocket 发送消息
```

### 核心文件结构

| 文件路径 | 功能 | 行数估计 |
|---------|------|----------|
| `src/web/session.ts` | Socket 创建和连接管理 | ~800 |
| `src/web/inbound/monitor.ts` | 消息接收管道 | ~600 |
| `src/web/auto-reply/monitor.ts` | 网关主循环集成 | ~500 |
| `src/web/outbound.ts` | 消息发送实现 | ~400 |
| `src/channels/plugins/outbound/whatsapp.ts` | 出站适配器 | ~300 |
| `src/web/auth-store.ts` | 认证存储和恢复 | ~300 |
| `src/whatsapp/normalize.ts` | 目标规范化（E.164 ↔ JID） | ~200 |
| `src/markdown/whatsapp.ts` | Markdown 转 WhatsApp 格式 | ~150 |

## 认证与会话管理

### QR 码登录流程

```typescript
// src/web/session.ts
export async function createWaSocket(options: {
  authDir: string;
  printQr?: boolean;
  onQr?: (qr: string) => void;
  // ...
}) {
  // 1. 初始化 Baileys socket
  const { state, saveCreds } = await useMultiFileAuthState(authDir);

  // 2. 创建 WebSocket 连接
  const sock = makeWASocket({
    auth: state,
    printQRInTerminal: printQr,
    // ... Baileys 配置
  });

  // 3. 监听连接事件
  sock.ev.on("connection.update", (update) => {
    const { connection, qr } = update;

    if (qr && onQr) {
      onQr(qr); // 显示 QR 码
    }

    if (connection === "open") {
      console.log("✓ WhatsApp 连接成功");
    }
  });

  // 4. 自动保存认证状态
  sock.ev.on("creds.update", saveCreds);

  return sock;
}
```

**登录步骤**：

1. **用户执行命令**：`openclaw channels login --channel whatsapp`
2. **生成 QR 码**：终端显示二维码
3. **用户扫描**：使用 WhatsApp 的"关联设备"功能扫描
4. **建立连接**：Baileys 捕获连接并保存凭证
5. **凭证持久化**：多文件认证状态保存到磁盘

### 凭证存储

```typescript
// src/web/auth-store.ts

// 默认路径结构
export function resolveDefaultWebAuthDir(accountId?: string): string {
  const baseDir = path.join(os.homedir(), ".openclaw", "oauth", "whatsapp");

  if (accountId && accountId !== DEFAULT_ACCOUNT_ID) {
    return path.join(baseDir, accountId);
  }

  return path.join(baseDir, "default");
}

// 凭证文件结构
// ~/.openclaw/oauth/whatsapp/{accountId}/
//   ├── creds.json          # 主凭证文件
//   ├── creds.json.bak      # 备份文件
//   ├── app-state-sync-key-*.json
//   ├── app-state-sync-version-*.json
//   └── ... (其他 Baileys 状态文件)
```

**安全特性**：

- **文件权限**：`0o600`（仅所有者读写）
- **自动备份**：每次保存时创建 `.bak` 备份
- **崩溃恢复**：从备份自动恢复损坏的凭证
- **迁移支持**：自动从旧路径迁移到新路径

### 多账户支持

```typescript
// src/config/types.whatsapp.ts
type WhatsAppConfig = {
  accounts?: Record<string, WhatsAppAccountConfig>;
  // ...
};

type WhatsAppAccountConfig = WhatsAppConfig & {
  name?: string;           // 账户显示名称
  enabled?: boolean;       // 是否启用（默认 true）
  authDir?: string;        // 自定义认证目录
  // ... 继承所有 WhatsApp 配置选项
};
```

**配置示例**：

```json5
{
  "channels": {
    "whatsapp": {
      "accounts": {
        "work": {
          "name": "工作号",
          "enabled": true,
          "authDir": "/custom/path/work",
          "dmPolicy": "allowlist",
          "allowFrom": ["+15551234567"]
        },
        "personal": {
          "name": "个人号",
          "enabled": true,
          "dmPolicy": "open"
        }
      }
    }
  }
}
```

**实现机制**：

```typescript
// src/web/active-listener.ts
const ACTIVE_WEB_LISTENERS = new Map<string, ActiveWebListener>();

export function setActiveWebListener(
  accountId: string,
  listener: ActiveWebListener
) {
  const key = accountId || DEFAULT_ACCOUNT_ID;
  ACTIVE_WEB_LISTENERS.set(key, listener);
}

export function requireActiveWebListener(
  accountId?: string
): ActiveWebListener {
  const key = accountId || DEFAULT_ACCOUNT_ID;
  const listener = ACTIVE_WEB_LISTENERS.get(key);

  if (!listener) {
    throw new Error(
      `WhatsApp account "${key}" is not active. ` +
      `Run: openclaw channels login --channel whatsapp`
    );
  }

  return listener;
}
```

## 消息接收流程

### 入站管道详解

```
WhatsApp 消息到达
    ↓
┌─────────────────────────────────────────────────────────┐
│ 1. Socket 事件捕获 (monitorWebInbox)                   │
│    - 监听 messages.upsert 事件                          │
│    - 提取消息元数据 (JID, body, media, etc.)            │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ 2. 消息提取与规范化 (src/web/inbound/extract.ts)       │
│    - JID → E.164: "41796666864:0@s.whatsapp.net"       │
│               → "+41796666864"                          │
│    - 检测聊天类型: "direct" 或 "group"                 │
│    - 解析正文、媒体、提及、引用                         │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ 3. 访问控制 (src/web/inbound/access-control.ts)        │
│    - DM 策略检查: pairing → allowlist → open           │
│    - 群组策略检查: 群组白名单 + 提及门控               │
│    - 配对请求生成（未知发件人）                         │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ 4. 防抖处理 (src/auto-reply/inbound-debounce.ts)       │
│    - 可配置防抖窗口（默认 0ms）                         │
│    - 批处理来自同一发件人的连续快速消息                │
│    - 媒体/位置/回复/控制命令跳过防抖                    │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ 5. 会话路由 (resolveAgentRoute)                        │
│    - channel: "whatsapp"                                │
│    - accountId: 账户 ID                                 │
│    - peer: { kind: chatType, id: peerId }              │
│    → 解析到特定 Agent 绑定                              │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ 6. 消息处理 (processMessage)                           │
│    - Echo 检测（跳过机器人自己的消息）                  │
│    - 群组历史跟踪                                       │
│    - 提及解析                                           │
│    - 分发到 Agent Loop: getReplyFromConfig()            │
└─────────────────────────────────────────────────────────┘
```

### 关键代码：消息监听器

```typescript
// src/web/inbound/monitor.ts
export function monitorWebInbox(options: {
  accountId: string;
  authDir: string;
  mediaMaxMb?: number;
  sendReadReceipts?: boolean;
  debounceMs?: number;
  onMessage: (msg: WebInboundMsg) => void | Promise<void>;
  // ...
}): Promise<WebInboxListener> {
  const { accountId, onMessage } = options;

  // 创建 WebSocket 连接
  const sock = await createWaSocket({
    authDir: options.authDir,
    // ...
  });

  // 消息去重（使用 LRU 缓存）
  const dedupe = new LRUCache<string, boolean>({ max: 1000 });

  // 监听消息更新事件
  sock.ev.on("messages.upsert", async ({ messages, type }) => {
    for (const raw of messages) {
      // 去重检查
      const dedupeKey = `${accountId}:${raw.key.remoteJid}:${raw.key.id}`;
      if (dedupe.has(dedupeKey)) continue;
      dedupe.set(dedupeKey, true);

      // 提取消息数据
      const msg = extractWebInboundMessage(raw, accountId, sock);

      // 访问控制检查
      const accessResult = checkWebAccess(msg, options.config);
      if (!accessResult.allowed) {
        // 生成配对码或拒绝
        continue;
      }

      // 发送已读回执（如果启用）
      if (options.sendReadReceipts && msg.chatType === "direct") {
        await sock.readMessages([raw.key]);
      }

      // 调用消息处理器
      await onMessage(msg);
    }
  });

  return {
    sock,
    sendMessage: createSendMessageFn(sock),
    sendPoll: createSendPollFn(sock),
    sendReaction: createSendReactionFn(sock),
    // ...
  };
}
```

### 访问控制策略

```typescript
// src/web/inbound/access-control.ts
export function checkWebAccess(
  msg: WebInboundMsg,
  cfg?: OpencClawConfig
): AccessCheckResult {
  const whatsappCfg = resolveAccountConfig(cfg, msg.accountId);

  // 1. 检查 DM 策略
  if (msg.chatType === "direct") {
    const dmPolicy = whatsappCfg.dmPolicy ?? "pairing";

    switch (dmPolicy) {
      case "disabled":
        return { allowed: false, reason: "DM disabled" };

      case "pairing":
        // 检查是否在配对列表或白名单中
        if (!isPaired(msg.from, cfg) && !isInAllowlist(msg.from, whatsappCfg)) {
          // 生成配对码
          return {
            allowed: false,
            reason: "Not paired",
            shouldGeneratePairingCode: true
          };
        }
        break;

      case "allowlist":
        if (!isInAllowlist(msg.from, whatsappCfg)) {
          return { allowed: false, reason: "Not in allowlist" };
        }
        break;

      case "open":
        // 允许所有 DM
        break;
    }
  }

  // 2. 检查群组策略
  if (msg.chatType === "group") {
    const groupPolicy = whatsappCfg.groupPolicy ?? "open";

    if (groupPolicy === "disabled") {
      return { allowed: false, reason: "Groups disabled" };
    }

    if (groupPolicy === "allowlist") {
      if (!isGroupInAllowlist(msg.chatJid, whatsappCfg)) {
        return { allowed: false, reason: "Group not in allowlist" };
      }
    }

    // 检查提及要求
    const groupCfg = whatsappCfg.groups?.[msg.chatJid];
    if (groupCfg?.requireMention && !msg.mentionedJids?.includes(selfJid)) {
      return { allowed: false, reason: "Mention required" };
    }
  }

  return { allowed: true };
}
```

### 消息提取

```typescript
// src/web/inbound/extract.ts
export function extractWebInboundMessage(
  raw: proto.IWebMessageInfo,
  accountId: string,
  sock: WASocket
): WebInboundMsg {
  const key = raw.key;
  const message = raw.message;

  // 提取文本
  const text = extractText(message);

  // 提取媒体占位符
  const media = extractMediaPlaceholder(message);

  // 提取位置数据
  const location = extractLocationData(message);

  // 提取提及的 JID
  const mentionedJids = extractMentionedJids(message);

  // 提取引用的消息
  const quotedMsg = message?.extendedTextMessage?.contextInfo?.quotedMessage;

  // 判断聊天类型
  const chatType = isWhatsAppGroupJid(key.remoteJid)
    ? "group"
    : "direct";

  // 规范化发件人到 E.164
  const from = normalizeWhatsAppJidToE164(
    chatType === "group" ? key.participant : key.remoteJid
  );

  return {
    accountId,
    messageId: key.id,
    chatJid: key.remoteJid,
    chatType,
    from,
    fromJid: key.participant || key.remoteJid,
    text,
    media,
    location,
    mentionedJids,
    quotedMsg,
    timestamp: raw.messageTimestamp,
    // ...
  };
}
```

## 消息发送流程

### 出站管道详解

```
Agent 生成回复
    ↓
┌─────────────────────────────────────────────────────────┐
│ 1. dispatchReply (auto-reply/reply/dispatch-from-config)│
│    - 准备消息内容                                       │
│    - 选择通道适配器                                     │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ 2. ChannelOutboundAdapter (whatsapp.ts)                 │
│    - resolveTarget: 规范化接收者                        │
│    - sendText/sendMedia/sendPoll: 选择发送方法          │
│    - 分块处理（textChunkLimit: 4000 字符）             │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ 3. sendMessageWhatsApp (src/web/outbound.ts)            │
│    - Markdown → WhatsApp 格式转换                       │
│    - 加载媒体（如果提供 mediaUrl）                      │
│    - 获取活动监听器: requireActiveWebListener()         │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ 4. ActiveWebListener.sendMessage                        │
│    - 通过 Baileys socket 发送                           │
│    - 返回 messageId                                     │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ 5. 后处理（可选）                                       │
│    - 发送已读回执                                       │
│    - 发送确认反应（emoji）                              │
└─────────────────────────────────────────────────────────┘
```

### 关键代码：消息发送

```typescript
// src/web/outbound.ts
export async function sendMessageWhatsApp(
  to: string,                    // E.164 格式: "+15551234567"
  text: string,                  // 消息正文
  options?: {
    accountId?: string;
    mediaUrl?: string;           // 可选：媒体 URL
    mediaType?: "image" | "video" | "audio" | "document";
    markdownToWhatsApp?: boolean; // 默认 true
    // ...
  }
): Promise<{ messageId: string }> {
  const accountId = options?.accountId || DEFAULT_ACCOUNT_ID;

  // 1. 获取活动的 Web 监听器
  const active = requireActiveWebListener(accountId);

  // 2. 转换 E.164 → JID
  const toJid = normalizeWhatsAppTargetToJid(to);

  // 3. Markdown 转换（如果启用）
  let finalText = text;
  if (options?.markdownToWhatsApp !== false) {
    finalText = convertMarkdownToWhatsApp(text);
  }

  // 4. 加载媒体（如果提供）
  let mediaBuffer: Buffer | undefined;
  if (options?.mediaUrl) {
    mediaBuffer = await loadMediaFromUrl(options.mediaUrl);
  }

  // 5. 发送消息
  const result = await active.sendMessage(
    toJid,
    finalText,
    mediaBuffer,
    options?.mediaType,
    {
      // Baileys 发送选项
      quoted: options?.quotedMessageId,
      // ...
    }
  );

  return {
    messageId: result.key.id!
  };
}
```

### Markdown 格式转换

```typescript
// src/markdown/whatsapp.ts
export function convertMarkdownToWhatsApp(text: string): string {
  // WhatsApp 原生格式：
  // *粗体* (星号)
  // _斜体_ (下划线)
  // ~删除线~ (波浪号)
  // ```代码``` (三个反引号)

  // 转换规则：
  // **粗体** → *粗体*
  // ~~删除线~~ → ~删除线~
  // _斜体_ → _斜体_ (保持不变)

  let result = text;

  // 1. 保护代码块（内联和围栏）
  const codeBlocks: string[] = [];
  result = result.replace(
    /(`{1,3})([\s\S]*?)\1/g,
    (match, delim, content) => {
      const idx = codeBlocks.length;
      codeBlocks.push(match);
      return `__CODE_BLOCK_${idx}__`;
    }
  );

  // 2. 转换 Markdown → WhatsApp
  result = result
    // **粗体** → *粗体*
    .replace(/\*\*(.+?)\*\*/g, "*$1*")
    // ~~删除线~~ → ~删除线~
    .replace(/~~(.+?)~~/g, "~$1~");

  // 3. 恢复代码块
  result = result.replace(
    /__CODE_BLOCK_(\d+)__/g,
    (_, idx) => codeBlocks[parseInt(idx)]
  );

  return result;
}
```

### 出站适配器

```typescript
// src/channels/plugins/outbound/whatsapp.ts
export const whatsappOutbound: ChannelOutboundAdapter = {
  deliveryMode: "gateway",        // 使用网关模式

  chunker: chunkText,             // 文本分块函数
  textChunkLimit: 4000,           // 单条消息最大字符数
  pollMaxOptions: 12,             // 投票最多选项数

  // 目标规范化
  resolveTarget: {
    normalize: (to: string, options?: ResolveTargetOptions) => {
      // E.164 格式验证
      if (!to.startsWith("+")) {
        throw new Error("WhatsApp number must start with +");
      }

      // 检查白名单（如果启用）
      const allowFrom = options?.accountConfig?.allowFrom;
      if (options?.mode === "strict" && allowFrom) {
        if (!allowFrom.includes(to)) {
          throw new Error(`Number ${to} not in allowlist`);
        }
      }

      return to; // 返回 E.164 格式
    }
  },

  // 发送文本消息
  sendText: {
    async send(target, text, options) {
      return sendMessageWhatsApp(target, text, {
        accountId: options?.accountId,
        markdownToWhatsApp: true,
      });
    }
  },

  // 发送媒体消息
  sendMedia: {
    async send(target, text, media, options) {
      return sendMessageWhatsApp(target, text, {
        accountId: options?.accountId,
        mediaUrl: media.url,
        mediaType: media.type,
        markdownToWhatsApp: true,
      });
    }
  },

  // 发送投票
  sendPoll: {
    async send(target, poll, options) {
      return sendPollWhatsApp(target, poll, {
        accountId: options?.accountId,
      });
    }
  },
};
```

### 媒体处理

```typescript
// src/web/media.ts
export async function loadMediaFromUrl(
  url: string,
  options?: {
    maxMb?: number;        // 默认 50MB
    timeout?: number;      // 默认 30s
  }
): Promise<Buffer> {
  const maxBytes = (options?.maxMb ?? 50) * 1024 * 1024;

  // 1. 下载媒体
  const response = await fetch(url, {
    timeout: options?.timeout ?? 30_000,
  });

  if (!response.ok) {
    throw new Error(`Failed to download media: ${response.statusText}`);
  }

  // 2. 读取为 Buffer
  const buffer = Buffer.from(await response.arrayBuffer());

  // 3. 大小检查
  if (buffer.length > maxBytes) {
    throw new Error(
      `Media too large: ${(buffer.length / 1024 / 1024).toFixed(2)}MB ` +
      `(max: ${options?.maxMb ?? 50}MB)`
    );
  }

  return buffer;
}

// 语音消息特殊处理
export async function sendVoiceNote(
  to: string,
  audioBuffer: Buffer,
  options?: {
    accountId?: string;
  }
): Promise<{ messageId: string }> {
  const active = requireActiveWebListener(options?.accountId);

  // WhatsApp 语音消息必须是 Opus 编码的 OGG
  const result = await active.sendMessage(
    normalizeWhatsAppTargetToJid(to),
    "",
    audioBuffer,
    "audio",
    {
      ptt: true,  // Push-to-talk（语音消息标志）
      mimetype: "audio/ogg; codecs=opus",
    }
  );

  return { messageId: result.key.id! };
}
```

## 配置与设置

### 配置架构

```typescript
// src/config/types.whatsapp.ts
export const whatsappConfigSchema = z.object({
  // 多账户配置
  accounts: z.record(
    z.string(),
    z.lazy(() => whatsappConfigSchema.extend({
      name: z.string().optional(),
      enabled: z.boolean().optional(),
      authDir: z.string().optional(),
    }))
  ).optional(),

  // DM 策略
  dmPolicy: z.enum([
    "pairing",      // 需要配对码（默认）
    "allowlist",    // 仅白名单
    "open",         // 开放所有 DM
    "disabled"      // 禁用 DM
  ]).optional(),

  // 白名单（E.164 格式）
  allowFrom: z.array(z.string()).optional(),

  // 群组策略
  groupPolicy: z.enum([
    "open",         // 开放（默认）
    "allowlist",    // 仅白名单群组
    "disabled"      // 禁用群组
  ]).optional(),

  // 群组白名单（JID 格式）
  groupAllowFrom: z.array(z.string()).optional(),

  // 已读回执
  sendReadReceipts: z.boolean().optional(),

  // 消息前缀
  messagePrefix: z.string().optional(),     // 入站前缀
  responsePrefix: z.string().optional(),    // 出站前缀

  // 个人号码模式
  selfChatMode: z.boolean().optional(),

  // 文本分块限制
  textChunkLimit: z.number().optional(),    // 默认 4000

  // 媒体大小限制
  mediaMaxMb: z.number().optional(),        // 默认 50

  // 防抖配置
  debounceMs: z.number().optional(),        // 默认 0

  // 确认反应
  ackReaction: z.object({
    emoji: z.string(),
    direct: z.boolean().optional(),
    group: z.enum(["always", "mentions", "never"]).optional(),
  }).optional(),

  // 群组特定配置
  groups: z.record(
    z.string(),  // 群组 JID
    z.object({
      requireMention: z.boolean().optional(),
      tools: z.array(z.string()).optional(),
      toolsBySender: z.record(
        z.string(),  // E.164
        z.array(z.string())
      ).optional(),
    })
  ).optional(),

  // 流式消息配置
  blockStreaming: z.boolean().optional(),
  blockStreamingCoalesce: z.object({
    debounceMs: z.number().optional(),
    maxChunks: z.number().optional(),
  }).optional(),
});
```

### 配置示例

#### 1. 专用号码（推荐配置）

```json5
{
  "channels": {
    "whatsapp": {
      "dmPolicy": "allowlist",
      "allowFrom": [
        "+15551234567",    // 白名单用户
        "+15559876543"
      ],
      "groupPolicy": "allowlist",
      "groupAllowFrom": [
        "120363123456789012@g.us"  // 白名单群组
      ],
      "sendReadReceipts": true,
      "mediaMaxMb": 50,
      "ackReaction": {
        "emoji": "👍",
        "direct": true,
        "group": "mentions"
      }
    }
  }
}
```

#### 2. 个人号码配置

```json5
{
  "channels": {
    "whatsapp": {
      "dmPolicy": "allowlist",
      "allowFrom": ["+15551234567"],
      "selfChatMode": true,        // 启用个人号码模式
      "sendReadReceipts": false,   // 关闭已读回执
      "groupPolicy": "disabled"    // 禁用群组
    }
  }
}
```

#### 3. 多账户配置

```json5
{
  "channels": {
    "whatsapp": {
      "accounts": {
        "work": {
          "name": "工作账户",
          "enabled": true,
          "dmPolicy": "allowlist",
          "allowFrom": ["+15551234567"],
          "groupPolicy": "open",
          "sendReadReceipts": true
        },
        "personal": {
          "name": "个人账户",
          "enabled": true,
          "authDir": "/custom/path",
          "dmPolicy": "open",
          "groupPolicy": "disabled"
        },
        "dev": {
          "name": "开发测试",
          "enabled": false  // 暂时禁用
        }
      }
    }
  }
}
```

#### 4. 群组细粒度配置

```json5
{
  "channels": {
    "whatsapp": {
      "groupPolicy": "allowlist",
      "groupAllowFrom": [
        "120363123456789012@g.us",
        "120363987654321098@g.us"
      ],
      "groups": {
        "120363123456789012@g.us": {
          "requireMention": true,     // 必须提及才响应
          "tools": [                  // 限制可用工具
            "bash",
            "read",
            "write"
          ],
          "toolsBySender": {          // 按发件人限制工具
            "+15551234567": ["bash", "read", "write", "edit"],
            "+15559876543": ["read"]  // 只读权限
          }
        },
        "120363987654321098@g.us": {
          "requireMention": false,    // 无需提及
          "tools": ["read"]           // 仅查询工具
        }
      }
    }
  }
}
```

### 交互式设置向导

```typescript
// src/channels/plugins/onboarding/whatsapp.ts
export const whatsappOnboardingAdapter: ChannelOnboardingAdapter = {
  // 检查连接状态
  async getStatus(accountId?: string): Promise<ChannelOnboardingStatus> {
    const authDir = resolveDefaultWebAuthDir(accountId);
    const hasCredsFile = await hasWebCredsSync(authDir);

    if (!hasCredsFile) {
      return {
        linked: false,
        message: "WhatsApp 未连接",
      };
    }

    // 尝试读取已连接的号码
    const selfId = await readWebSelfId(authDir);
    if (selfId) {
      return {
        linked: true,
        message: `已连接: ${selfId}`,
        accountInfo: { phoneNumber: selfId },
      };
    }

    return {
      linked: false,
      message: "凭证文件损坏，请重新登录",
    };
  },

  // 交互式配置向导
  async configure(options: {
    accountId?: string;
    interactive?: boolean;
  }): Promise<ChannelOnboardingResult> {
    const { accountId, interactive = true } = options;

    if (!interactive) {
      return { success: false, message: "需要交互模式" };
    }

    console.log("=== WhatsApp 设置向导 ===\n");

    // 1. 检查是否已连接
    const status = await this.getStatus(accountId);
    if (status.linked) {
      const overwrite = await confirm({
        message: `已连接到 ${status.accountInfo?.phoneNumber}，是否重新登录？`,
        default: false,
      });

      if (!overwrite) {
        return { success: true, message: "保持现有连接" };
      }

      // 注销现有连接
      await logoutWeb(accountId);
    }

    // 2. 选择账户类型
    const accountType = await select({
      message: "选择 WhatsApp 账户类型：",
      choices: [
        { name: "专用号码（推荐）", value: "dedicated" },
        { name: "个人号码", value: "personal" },
      ],
    });

    // 3. QR 码登录
    console.log("\n请使用 WhatsApp 扫描以下二维码：");
    console.log("1. 打开 WhatsApp");
    console.log("2. 点击 设置 > 关联设备");
    console.log("3. 点击 关联设备");
    console.log("4. 扫描下方二维码\n");

    const authDir = resolveDefaultWebAuthDir(accountId);
    const loginResult = await loginWeb({
      authDir,
      printQr: true,
      timeout: 60_000,
    });

    if (!loginResult.success) {
      return {
        success: false,
        message: `登录失败: ${loginResult.error}`,
      };
    }

    console.log(`\n✓ 登录成功: ${loginResult.selfId}\n`);

    // 4. 配置 DM 策略
    const dmPolicy = await select({
      message: "选择私聊（DM）策略：",
      choices: [
        { name: "配对模式（需要配对码）", value: "pairing" },
        { name: "白名单模式（仅允许指定号码）", value: "allowlist" },
        { name: "开放模式（接受所有消息）", value: "open" },
        { name: "禁用私聊", value: "disabled" },
      ],
      default: accountType === "personal" ? "allowlist" : "pairing",
    });

    // 5. 配置白名单（如果需要）
    let allowFrom: string[] = [];
    if (dmPolicy === "allowlist" || dmPolicy === "pairing") {
      const numbers = await input({
        message: "输入允许的号码（逗号分隔，E.164 格式，如 +15551234567）：",
        validate: (value) => {
          if (!value.trim()) return "请输入至少一个号码";
          const nums = value.split(",").map(n => n.trim());
          const invalid = nums.filter(n => !n.startsWith("+"));
          if (invalid.length > 0) {
            return `号码必须以 + 开头: ${invalid.join(", ")}`;
          }
          return true;
        },
      });

      allowFrom = numbers.split(",").map(n => n.trim());
    }

    // 6. 配置群组策略
    const groupPolicy = await select({
      message: "选择群组策略：",
      choices: [
        { name: "开放（接受所有群组消息）", value: "open" },
        { name: "白名单（仅允许指定群组）", value: "allowlist" },
        { name: "禁用群组", value: "disabled" },
      ],
      default: accountType === "personal" ? "disabled" : "open",
    });

    // 7. 生成配置
    const config: WhatsAppConfig = {
      dmPolicy,
      allowFrom: allowFrom.length > 0 ? allowFrom : undefined,
      groupPolicy,
      sendReadReceipts: true,
      selfChatMode: accountType === "personal",
    };

    // 8. 写入配置文件
    const configPath = path.join(process.cwd(), "openclaw.json");
    const existingConfig = await readConfig(configPath);

    const updatedConfig = {
      ...existingConfig,
      channels: {
        ...existingConfig.channels,
        whatsapp: accountId
          ? {
              accounts: {
                ...existingConfig.channels?.whatsapp?.accounts,
                [accountId]: config,
              },
            }
          : config,
      },
    };

    await writeConfig(configPath, updatedConfig);

    console.log(`\n✓ 配置已保存到 ${configPath}`);
    console.log("\n启动网关:");
    console.log("  openclaw gateway run\n");

    return {
      success: true,
      message: "WhatsApp 配置完成",
      config,
    };
  },
};
```

## 与 Agent Loop 集成

### 网关启动流程

```typescript
// src/web/auto-reply/monitor.ts
export async function monitorWebChannel(
  verbose: boolean,
  listenerFactory: WebListenerFactory,
  keepAlive: KeepAliveOptions,
  replyResolver: ReplyResolver,
  runtime: WebChannelRuntime,
  abortSignal?: AbortSignal,
  tuning?: WebChannelTuning
): Promise<void> {
  const { cfg, accountId } = runtime;

  // 解析 WhatsApp 配置
  const whatsappCfg = resolveAccountConfig(cfg, accountId);
  const authDir = resolveAuthDirForAccount(cfg, accountId);

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
        authDir,
        mediaMaxMb: whatsappCfg.mediaMaxMb ?? 50,
        sendReadReceipts: whatsappCfg.sendReadReceipts ?? true,
        debounceMs: whatsappCfg.debounceMs ?? 0,

        // 消息处理器
        onMessage: createWebOnMessageHandler({
          cfg,
          accountId,
          replyResolver,
          runtime,
        }),

        // 连接事件
        onConnected: () => {
          log.info(`✓ WhatsApp connected (account: ${accountId})`);
          reconnectAttempts = 0;  // 重置重连计数
        },

        onDisconnected: (reason) => {
          log.warn(
            `WhatsApp disconnected (account: ${accountId}): ${reason}`
          );
        },

        onError: (error) => {
          log.error(
            `WhatsApp error (account: ${accountId}): ${error.message}`
          );
        },
      });

      // 注册活动监听器（用于出站）
      setActiveWebListener(accountId, {
        sendMessage: listener.sendMessage,
        sendPoll: listener.sendPoll,
        sendReaction: listener.sendReaction,
        sendComposingTo: listener.sendComposingTo,
      });

      // 启动心跳循环（可选）
      if (keepAlive.enabled) {
        startHeartbeatLoop({
          accountId,
          cfg,
          intervalMs: keepAlive.intervalMs ?? 60_000,
          recipients: resolveWhatsAppHeartbeatRecipients(cfg, accountId),
          abortSignal,
        });
      }

      // 等待断开或中止
      await listener.waitForDisconnect();

      // 清理活动监听器
      clearActiveWebListener(accountId);

      // 检查是否应该重连
      if (abortSignal?.aborted) break;

      reconnectAttempts++;
      if (reconnectAttempts >= maxReconnectAttempts) {
        log.error(
          `Max reconnection attempts reached for account ${accountId}`
        );
        break;
      }

      // 指数退避
      const backoffMs = Math.min(
        2_000 * Math.pow(2, reconnectAttempts - 1),
        30_000  // 最大 30 秒
      );

      log.info(
        `Reconnecting in ${(backoffMs / 1000).toFixed(1)}s ` +
        `(attempt ${reconnectAttempts}/${maxReconnectAttempts})`
      );

      await sleep(backoffMs);

    } catch (error) {
      log.error(`Fatal error in WhatsApp monitor: ${error}`);

      if (abortSignal?.aborted) break;

      // 短暂延迟后重试
      await sleep(5_000);
    }
  }

  log.info(`WhatsApp monitor stopped (account: ${accountId})`);
}
```

### 消息处理器

```typescript
// src/web/auto-reply/monitor/on-message.ts
export function createWebOnMessageHandler(options: {
  cfg: OpenClawConfig;
  accountId: string;
  replyResolver: ReplyResolver;
  runtime: WebChannelRuntime;
}): (msg: WebInboundMsg) => Promise<void> {
  const { cfg, accountId, replyResolver, runtime } = options;

  return async (msg: WebInboundMsg) => {
    // 1. Echo 检测
    if (msg.fromMe || msg.from === runtime.selfId) {
      return; // 跳过机器人自己的消息
    }

    // 2. 解析 Agent 路由
    const peerId = msg.chatType === "group" ? msg.chatJid : msg.from;
    const route = resolveAgentRoute({
      cfg,
      channel: "whatsapp",
      accountId,
      peer: {
        kind: msg.chatType,
        id: peerId,
      },
    });

    if (!route) {
      log.debug(`No route found for ${peerId}`);
      return;
    }

    // 3. 处理消息
    await processMessage({
      msg,
      route,
      cfg,
      accountId,

      // Agent 执行入口
      replyResolver: async (params) => {
        return getReplyFromConfig({
          ...params,
          channel: "whatsapp",
          accountId,
          provider: msg.chatType === "group" ? "whatsapp-group" : "whatsapp",
        });
      },

      // 回复分发器
      dispatchReply: async (reply) => {
        await dispatchReplyToWhatsApp({
          reply,
          to: peerId,
          accountId,
          chatType: msg.chatType,
          quotedMessageId: msg.messageId,
          cfg,
        });
      },
    });
  };
}
```

### Agent 路由解析

```typescript
// src/routing/resolve-route.ts
export function resolveAgentRoute(params: {
  cfg: OpenClawConfig;
  channel: string;
  accountId?: string;
  peer: { kind: string; id: string };
}): AgentRoute | null {
  const { cfg, channel, accountId, peer } = params;

  if (channel !== "whatsapp") return null;

  const whatsappCfg = resolveAccountConfig(cfg, accountId);

  // 1. 群组路由
  if (peer.kind === "group") {
    // 检查群组特定配置
    const groupCfg = whatsappCfg.groups?.[peer.id];
    if (groupCfg) {
      return {
        agentId: groupCfg.agentId ?? "default",
        sessionKey: `whatsapp:${accountId}:group:${peer.id}`,
        tools: groupCfg.tools,
        // ...
      };
    }
  }

  // 2. DM 路由
  if (peer.kind === "direct") {
    // 检查白名单中的特定绑定
    const allowFromEntry = whatsappCfg.allowFrom?.find(
      entry => typeof entry === "object" && entry.number === peer.id
    );

    if (allowFromEntry && allowFromEntry.agentId) {
      return {
        agentId: allowFromEntry.agentId,
        sessionKey: `whatsapp:${accountId}:direct:${peer.id}`,
        // ...
      };
    }
  }

  // 3. 默认路由
  return {
    agentId: cfg.agents?.defaults?.agentId ?? "default",
    sessionKey: `whatsapp:${accountId}:${peer.kind}:${peer.id}`,
  };
}
```

## 核心组件详解

### 1. WebSocket 管理 (session.ts)

```typescript
// src/web/session.ts
export async function createWaSocket(options: {
  authDir: string;
  printQr?: boolean;
  onQr?: (qr: string) => void;
  onConnected?: () => void;
  onDisconnected?: (reason: string) => void;
  onError?: (error: Error) => void;
  logger?: Logger;
}): Promise<WASocket> {
  const { authDir, logger = defaultLogger } = options;

  // 1. 初始化多文件认证状态
  const { state, saveCreds } = await useMultiFileAuthState(authDir);

  // 2. 创建 Baileys WebSocket
  const sock = makeWASocket({
    auth: state,
    printQRInTerminal: options.printQr ?? false,
    logger: wrapLogger(logger),

    // 浏览器信息（伪装为 WhatsApp Web）
    browser: ["OpenClaw", "Safari", "1.0.0"],

    // 同步配置
    syncFullHistory: false,
    markOnlineOnConnect: true,

    // 重连配置
    retryRequestDelayMs: 250,
    maxMsgRetryCount: 5,

    // 消息历史配置
    getMessage: async (key) => {
      // 尝试从本地缓存获取消息
      return await loadMessageFromCache(key);
    },
  });

  // 3. 监听连接更新
  sock.ev.on("connection.update", (update) => {
    const { connection, lastDisconnect, qr } = update;

    if (qr) {
      logger.debug("QR code received");
      options.onQr?.(qr);
    }

    if (connection === "close") {
      const shouldReconnect =
        (lastDisconnect?.error as any)?.output?.statusCode !==
        DisconnectReason.loggedOut;

      const reason =
        (lastDisconnect?.error as any)?.output?.statusCode?.toString() ??
        "unknown";

      logger.warn(`Connection closed: ${reason}`);
      options.onDisconnected?.(reason);

      if (!shouldReconnect) {
        logger.warn("Logged out, will not reconnect");
      }
    }

    if (connection === "open") {
      logger.info("✓ WhatsApp connected");
      options.onConnected?.();
    }
  });

  // 4. 监听凭证更新
  sock.ev.on("creds.update", async () => {
    try {
      await saveCreds();
      logger.debug("Credentials saved");
    } catch (error) {
      logger.error(`Failed to save credentials: ${error}`);
    }
  });

  // 5. 监听错误事件
  sock.ev.on("error", (error) => {
    logger.error(`Socket error: ${error.message}`);
    options.onError?.(error);
  });

  return sock;
}
```

### 2. 消息去重 (deduplication)

```typescript
// src/web/inbound/monitor.ts (部分)
import { LRUCache } from "lru-cache";

export function monitorWebInbox(/* ... */) {
  // 使用 LRU 缓存进行消息去重
  const dedupe = new LRUCache<string, boolean>({
    max: 1000,           // 最多缓存 1000 条
    ttl: 60_000,         // TTL: 60 秒
    updateAgeOnGet: false,
  });

  sock.ev.on("messages.upsert", async ({ messages }) => {
    for (const raw of messages) {
      // 生成去重键: accountId:remoteJid:messageId
      const dedupeKey = [
        accountId,
        raw.key.remoteJid,
        raw.key.id,
      ].join(":");

      // 检查是否已处理
      if (dedupe.has(dedupeKey)) {
        continue; // 跳过重复消息
      }

      // 标记为已处理
      dedupe.set(dedupeKey, true);

      // 处理消息...
    }
  });
}
```

### 3. 访问控制 (access-control.ts)

**配对码生成**：

```typescript
// src/web/inbound/access-control.ts
export async function generatePairingCode(
  from: string,
  accountId: string,
  cfg: OpenClawConfig
): Promise<PairingCode> {
  const code = generateRandomCode(6);  // 6 位数字
  const expiresAt = Date.now() + 3600_000;  // 1 小时后过期

  // 存储到配对存储
  const pairingStore = getPairingStore(cfg);
  await pairingStore.set({
    channel: "whatsapp",
    accountId,
    from,
    code,
    expiresAt,
  });

  return {
    code,
    expiresAt,
    instructions: [
      "To pair this number with OpenClaw:",
      `1. Send: /pair ${code}`,
      "2. The code expires in 1 hour",
    ].join("\n"),
  };
}
```

### 4. 防抖处理 (debounce)

```typescript
// src/auto-reply/inbound-debounce.ts
export function createInboundDebouncer(options: {
  debounceMs: number;
  onFlush: (messages: WebInboundMsg[]) => Promise<void>;
}): Debouncer {
  const { debounceMs, onFlush } = options;

  // 按发件人分组的待处理消息
  const pending = new Map<string, {
    messages: WebInboundMsg[];
    timer: NodeJS.Timeout;
  }>();

  return {
    async push(msg: WebInboundMsg) {
      const key = `${msg.accountId}:${msg.from}`;

      // 检查是否应跳过防抖
      const shouldSkip =
        msg.media ||           // 媒体消息
        msg.location ||        // 位置消息
        msg.quotedMsg ||       // 引用消息
        msg.text.startsWith("/");  // 控制命令

      if (shouldSkip || debounceMs === 0) {
        // 立即处理
        await onFlush([msg]);
        return;
      }

      // 添加到待处理队列
      const entry = pending.get(key);
      if (entry) {
        // 清除旧定时器
        clearTimeout(entry.timer);
        entry.messages.push(msg);
      } else {
        pending.set(key, {
          messages: [msg],
          timer: null as any,
        });
      }

      // 设置新定时器
      const newEntry = pending.get(key)!;
      newEntry.timer = setTimeout(async () => {
        const messages = newEntry.messages;
        pending.delete(key);

        // 合并消息文本
        const combinedText = messages
          .map(m => m.text)
          .join("\n");

        // 使用合并后的文本创建新消息
        const combined = {
          ...messages[0],
          text: combinedText,
        };

        await onFlush([combined]);
      }, debounceMs);
    },

    async flush() {
      for (const [key, entry] of pending.entries()) {
        clearTimeout(entry.timer);
        await onFlush(entry.messages);
        pending.delete(key);
      }
    },
  };
}
```

### 5. 心跳机制 (heartbeat)

```typescript
// src/channels/plugins/whatsapp-heartbeat.ts
export async function startHeartbeatLoop(options: {
  accountId: string;
  cfg: OpenClawConfig;
  intervalMs: number;
  recipients: string[];
  abortSignal?: AbortSignal;
}): Promise<void> {
  const { accountId, cfg, intervalMs, recipients, abortSignal } = options;

  log.info(
    `Starting heartbeat loop (account: ${accountId}, ` +
    `interval: ${intervalMs}ms, recipients: ${recipients.length})`
  );

  while (!abortSignal?.aborted) {
    try {
      // 等待间隔
      await sleep(intervalMs);

      if (abortSignal?.aborted) break;

      // 生成心跳消息
      const timestamp = new Date().toISOString();
      const message = `[heartbeat] ${timestamp}`;

      // 发送到所有接收者
      for (const recipient of recipients) {
        try {
          await sendMessageWhatsApp(recipient, message, {
            accountId,
            markdownToWhatsApp: false,
          });

          log.debug(`Heartbeat sent to ${recipient}`);
        } catch (error) {
          log.warn(`Failed to send heartbeat to ${recipient}: ${error}`);
        }
      }

    } catch (error) {
      log.error(`Heartbeat loop error: ${error}`);

      // 短暂延迟后继续
      await sleep(5_000);
    }
  }

  log.info(`Heartbeat loop stopped (account: ${accountId})`);
}

// 解析心跳接收者
export function resolveWhatsAppHeartbeatRecipients(
  cfg: OpenClawConfig,
  accountId?: string
): string[] {
  const whatsappCfg = resolveAccountConfig(cfg, accountId);

  // 优先级：
  // 1. 命令行参数 --to
  // 2. 会话存储中的最近联系人
  // 3. allowFrom 白名单

  const fromCli = process.argv.includes("--to")
    ? process.argv[process.argv.indexOf("--to") + 1]?.split(",")
    : undefined;

  if (fromCli && fromCli.length > 0) {
    return fromCli;
  }

  // 从会话存储读取
  const sessionStore = getSessionStore(cfg);
  const recentSessions = sessionStore.getRecentSessions({
    channel: "whatsapp",
    accountId,
    limit: 1,
  });

  if (recentSessions.length > 0) {
    return recentSessions.map(s => s.peerId);
  }

  // 使用白名单
  return whatsappCfg.allowFrom ?? [];
}
```

## 完整数据流图

```
┌─────────────────────────────────────────────────────────────────┐
│                    WhatsApp 集成完整流程                        │
└─────────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════
入站路径 (用户 → Agent)
═══════════════════════════════════════════════════════════════════

用户在 WhatsApp 发送消息
    ↓
WhatsApp 服务器 → WebSocket 事件
    ↓
Baileys Socket 接收 (src/web/session.ts)
    ↓
monitorWebInbox() (src/web/inbound/monitor.ts)
    ├─ 监听 "messages.upsert" 事件
    ├─ 提取消息: extractWebInboundMessage()
    │   ├─ JID 解析: "41796666864:0@s.whatsapp.net"
    │   ├─ E.164 规范化: "+41796666864"
    │   ├─ 聊天类型: "direct" | "group"
    │   ├─ 提取正文、媒体、提及、引用
    │   └─ 时间戳、消息 ID
    ├─ 去重检查: LRUCache<dedupeKey>
    │   └─ dedupeKey: "{accountId}:{remoteJid}:{messageId}"
    ├─ 访问控制: checkWebAccess()
    │   ├─ DM 策略检查
    │   │   ├─ "pairing": 检查配对/白名单
    │   │   ├─ "allowlist": 仅白名单
    │   │   ├─ "open": 允许所有
    │   │   └─ "disabled": 拒绝所有
    │   ├─ 群组策略检查
    │   │   ├─ 白名单验证
    │   │   └─ 提及要求检查
    │   └─ 生成配对码（如需要）
    ├─ 发送已读回执（可选）
    └─ 记录通道活动
    ↓
createInboundDebouncer() (src/auto-reply/inbound-debounce.ts)
    ├─ 媒体/位置/引用/命令: 跳过防抖
    ├─ 其他消息: 按发件人批处理
    └─ 防抖窗口后刷新: 合并文本
    ↓
createWebOnMessageHandler() (src/web/auto-reply/monitor/on-message.ts)
    ├─ Echo 检测: 跳过 fromMe 消息
    ├─ 群组历史跟踪
    ├─ 解析 Agent 路由: resolveAgentRoute()
    │   ├─ 输入: channel="whatsapp", accountId, peer
    │   ├─ 群组路由: 检查 groups[groupJid]
    │   ├─ DM 路由: 检查 allowFrom 绑定
    │   └─ 输出: { agentId, sessionKey, tools, ... }
    └─ processMessage()
    ↓
getReplyFromConfig() (auto-reply 核心)
    ├─ 构建会话上下文
    ├─ 应用工具限制
    ├─ 执行 Agent Loop
    └─ 返回回复
    ↓
dispatchReplyToWhatsApp()
    ├─ 准备回复内容
    ├─ 文本分块（4000 字符）
    └─ 调用出站适配器
    ↓

═══════════════════════════════════════════════════════════════════
出站路径 (Agent → 用户)
═══════════════════════════════════════════════════════════════════

Agent 生成回复
    ↓
dispatchReply() (auto-reply/reply/dispatch-from-config.ts)
    ├─ 选择通道适配器
    ├─ 准备消息负载
    └─ 调用 whatsappOutbound
    ↓
whatsappOutbound (src/channels/plugins/outbound/whatsapp.ts)
    ├─ resolveTarget()
    │   ├─ 验证 E.164 格式
    │   ├─ 白名单检查（strict 模式）
    │   └─ 返回规范化目标
    ├─ chunker: chunkText()
    │   └─ 按 textChunkLimit (4000) 分块
    ├─ sendText() / sendMedia() / sendPoll()
    └─ 调用 sendMessageWhatsApp()
    ↓
sendMessageWhatsApp() (src/web/outbound.ts)
    ├─ E.164 → JID 转换
    │   └─ "+15551234567" → "15551234567@s.whatsapp.net"
    ├─ Markdown 转换: convertMarkdownToWhatsApp()
    │   ├─ **粗体** → *粗体*
    │   ├─ ~~删除线~~ → ~删除线~
    │   └─ 保护代码块
    ├─ 加载媒体（如果提供）
    │   ├─ 下载 URL
    │   ├─ 大小检查 (maxMb)
    │   └─ 转换为 Buffer
    ├─ 获取活动监听器: requireActiveWebListener(accountId)
    └─ active.sendMessage(toJid, text, mediaBuffer, mediaType, options)
    ↓
ActiveWebListener.sendMessage()
    ├─ 使用 Baileys socket
    ├─ 构建消息负载
    └─ sock.sendMessage(toJid, { text, ... })
    ↓
Baileys → WhatsApp 服务器 (WebSocket)
    ↓
消息发送到用户的 WhatsApp
    ↓
后处理（可选）:
    ├─ 发送确认反应（ackReaction）
    └─ 记录发送统计

═══════════════════════════════════════════════════════════════════
认证路径 (登录流程)
═══════════════════════════════════════════════════════════════════

用户执行: openclaw channels login --channel whatsapp
    ↓
loginWeb(options) (src/web/auth-store.ts)
    ├─ 检查现有凭证
    ├─ authDir = resolveDefaultWebAuthDir(accountId)
    └─ 调用 createWaSocket(printQr=true)
    ↓
createWaSocket() (src/web/session.ts)
    ├─ useMultiFileAuthState(authDir)
    ├─ makeWASocket({ auth: state, printQRInTerminal: true })
    └─ 监听 "connection.update"
    ↓
Baileys 生成 QR 码
    ├─ 终端显示二维码（ASCII 艺术）
    └─ 等待扫描
    ↓
用户扫描 QR 码
    ├─ 打开 WhatsApp
    ├─ 设置 > 关联设备 > 关联设备
    └─ 扫描二维码
    ↓
WhatsApp 服务器验证 → 建立 WebSocket 连接
    ↓
connection.update: { connection: "open" }
    ↓
saveCreds() 自动触发
    ├─ 保存到 {authDir}/creds.json
    ├─ 文件权限: 0o600
    ├─ 创建备份: creds.json.bak
    └─ 保存其他 Baileys 状态文件
    ↓
✓ 登录成功
    ├─ 读取 selfId: readWebSelfId(authDir)
    ├─ 显示: "✓ 已连接: +41796666864"
    └─ 会话准备就绪

═══════════════════════════════════════════════════════════════════
网关启动路径 (后台服务)
═══════════════════════════════════════════════════════════════════

用户执行: openclaw gateway run
    ↓
startGatewayServer() (src/gateway/server-startup.ts)
    ├─ 加载配置
    ├─ 初始化插件运行时
    └─ 启动通道监听器
    ↓
PluginRuntime.channel.whatsapp.monitorWebChannel()
    ├─ 加载 WhatsApp 配置
    ├─ 解析账户详情
    └─ 进入重连循环
    ↓
monitorWebChannel() - 主循环 (src/web/auto-reply/monitor.ts)
    While (不中止):
        ├─ monitorWebInbox() → 创建 socket 监听器
        ├─ setActiveWebListener() → 注册用于出站
        ├─ 监听 onMessage 事件 → 处理入站消息
        ├─ 断开时:
        │   ├─ 清理监听器
        │   ├─ reconnectAttempts++
        │   ├─ 指数退避: backoff = min(2^n * 2s, 30s)
        │   └─ 睡眠后重试 (最多 12 次尝试)
        ├─ 心跳循环 (可选):
        │   ├─ 间隔: 60 秒（默认）
        │   ├─ 发送到: allowFrom 或最近联系人
        │   └─ 消息: "[heartbeat] {timestamp}"
        └─ 看门狗超时: 30 分钟无消息 → 警告
    ↓
✓ 网关运行中
    ├─ 接收消息 → Agent 处理 → 发送回复
    ├─ 自动重连（断开时）
    └─ 健康检查（心跳）
```

## 安全性与最佳实践

### 1. 凭证安全

```typescript
// 文件权限
await fs.chmod(credsPath, 0o600);  // 仅所有者读写

// 从不记录敏感信息
log.debug(`Saving creds to ${redactPath(credsPath)}`);

// 自动备份机制
await fs.copyFile(credsPath, `${credsPath}.bak`);
```

### 2. 访问控制分层

```
第 1 层: DM 策略 (pairing/allowlist/open/disabled)
第 2 层: 白名单 (allowFrom)
第 3 层: 群组策略 (open/allowlist/disabled)
第 4 层: 群组白名单 (groupAllowFrom)
第 5 层: 提及要求 (requireMention)
第 6 层: 工具限制 (tools, toolsBySender)
```

### 3. 速率限制

```typescript
// WhatsApp 的非官方限制（经验值）:
// - 每秒 ~10 条消息
// - 每分钟 ~600 条消息
// - 超过限制可能导致临时封禁

// OpenClaw 内置保护:
const rateLimiter = createRateLimiter({
  maxPerSecond: 5,      // 保守限制
  maxPerMinute: 100,
  maxPerHour: 1000,
});
```

### 4. 错误处理

```typescript
// 优雅降级
try {
  await sendMessageWhatsApp(to, text);
} catch (error) {
  if (isRateLimitError(error)) {
    // 等待并重试
    await sleep(5_000);
    return sendMessageWhatsApp(to, text);
  }

  if (isNotLinkedError(error)) {
    throw new Error(
      "WhatsApp not linked. Run: openclaw channels login --channel whatsapp"
    );
  }

  // 记录并重新抛出
  log.error(`Failed to send message: ${error}`);
  throw error;
}
```

### 5. 最佳实践清单

- ✅ **使用专用号码**：不要在个人号上运行（可能违反 ToS）
- ✅ **启用白名单**：使用 `dmPolicy: "allowlist"`
- ✅ **限制工具**：按群组/用户配置 `tools`
- ✅ **监控连接**：检查心跳和状态报告
- ✅ **定期备份**：备份 `~/.openclaw/oauth/whatsapp/`
- ✅ **使用配对模式**：新用户需要配对码
- ✅ **群组提及**：群组中设置 `requireMention: true`
- ✅ **日志审计**：启用详细日志记录敏感操作

---

## 总结

WhatsApp 通过 Baileys 库以 **Web 通道**形式深度集成到 OpenClaw 中，提供了：

### 核心优势

1. **QR 码登录**：类似 WhatsApp Web 的便捷配对
2. **多账户**：支持同时运行多个 WhatsApp 账户
3. **完整媒体**：图片、视频、音频、文档、语音消息
4. **智能路由**：按发件人、群组、提及进行灵活路由
5. **访问控制**：多层安全策略（DM、群组、白名单、提及）
6. **自动重连**：带指数退避的健壮连接恢复
7. **Agent 集成**：无缝对接 Agent Loop 和工具系统

### 关键设计模式

1. **事件驱动**：基于 Baileys EventEmitter 的异步架构
2. **适配器模式**：统一的出站接口适配不同通道
3. **策略模式**：可配置的访问控制策略
4. **发布-订阅**：消息流通过事件总线
5. **断路器**：重连循环中的退避和限制

### 技术栈

- **Baileys**：WhatsApp Web 协议实现
- **WebSocket**：实时双向通信
- **Protobuf**：WhatsApp 消息格式
- **Multi-file Auth**：分布式认证状态
- **LRU Cache**：消息去重
- **Zod**：配置验证

---

**文档版本**: 1.0
**最后更新**: 2026-02-19
**分析范围**: OpenClaw WhatsApp Web 通道集成
