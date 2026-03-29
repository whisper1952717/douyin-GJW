---
name: login-verification-handler
description: Use when Douyin collection encounters login QR pages, verification pages, recovery transitions, or monitoring exceptions and needs standardized Chinese event JSON plus task status updates. Outputs 登录页需扫码 and 验证页需人工 events with screenshot_path for image-enabled notification chain.
metadata:
  openclaw:
    emoji: 🔐
    requires:
      bins: []
---

# Login Verification Handler

Generate standardized task-scoped status events for Douyin monitoring.
No manual login or verification actions.
Browser capture / visible-mode handling belongs to the global `browser-broker` skill and implementation layer; this skill only defines the status-event contract.

## Contract

- `emit_login_required(run_id, homepage_url, group_key, target_peer_id, screenshot_path="")` → 事件类型: 登录页需扫码
- `emit_verification_required(run_id, homepage_url, group_key, target_peer_id, screenshot_path="")` → 事件类型: 验证页需人工
- `emit_recovering(run_id, homepage_url, group_key, target_peer_id)` → 事件类型: 恢复
- `emit_recovered(run_id, homepage_url, group_key, target_peer_id)` → 事件类型: 恢复
- `emit_exception(run_id, homepage_url, group_key, target_peer_id, error_summary)` → 事件类型: 异常

## Scope Rules

- Status is task-scoped: `homepage_url + group_key + target_peer_id`
- Status events must follow the same task/group isolation as normal video events
- Do not mix login/verification/exception events across tasks or groups

## Output Fields

All events include:
- `事件类型`
- `账号名`
- `账号ID`
- `群名`
- `视频ID`(空)
- `视频标题`(空)
- `当前点赞数`(0)
- `上次检测点赞`(0)
- `上次汇报点赞`(0)
- `新增点赞`(0)
- `增长比例`(0.0)
- `触发原因`
- `触发时间`

Login / Verification events additionally include:
- `screenshot_path`

## Status Values

- `login_required`
- `verification_required`
- `recovering`
- `healthy`
- `exception`

## Handling Rules

### 登录页需扫码
When the homepage shows a clear login gate, such as:
- `扫码登录`
- `登录后查看更多作品`
- `立即登录`

Actions:
1. wait briefly for the QR/login visual to render stably
2. capture screenshot
3. emit `登录页需扫码`
4. do not overwrite the last healthy 7-day window with partial or empty collection results

### 验证页需人工
When the homepage shows verification or risk-control UI, such as:
- slider verification
- captcha / 安全验证
- abnormal access verification

Actions:
1. wait briefly for the verification visual to render stably
2. capture screenshot
3. emit `验证页需人工`
4. do not overwrite the last healthy 7-day window with partial or empty collection results

### 异常
When collection cannot continue reliably, for example:
- page open timeout
- broken DOM / missing required homepage signals
- failure during 7-day boundary rebuild

Actions:
1. emit `异常`
2. store task status as `exception`
3. keep the last healthy window untouched until a later successful recovery

### 恢复
When the previous task status was one of:
- `login_required`
- `verification_required`
- `exception`

and the current run returns to a healthy collectible homepage:
1. emit `恢复`
2. switch task status back to `healthy`
3. resume normal incremental monitoring or full rebuild as needed
