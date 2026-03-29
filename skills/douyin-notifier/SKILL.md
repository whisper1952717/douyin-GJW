---
name: douyin-notifier
description: Sends Feishu group notifications with images for login/verification events. Use when you need to send screenshot + text messages to a Feishu group for douyin-monitor events. In the current single-bot setup, orchestrator invokes local notification logic directly; spawned/sub-agent flows are optional compatibility paths.
metadata:
  openclaw:
    emoji: 🔔
    requires:
      bins: []
---

# Douyin Notifier

Sends login QR code and verification events with screenshot to Feishu group.

## Input Contract

As a notification execution layer, it is primarily called directly by orchestrator in-process / locally via script import or stdin JSON.
For events with `screenshot_path` and `事件类型` in `["登录页需扫码", "验证页需人工"]`, sends image + text to `target_peer_id`.

If only `群名` / `group_key` is present, orchestrator-side dispatch script resolves the target group and then invokes local notification logic directly.

## Environment Variables

- `FEISHU_APP_ID` - Feishu app ID (auto-configured by OpenClaw)
- `FEISHU_APP_SECRET` - Feishu app secret (auto-configured by OpenClaw)

These are auto-configured by OpenClaw's Feishu integration - do not set manually.

## Feishu API Flow

1. Get tenant_access_token via `https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal`
2. Upload image via `https://open.feishu.cn/open-apis/im/v1/images` (multipart, image_type=message)
3. Send image message via `https://open.feishu.cn/open-apis/im/v1/messages?receive_id_type=chat_id`
4. Send text message via same endpoint

## Message Format

```
🔔 事件类型：登录页需扫码
👤 账号名：{账号名}
📋 账号ID：{账号ID}
🆔 群组：{群名}
⏰ 触发时间：{触发时间}

📝 触发原因：{触发原因}

⚠️ 请群成员扫码完成登录
```

## Output

JSON array of send results to stdout.
