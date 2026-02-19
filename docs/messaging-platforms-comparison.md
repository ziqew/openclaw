# 消息平台集成对比分析

> 本文档详细分析了 WhatsApp、Slack、Telegram、Discord 四个消息平台如何接入 OpenClaw，并对比了它们的异同

## 目录

1. [概述](#概述)
2. [WhatsApp 集成](#whatsapp-集成)
3. [Slack 集成](#slack-集成)
4. [Telegram 集成](#telegram-集成)
5. [Discord 集成](#discord-集成)
6. [架构对比](#架构对比)
7. [功能对比矩阵](#功能对比矩阵)
8. [设置复杂度对比](#设置复杂度对比)
9. [选型建议](#选型建议)

---

## 概述

OpenClaw 支持四个主流消息平台的接入，每个平台都有其独特的架构和实现方式。本文档详细分析了它们的技术实现、优缺点和使用场景。

### 四大平台定位

| 平台 | 类型 | 主要场景 | 官方 API |
|------|------|---------|---------|
| **WhatsApp** | 个人/企业通讯 | 端到端加密通信 | WhatsApp Web (非官方) |
| **Slack** | 企业协作 | 团队工作空间 | 官方 Bot API |
| **Telegram** | 开放平台 | Bot 生态系统 | 官方 Bot API |
| **Discord** | 社区/游戏 | 服务器社区 | 官方 Gateway API |

---

## WhatsApp 集成

### 连接架构

```
WhatsApp 服务器
    ↕ (WebSocket)
Baileys 库 (WhatsApp Web 模拟)
    ↓
Web 会话管理器 (src/web/session.ts)
    ↓
入站监听器 (src/web/inbound/monitor.ts)
    ↓
消息处理 → Agent Loop
    ↓
出站适配器 (src/channels/plugins/outbound/whatsapp.ts)
```

### 认证方式

- **QR 码登录**：类似 WhatsApp Web 的配对方式
- **凭证存储**：`~/.openclaw/oauth/whatsapp/{accountId}/creds.json`
- **安全性**：文件权限 0o600，自动备份
- **多账户**：每个账户独立的认证目录

### 技术栈

- **库**: Baileys (WhatsApp Web 协议实现)
- **协议**: WebSocket + Protobuf
- **认证**: Multi-file auth state (分布式)

### 核心特性

```typescript
// 关键文件
src/web/session.ts              // Socket 创建和连接管理
src/web/inbound/monitor.ts      // 消息接收管道
src/web/outbound.ts             // 消息发送实现
src/web/auth-store.ts           // 认证存储和恢复
src/whatsapp/normalize.ts       // E.164 ↔ JID 转换
src/markdown/whatsapp.ts        // Markdown 格式转换
```

### 消息流程

**入站**:
1. Baileys Socket 接收 `messages.upsert` 事件
2. 提取消息：JID → E.164，解析正文/媒体/提及
3. LRU 缓存去重（dedupeKey: `accountId:remoteJid:messageId`）
4. 访问控制：DM 策略、白名单、群组策略
5. 防抖处理：批处理连续消息
6. Agent 路由解析 → 执行

**出站**:
1. Agent 生成回复
2. Markdown → WhatsApp 格式转换
3. 获取活动监听器 (`requireActiveWebListener`)
4. E.164 → JID 转换
5. 通过 Baileys Socket 发送

### 配置示例

```json5
{
  "channels": {
    "whatsapp": {
      "dmPolicy": "allowlist",
      "allowFrom": ["+15551234567"],
      "groupPolicy": "allowlist",
      "groupAllowFrom": ["120363123456789012@g.us"],
      "sendReadReceipts": true,
      "mediaMaxMb": 50,
      "textChunkLimit": 4000,
      "ackReaction": {
        "emoji": "👍",
        "direct": true,
        "group": "mentions"
      }
    }
  }
}
```

### 优势

✅ 端到端加密（原生 WhatsApp）
✅ 全球用户基础大
✅ 支持语音消息（PTT）
✅ 媒体质量高（不压缩）
✅ 自动重连机制健壮

### 限制

⚠️ 非官方 API（可能违反 ToS）
⚠️ 连接不稳定时需要重新扫码
⚠️ 速率限制严格（~5 消息/秒）
⚠️ 需要一直保持连接
⚠️ 不适合大规模自动化

---

## Slack 集成

### 连接架构

```
Slack API
    ↕
Socket Mode (WebSocket) 或 HTTP Events API
    ↓
Slack Bolt App (src/slack/monitor/provider.ts)
    ↓
事件处理器 → 消息路由
    ↓
Agent Loop
    ↓
Slack Web API 客户端 (src/slack/client.ts)
```

### 认证方式

- **两种模式**：
  - **Socket Mode**（默认）：WebSocket 连接，无需公网 URL
  - **HTTP Mode**：Webhook 模式，需要公网 URL
- **Token 类型**：
  - Bot Token (`xoxb-...`)：主要认证
  - App Token (`xapp-...`)：Socket Mode 连接
  - User Token（可选）：读取私人信息
- **环境变量**：`SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN`

### 技术栈

- **库**: `@slack/bolt` (官方 SDK)
- **协议**: Socket Mode (WebSocket) 或 Events API (HTTP)
- **重试策略**: 2 次重试，指数退避

### 核心特性

```typescript
// 关键文件
src/slack/monitor/provider.ts        // 主入口，创建 Bolt App
src/slack/monitor/message-handler.ts // 消息处理和防抖
src/slack/client.ts                  // Web API 客户端
src/slack/send.ts                    // 消息发送
src/slack/allowlist.ts               // 用户/频道白名单解析
```

### 消息流程

**入站**:
1. Slack 事件通过 Socket/HTTP 到达
2. Bolt App 路由到事件处理器
3. 消息去重（已见过的消息）
4. 防抖处理：合并连续输入
5. 解析线程上下文 (`thread_ts`)
6. 剥离提及符号用于命令检测
7. `prepareSlackMessage()` 构建上下文
8. `dispatchPreparedSlackMessage()` 路由到 Agent

**出站**:
1. Agent 生成回复
2. 文本分块（40,000 字符限制）
3. Slack Web API 发送
4. 支持线程回复、reactions、文件上传

### 配置示例

```json5
{
  "channels": {
    "slack": {
      "mode": "socket",  // 或 "http"
      "botToken": "xoxb-...",
      "appToken": "xapp-...",  // Socket mode
      "dm": {
        "enabled": true,
        "policy": "allowlist",
        "allowFrom": ["U01ABC123", "@john"],
        "groupEnabled": false
      },
      "channels": {
        "C01XYZ789": {
          "enabled": true,
          "requireMention": true,
          "tools": ["bash", "read", "write"]
        }
      },
      "groupPolicy": "allowlist",
      "replyToMode": "first"  // "off" | "first" | "all"
    }
  }
}
```

### 优势

✅ 官方 API，稳定可靠
✅ 企业级功能（SSO、审计日志）
✅ 丰富的交互组件（Block Kit）
✅ Socket Mode 无需公网 URL
✅ 强大的权限管理
✅ 线程化对话原生支持

### 限制

⚠️ 设置复杂（需要创建 App、配置事件、安装）
⚠️ 付费版才有完整功能
⚠️ 速率限制（Tier-based）
⚠️ Message 更新限制（5 分钟内）

---

## Telegram 集成

### 连接架构

```
Telegram Bot API
    ↕
Long Polling (默认) 或 Webhook
    ↓
grammY Bot Framework (src/telegram/bot.ts)
    ↓
Middleware 链（节流、顺序处理）
    ↓
消息处理器 → Agent Loop
    ↓
Telegram API 发送 (src/telegram/send.ts)
```

### 认证方式

- **Bot Token**：从 BotFather 获取
- **Token 解析顺序**：`tokenFile` → `botToken` → `TELEGRAM_BOT_TOKEN` 环境变量
- **多账户**：每个账户独立 token
- **Token 格式**：标准 Telegram bot token（`123456:ABC-DEF...`）

### 技术栈

- **库**: grammY (现代 Telegram Bot 框架)
- **协议**: HTTPS Long Polling 或 Webhook
- **Middleware**: 节流、顺序处理、错误捕获
- **代理支持**: HTTP/SOCKS 代理（区域封锁）

### 核心特性

```typescript
// 关键文件
src/telegram/bot.ts                    // Bot 创建和 middleware
src/telegram/bot-message.ts           // 消息处理器工厂
src/telegram/send.ts                   // 发送（文本、媒体、语音）
src/telegram/streaming.ts              // 流式消息更新
src/telegram/inline-buttons.ts        // 内联按钮支持
src/telegram/reactions.ts             // 反应处理
```

### 消息流程

**入站**:
1. Telegram 发送 update（轮询或 webhook）
2. grammY middleware 链：
   - API 节流
   - 按聊天顺序处理
   - 错误捕获
3. Update 去重检查
4. `getTelegramSequentialKey()` 确保顺序处理
5. 原生命令处理（可选）
6. 反应处理
7. 消息处理器：
   - 构建历史上下文
   - 检查白名单和策略
   - 解析群组/话题配置
8. `dispatchTelegramMessage()` 路由到 Agent

**出站**:
1. Agent 生成回复
2. 选择流式模式：
   - `off`: 不流式
   - `partial`: 更新现有消息
   - `block`: 发送消息块
3. 文本分块（4000 字符限制）
4. 标题分割（1024 字符限制）
5. Telegram API 发送
6. 支持线程、话题、媒体组

### 配置示例

```json5
{
  "channels": {
    "telegram": {
      "botToken": "123456:ABC-DEF...",
      "dmPolicy": "allowlist",
      "allowFrom": [123456789, "@username"],
      "groupAllowFrom": [-1001234567890],
      "groupPolicy": "allowlist",
      "groups": {
        "-1001234567890": {
          "enabled": true,
          "requireMention": true,
          "tools": ["bash", "read"],
          "topics": {
            "42": {
              "requireMention": false,
              "tools": ["bash", "read", "write"]
            }
          }
        }
      },
      "streamMode": "partial",  // "off" | "partial" | "block"
      "replyToMode": "first",
      "proxy": "socks5://localhost:1080"
    }
  }
}
```

### 独特功能

🌟 **论坛话题**：完整支持 Telegram 论坛超级群组
🌟 **媒体组**：收集并一起处理相册消息
🌟 **流式模式**：三种流式更新模式
🌟 **Sticker 缓存**：缓存 sticker file_id 供重用
🌟 **代理支持**：HTTP/SOCKS 代理，突破区域封锁
🌟 **视频笔记**：圆形视频消息支持

### 优势

✅ 官方 Bot API，功能强大
✅ 设置最简单（单个 token）
✅ 无速率限制（合理使用）
✅ 开放的 Bot 生态
✅ 支持大文件（20MB bots, 2GB premium）
✅ 流式消息更新
✅ 内联按钮和键盘
✅ 代理支持

### 限制

⚠️ 某些地区被封锁
⚠️ 消息编辑有时间限制（48小时）
⚠️ Markdown 语法受限（有特殊字符转义要求）

---

## Discord 集成

### 连接架构

```
Discord Gateway
    ↕ (WebSocket)
Carbon Framework (src/discord/monitor/provider.ts)
    ↓
Gateway Plugin（presence、intents、reconnection）
    ↓
事件监听器（消息、反应、presence）
    ↓
消息处理 → Agent Loop
    ↓
Discord API 发送 (src/discord/send*.ts)
```

### 认证方式

- **Bot Token**：从 Discord Developer Portal 获取
- **Application ID**：自动从 Discord API 获取
- **Intents**（权限）：
  - Message Content（必需）
  - Server Members（推荐，用于白名单）
  - Guild Presences（可选，用于 presence 更新）
- **环境变量**：`DISCORD_BOT_TOKEN`

### 技术栈

- **库**: `@buape/carbon` (Discord.js 替代品)
- **协议**: Gateway WebSocket
- **特殊功能**: 原生斜杠命令、交互组件

### 核心特性

```typescript
// 关键文件
src/discord/monitor/provider.ts        // 主入口，创建 Carbon Client
src/discord/monitor/message-handler.ts // 消息处理和预检
src/discord/send.ts                    // 消息发送
src/discord/send-poll.ts               // 投票
src/discord/send-voice.ts              // 语音消息
src/discord/commands/                  // 斜杠命令处理
src/discord/gateway-plugin.ts          // 自定义 Gateway 插件
```

### 消息流程

**入站**:
1. Discord Gateway 事件到达
2. Carbon 监听器处理：
   - `DiscordMessageListener` - 消息
   - `DiscordReactionListener` - 反应
   - `DiscordPresenceListener` - Presence 更新
3. 消息防抖：合并快速输入
4. `preflightDiscordMessage()` 验证：
   - 服务器/频道白名单
   - 用户/角色白名单
   - Bot 权限检查
5. `processDiscordMessage()` 构建 Agent 上下文：
   - 线程启动器上下文（可选）
   - 服务器历史
   - 媒体附件
6. 路由到 Agent Loop
7. 响应分块发送（2000 字符限制，17 行软限制）

**出站**:
1. Agent 生成回复
2. 文本分块（2000 字符）
3. Discord API 发送
4. 支持：threads、reactions、embeds、buttons、select menus

### 配置示例

```json5
{
  "channels": {
    "discord": {
      "token": "MTk4NjIyNDgzNDcxOTI1MjQ4.GK7PC8.xncdA",
      "dm": {
        "enabled": true,
        "policy": "allowlist",
        "allowFrom": ["123456789012345678", "@username#1234"],
        "groupEnabled": false
      },
      "guilds": {
        "987654321098765432": {
          "enabled": true,
          "name": "My Server",
          "channels": {
            "111222333444555666": {
              "enabled": true,
              "requireMention": true,
              "allowFrom": ["123456789012345678"],
              "tools": ["bash", "read", "write"]
            }
          }
        }
      },
      "groupPolicy": "allowlist",
      "intents": {
        "presence": true,
        "guildMembers": true
      },
      "execApprovals": {
        "enabled": true,
        "approvers": ["123456789012345678"]
      }
    }
  }
}
```

### 独特功能

🌟 **基于角色的路由**：按 Discord 角色将用户路由到不同 Agent
🌟 **执行审批**：将命令审批请求转发到 Discord DM
🌟 **PluralKit 集成**：解析代理消息身份
🌟 **原生斜杠命令**：自动注册命令，支持自动完成
🌟 **Agent 组件**：Agent 控制的按钮和选择菜单
🌟 **Presence 更新**：Bot 状态和活动控制

### 优势

✅ 官方 API，功能完整
✅ 强大的交互组件（按钮、选择菜单、模态框）
✅ 原生斜杠命令系统
✅ 角色和权限系统
✅ 线程化对话
✅ 丰富的嵌入消息（embeds）
✅ 语音频道（未来可能支持）

### 限制

⚠️ 需要启用 Message Content Intent（需审核）
⚠️ 速率限制较严格
⚠️ 消息字符限制较小（2000）
⚠️ WebSocket 连接需要持续维护

---

## 架构对比

### 1. 连接模式对比

| 平台 | 连接方式 | 协议 | 框架/库 | 重连机制 |
|------|---------|------|---------|---------|
| **WhatsApp** | WebSocket (长连接) | WhatsApp Web | Baileys | 指数退避，最多 12 次 |
| **Slack** | Socket Mode / HTTP | WebSocket / HTTP | Bolt | Bolt 内置重连 |
| **Telegram** | Long Polling / Webhook | HTTPS | grammY | grammY 自动重试 |
| **Discord** | Gateway WebSocket | WebSocket | Carbon | Carbon Gateway 插件 |

### 2. 认证方式对比

| 平台 | 认证方式 | Token 类型 | 存储位置 |
|------|---------|-----------|---------|
| **WhatsApp** | QR 码扫描 | Multi-file auth state | `~/.openclaw/oauth/whatsapp/` |
| **Slack** | OAuth / Token | Bot + App Token | 配置文件 / 环境变量 |
| **Telegram** | Bot Token | 单一 token | 配置文件 / 环境变量 |
| **Discord** | Bot Token | 单一 token + intents | 配置文件 / 环境变量 |

### 3. 消息处理流程对比

所有平台都遵循类似的流程，但实现细节不同：

```
入站通用流程:
接收事件 → 去重 → 防抖 → 访问控制 → 上下文构建 → Agent 路由 → 执行

出站通用流程:
生成回复 → 格式转换 → 分块 → 发送 → 后处理
```

**关键差异**：

| 阶段 | WhatsApp | Slack | Telegram | Discord |
|------|----------|-------|----------|---------|
| **去重** | LRU Cache | Set | Update ID | Message ID |
| **防抖** | 0ms 默认 | 默认启用 | 默认启用 | 默认启用 |
| **访问控制** | JID + E.164 | User/Channel ID | User/Chat ID | User/Role/Channel ID |
| **格式转换** | MD → WhatsApp | 无需 | MD → Telegram | MD → Discord |
| **分块限制** | 4000 | 40,000 | 4000 | 2000 |

### 4. Agent Loop 集成对比

所有平台使用统一的 Agent 路由系统：

```typescript
// 统一路由接口
resolveAgentRoute({
  cfg: OpenClawConfig,
  channel: "whatsapp" | "slack" | "telegram" | "discord",
  accountId?: string,
  peer: { kind: string, id: string }
}) → AgentRoute
```

**会话密钥格式**：
- WhatsApp: `whatsapp:{accountId}:{chatType}:{peerId}`
- Slack: `slack:{accountId}:{channelId}`
- Telegram: `telegram:{accountId}:{chatId}`
- Discord: `discord:{accountId}:{channelId}`

### 5. 配置层次对比

| 层级 | WhatsApp | Slack | Telegram | Discord |
|------|----------|-------|----------|---------|
| **全局策略** | dmPolicy, groupPolicy | dm.policy, groupPolicy | dmPolicy, groupPolicy | dm.policy, groupPolicy |
| **账户级** | accounts[id] | accounts[id] | accounts[id] | accounts[id] |
| **组/频道级** | groups[jid] | channels[id] | groups[id] | guilds[id].channels[id] |
| **用户级** | allowFrom | dm.allowFrom | allowFrom | guilds[id].allowFrom |
| **工具限制** | groups[jid].tools | channels[id].tools | groups[id].tools | channels[id].tools |

---

## 功能对比矩阵

### 基础功能

| 功能 | WhatsApp | Slack | Telegram | Discord |
|------|:--------:|:-----:|:--------:|:-------:|
| **文本消息** | ✅ | ✅ | ✅ | ✅ |
| **媒体消息** | ✅ | ✅ | ✅ | ✅ |
| **语音消息** | ✅ (PTT) | ❌ | ✅ | ✅ |
| **文件发送** | ✅ | ✅ | ✅ | ✅ |
| **位置分享** | ✅ | ❌ | ✅ | ❌ |
| **联系人卡片** | ✅ | ❌ | ✅ | ❌ |
| **投票** | ✅ | ❌ | ✅ | ✅ |
| **反应** | ✅ | ✅ | ✅ | ✅ |

### 高级功能

| 功能 | WhatsApp | Slack | Telegram | Discord |
|------|:--------:|:-----:|:--------:|:-------:|
| **线程/话题** | ❌ | ✅ | ✅ (Forum) | ✅ |
| **流式更新** | ❌ | ❌ | ✅ (3 modes) | ❌ |
| **原生命令** | ❌ | ✅ (opt-in) | ✅ | ✅ (auto) |
| **交互组件** | ❌ | ✅ (Block Kit) | ✅ (Inline) | ✅ (Full) |
| **嵌入消息** | ❌ | ✅ | ❌ | ✅ |
| **自定义键盘** | ❌ | ❌ | ✅ | ❌ |
| **角色路由** | ❌ | ❌ | ❌ | ✅ |
| **代理支持** | ❌ | ❌ | ✅ | ✅ (WS) |

### 访问控制

| 功能 | WhatsApp | Slack | Telegram | Discord |
|------|:--------:|:-----:|:--------:|:-------:|
| **DM 策略** | ✅ | ✅ | ✅ | ✅ |
| **白名单** | ✅ | ✅ | ✅ | ✅ |
| **黑名单** | ❌ | ❌ | ❌ | ❌ |
| **配对系统** | ✅ | ✅ | ✅ | ✅ |
| **提及要求** | ✅ | ✅ | ✅ | ✅ |
| **工具限制** | ✅ | ✅ | ✅ | ✅ |
| **按用户工具** | ✅ | ❌ | ❌ | ❌ |
| **按角色路由** | ❌ | ❌ | ❌ | ✅ |

### 技术特性

| 特性 | WhatsApp | Slack | Telegram | Discord |
|------|:--------:|:-----:|:--------:|:-------:|
| **多账户** | ✅ | ✅ | ✅ | ✅ |
| **自动重连** | ✅ | ✅ | ✅ | ✅ |
| **消息去重** | ✅ | ✅ | ✅ | ✅ |
| **消息防抖** | ✅ | ✅ | ✅ | ✅ |
| **顺序处理** | ❌ | ❌ | ✅ | ❌ |
| **批量操作** | ❌ | ✅ | ❌ | ✅ |
| **Webhook** | ❌ | ✅ | ✅ | ❌ |
| **已读回执** | ✅ | ✅ | ❌ | ❌ |

### 限制对比

| 限制类型 | WhatsApp | Slack | Telegram | Discord |
|---------|----------|-------|----------|---------|
| **消息长度** | 4,000 | 40,000 | 4,096 | 2,000 |
| **文件大小** | 50 MB | 20 MB | 20 MB (bot) | 8 MB (25 MB boost) |
| **媒体类型** | 全支持 | 全支持 | 全支持 | 全支持 |
| **速率限制** | ~5/秒 | Tier-based | 30/秒 (bot) | 5/秒 (varies) |
| **并发连接** | 1 per account | Multiple | Unlimited | 1 per shard |

---

## 设置复杂度对比

### 难度排序（从简单到困难）

1. **Telegram** ⭐
   - ✅ 单个 token（从 BotFather）
   - ✅ 立即可用，无需配置
   - ✅ 一行命令启动

2. **Discord** ⭐⭐
   - Token 获取简单
   - ⚠️ 需要在 Developer Portal 启用 intents
   - ⚠️ 斜杠命令需要注册

3. **WhatsApp** ⭐⭐⭐
   - QR 码扫描简单
   - ⚠️ 需要专用号码或承担风险
   - ⚠️ 连接不稳定需要重新扫码
   - ⚠️ 不适合生产环境

4. **Slack** ⭐⭐⭐⭐
   - ⚠️ 创建 App
   - ⚠️ 配置 events subscriptions
   - ⚠️ 安装到 workspace
   - ⚠️ 获取多个 token
   - ⚠️ 配置 scopes 和 permissions

### 最少配置示例

**Telegram** (最简):
```json5
{
  "channels": {
    "telegram": {
      "botToken": "123456:ABC-DEF...",
      "dmPolicy": "open"
    }
  }
}
```

**Discord**:
```json5
{
  "channels": {
    "discord": {
      "token": "MTk4NjIyNDgzNDcxOTI1MjQ4.GK7PC8.xncdA",
      "dm": { "enabled": true }
    }
  }
}
```

**WhatsApp**:
```json5
{
  "channels": {
    "whatsapp": {
      "dmPolicy": "allowlist",
      "allowFrom": ["+15551234567"]
    }
  }
}
// + 手动 QR 码登录
```

**Slack** (最复杂):
```json5
{
  "channels": {
    "slack": {
      "mode": "socket",
      "botToken": "xoxb-...",
      "appToken": "xapp-...",
      "dm": { "enabled": true }
    }
  }
}
```

---

## 选型建议

### 个人使用 / 小型团队

**首选：Telegram**
- ✅ 设置最简单
- ✅ 功能最丰富
- ✅ 无速率限制
- ✅ 流式消息更新
- ✅ 代理支持

**备选：Discord**
- ✅ 如果已有 Discord 社区
- ✅ 需要角色权限系统
- ✅ 需要斜杠命令

### 企业使用

**首选：Slack**
- ✅ 企业级功能
- ✅ SSO 集成
- ✅ 审计日志
- ✅ 合规性
- ⚠️ 需要付费版

**备选：Discord**
- ✅ 如果团队使用 Discord
- ✅ 成本更低
- ✅ 社区功能强

### 客户服务 / 支持

**WhatsApp** (有风险)
- ✅ 全球用户基础
- ✅ 高度个人化
- ⚠️ 非官方 API
- ⚠️ 不适合大规模

**Telegram**
- ✅ Bot 生态成熟
- ✅ 群组功能强大
- ✅ 无使用限制

### 游戏社区 / Discord 原生

**Discord**
- ✅ 原生斜杠命令
- ✅ 角色和权限
- ✅ 语音频道
- ✅ 嵌入消息

---

## 技术实现亮点

### 1. 统一抽象层

所有平台共享相同的核心架构：

```typescript
// 统一的消息上下文
interface MessageContext {
  channel: "whatsapp" | "slack" | "telegram" | "discord";
  accountId: string;
  chatType: "direct" | "group" | "channel";
  from: string;
  to: string;
  text: string;
  media?: MediaPayload[];
  history?: HistoryEntry[];
  sessionKey: string;
  messageId: string;
  wasMentioned?: boolean;
}

// 统一的路由解析
resolveAgentRoute(context) → AgentRoute

// 统一的出站接口
ChannelOutboundAdapter {
  sendText(target, text, options)
  sendMedia(target, media, options)
  sendPoll(target, poll, options)
}
```

### 2. 平台特定优化

每个平台保留其独特功能：

- **WhatsApp**: E.164 ↔ JID 转换，PTT 语音
- **Slack**: Block Kit，线程化，工作流
- **Telegram**: 流式更新，论坛话题，媒体组
- **Discord**: 斜杠命令，角色路由，交互组件

### 3. 防抖和去重

所有平台都实现了智能防抖：

```typescript
// 通用防抖逻辑
createInboundDebouncer({
  debounceMs: 0,  // 可配置
  onFlush: (messages) => {
    // 合并连续消息
    const combined = messages.map(m => m.text).join("\n");
    processMessage({ ...messages[0], text: combined });
  }
});
```

**去重策略对比**：
- WhatsApp: LRU Cache (1000 条，60 秒 TTL)
- Slack: Set (基于 message_ts)
- Telegram: Update ID tracking
- Discord: Message ID Set

### 4. 错误恢复

所有平台都实现了健壮的重连机制：

```typescript
// 通用重连模式
let reconnectAttempts = 0;
const maxAttempts = 12;

while (reconnectAttempts < maxAttempts) {
  try {
    await connectToPlatform();
    reconnectAttempts = 0;  // 重置
  } catch (error) {
    reconnectAttempts++;
    const backoff = Math.min(2000 * Math.pow(2, reconnectAttempts - 1), 30000);
    await sleep(backoff);
  }
}
```

---

## 总结

### 架构优势

OpenClaw 的多平台集成展示了优秀的软件设计：

1. **统一抽象**：相同的 Agent Loop 接口
2. **平台特性保留**：每个平台的独特功能得以保留
3. **配置灵活**：从全局到细粒度的多层配置
4. **健壮性**：自动重连、去重、防抖、错误恢复
5. **可扩展**：插件式架构，易于添加新平台

### 平台选择建议总结

| 场景 | 首选 | 理由 |
|------|------|------|
| **快速原型** | Telegram | 设置最简单，功能丰富 |
| **企业部署** | Slack | 企业功能，合规性 |
| **游戏社区** | Discord | 原生功能，交互丰富 |
| **客户支持** | Telegram | Bot 生态成熟 |
| **个人助手** | Telegram | 功能最完整 |
| **开发测试** | Telegram | 无限制，快速迭代 |

### 技术栈总结

| 平台 | 框架 | 协议 | 官方支持 | 推荐度 |
|------|------|------|----------|--------|
| **Telegram** | grammY | HTTPS | ✅ 官方 | ⭐⭐⭐⭐⭐ |
| **Discord** | Carbon | WebSocket | ✅ 官方 | ⭐⭐⭐⭐ |
| **Slack** | Bolt | WS/HTTP | ✅ 官方 | ⭐⭐⭐⭐ |
| **WhatsApp** | Baileys | WebSocket | ⚠️ 非官方 | ⭐⭐⭐ |

---

**文档版本**: 1.0
**最后更新**: 2026-02-19
**涵盖平台**: WhatsApp, Slack, Telegram, Discord
**分析范围**: OpenClaw 消息平台集成架构完整对比
