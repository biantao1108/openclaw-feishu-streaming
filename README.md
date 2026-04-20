# openclaw-feishu-streaming

**Author:** biantao@outlook.com

Enhanced Feishu/Lark channel plugin for OpenClaw with improved CardKit streaming output.

## Overview

This plugin is a fork of the official `@openclaw/feishu` (v2026.3.13) with key improvements to the streaming card implementation, providing a smoother "typewriter" effect for AI responses in Feishu.

## Improvements over Official Plugin

### 1. Immediate Trigger Update Mechanism (`streaming-card.ts`)

**Official behavior**: All updates were serialized through a Promise chain with a fixed 100ms throttle — every update had to wait for the previous one to complete.

**Improved behavior**: When the throttle window has passed, the update fires immediately without queue serialization overhead. New content arriving mid-flight triggers another update immediately. This matches the RTT natural backpressure pattern from AstrBot.

```typescript
// When elapsed >= throttle window: send immediately (fire-and-forget)
if (elapsed >= this.updateThrottleMs) {
  this.state.currentText = textToSend;
  this.updateCardContent(textToSend, ...); // no await
  return;
}
```

### 2. Delta Mode for Partial Updates (`reply-dispatcher.ts`)

**Official behavior**: `onPartialReply` used `mode: "snapshot"` which performed text merging on each update.

**Improved behavior**: Changed to `mode: "delta"` which appends new text incrementally, matching the AstrBot pattern where text is accumulated before each card update.

### 3. Fallback Streaming Behavior (`streaming-card.ts` close method)

When `streaming.start()` or card updates fail, the implementation falls back gracefully to non-streaming delivery, preserving the `FeishuStreamingSession` side effects.

## Credits

### AstrBot PR #5777

The core streaming logic improvements are inspired by [AstrBot PR #5777](https://github.com/AstrBotDevs/AstrBot/pull/5777) by **steipete**, which introduced CardKit streaming output support for the Lark/Feishu adapter.

Key concepts borrowed:
- **RTT natural backpressure**: Fire updates without awaiting for smoother streaming
- **Event-driven update loop**: Decoupled sender that reacts to text changes
- **Break chain handling**: Ignore `break` chunks in streaming mode (Feishu cards don't support segments)
- **Fallback buffering**: Buffer all text then send once if CardKit streaming fails

Reference: [AstrBot](https://github.com/AstrBotDevs/AstrBot) — enterprise AI assistant with multi-agent support.

## Installation

### From GitHub

```bash
npm install -g github:<your-username>/openclaw-feishu-streaming
```

### From Source

```bash
git clone https://github.com/<your-username>/openclaw-feishu-streaming.git
cd openclaw-feishu-streaming
npm install
```

Then configure in OpenClaw to use the local plugin path.

## Configuration

The plugin uses the same configuration schema as `@openclaw/feishu`. Key streaming-related options:

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `streaming` | `boolean` | `true` | Enable/disable CardKit streaming output |
| `renderMode` | `auto\|raw\|card` | `auto` | Message rendering mode |

## Feishu Permission Configuration

### Quick Import (paste directly into Feishu Open Platform)

```
cardkit:card:read
cardkit:card:write
cardkit:template:read
contact:contact.base:readonly
im:chat.members:bot_access
im:chat.members:read
im:chat:read
im:message.group_at_msg:readonly
im:message.group_msg
im:message.p2p_msg:readonly
im:message:readonly
im:message:send_as_bot
im:message:update
im:message.reactions:write_only
optical_char_recognition:image
speech_to_text:speech
contact:contact.base:readonly
```

### Permission Descriptions

| Permission | Scope | Description |
|---|---|---|
| `cardkit:card:read` | tenant | 读取消息卡片内容，用于卡片交互 |
| `cardkit:card:write` | tenant | 发送/更新消息卡片，机器人卡片回复必需 |
| `cardkit:template:read` | tenant | 读取卡片模板，复用卡片模板使用 |
| `contact:contact.base:readonly` | tenant | 获取用户基础信息，用于识别对话用户 |
| `contact:contact.base:readonly` | user | 用户维度联系人只读，辅助用户身份识别 |
| `im:chat.members:bot_access` | tenant | 机器人获取群成员权限，群聊必需 |
| `im:chat.members:read` | tenant | 读取会话成员信息，区分群内用户 |
| `im:chat:read` | tenant | 读取会话信息，识别单聊/群聊场景 |
| `im:message.group_at_msg:readonly` | tenant | 读取@机器人消息，对话触发必需 |
| `im:message.group_msg` | tenant | 接收群聊消息，机器人群聊对话必需 |
| `im:message.p2p_msg:readonly` | tenant | 接收单聊消息，机器人私聊对话必需 |
| `im:message:readonly` | tenant | 通用消息只读权限，兜底接收所有消息 |
| `im:message:send_as_bot` | tenant | 机器人以自身身份发送消息，核心必选 |
| `im:message:update` | tenant | 编辑/更新已发消息，用于流式回复、消息修正 |
| `im:message.reactions:write_only` | tenant | 机器人给消息添加表情互动 |
| `optical_char_recognition:image` | tenant | 图片OCR，识别用户发送图片中的文字 |
| `speech_to_text:speech` | tenant | 语音转文字，处理用户语音消息 |

---

## Architecture

```
bot.ts → createFeishuReplyDispatcher() → FeishuStreamingSession
                                      ↓
                             CardKit Streaming API
                             (create → update → close)
```

Key files:
- `src/streaming-card.ts` — `FeishuStreamingSession` class managing CardKit streaming lifecycle
- `src/reply-dispatcher.ts` — Orchestrates replies with typing indicators and streaming

## License

Same as `@openclaw/feishu` — MIT.