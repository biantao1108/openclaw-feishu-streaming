# openclaw-feishu-streaming

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