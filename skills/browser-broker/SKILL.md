# browser-broker

## Description
全局浏览器接入与调度 skill。所有需要使用 Playwright / Chromium 的 agent 都必须通过 Browser Broker 申请、占用、释放和切换浏览器资源；禁止业务代码直接创建 Playwright 实例。

## Implementation location
```text
runtime/browser-broker/     ← 执行实现（broker.py、broker_client.py、…）
skills/browser-broker/      ← 本文件：契约、使用边界、接入说明（只读参考）
docs/browser-broker-usage.md
docs/browser-broker-dev-brief.md
docs/multi-agent-browser-roadmap.md
```

启动 broker：
```bash
python3 runtime/browser-broker/broker.py
```

## When to use this skill
当你需要：
- 打开网页、采集页面、执行浏览器自动化
- 申请或释放浏览器资源
- 切换 hidden / visible 模式
- 处理登录、验证、滑块、安全验证等人工介入场景
- 获取截图、页面状态、浏览器占用状态

## Core rules
1. 浏览器只能由 Browser Broker 持有和管理。
2. 业务代码、主 Agent、Runner 不得直接 new Playwright。
3. 所有浏览器任务必须先进入 broker 队列，再由 broker/worker 执行。
4. 验证 / 登录 / 滑块 / 安全验证触发后必须切到 visible，并进入全局暂停语义。
5. 恢复后必须保持同一个 user_data_dir，重新以 hidden 模式恢复执行。

## Required contract
新 agent 启动后，应默认理解以下契约：
- `status`
- `collect_homepage`
- `fill_publish_dates`
- `resume_after_human`
- `collect_sound_page` *(找版机器人专用；broker 侧实现待完成)*
- `register_pending` *(多 agent 等待队列注册)*
- `unregister_pending` *(取消等待)*

补充说明：
- `acquire_browser` / `release_browser` / `switch_to_visible` 是 broker 内部的资源控制原语，主要给执行层与扩展实现使用。
- 业务层最常用的是 `collect_homepage`、`fill_publish_dates` 和 `resume_after_human`。
- `collect_sound_page` 与 `collect_homepage` 共享相同的 login/verification PAUSED 语义，`resume_after_human` 可直接复用。

## Multi-agent arbitration rules

### 核心原则
Broker 始终是唯一浏览器持有者。同一时刻只有一个 agent 能持有浏览器。

### 状态与仲裁行为

| 状态 | 其他 agent 行为 | retryable |
|------|----------------|-----------|
| `idle` | 任何 agent 可直接 acquire | — |
| `busy` | 所有 acquire / collect 请求被拒绝，返回错误 | `true` |
| `paused_for_human` | 所有 acquire / collect 请求被拒绝，返回错误 | `false` |

- **busy**：owner 任务完成后会自动释放。等待的 agent 可调用 `register_pending` 进入 FIFO 队列，收到 `retryable=true` 后轮询 `status` 并重试。
- **paused_for_human**：需要人工操作（扫码/验证）才能解除。不接受 `register_pending`。等待方必须等到 `resume_after_human` 成功后再重试。

### 越权保护
- `release_browser` / `switch_to_visible` / `resume_after_human` 只允许当前 owner（`owner_agent` + `owner_task_id`）调用。
- 跨 agent 操作一律被拒绝，错误内容包含当前 owner 信息。

### 等待队列
- `register_pending(owner_agent, owner_task_id)` — 注册等待意图（幂等）
- `unregister_pending(owner_agent, owner_task_id)` — 取消等待
- `status` 响应中的 `pending` 字段暴露当前等待列表（FIFO 顺序）
- 队列是建议性的：broker 不会主动通知等待方，由客户端轮询 `status` 并重试

### 查看当前状态
```bash
echo '{"action":"status","owner_agent":null,"owner_task_id":null}' | \
  python3 -c "
import socket, sys, json
s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.connect('runtime/browser-broker/runtime/broker.sock')
s.sendall(sys.stdin.buffer.read() + b'\n')
buf = b''
while b'\n' not in buf: buf += s.recv(4096)
print(json.dumps(json.loads(buf.split(b'\n')[0]), indent=2, ensure_ascii=False))
"
```

响应关键字段：
- `state` — `idle` / `busy` / `paused_for_human`
- `mode` — `hidden` / `visible`
- `owner_agent` / `owner_task_id` — 当前持有者
- `pending` — 等待队列 `[{owner_agent, owner_task_id}, ...]`
- `data.paused_file_exists` — PAUSED 哨兵文件是否存在

## collect_sound_page

| 字段 | 说明 |
|------|------|
| action | `"collect_sound_page"` |
| owner_agent | 调用方标识 |
| owner_task_id | 任务 run_id（与 resume_after_human 的 owner_task_id 必须一致）|
| url | 抖音声音页面 URL |
| scan_depth | `"shallow"` / `"medium"` / `"deep"` |
| known_video_ids | 已知 video_id 列表（优化滚动深度，非强制过滤）|

成功响应 `data` 字段：

| 字段 | 说明 |
|------|------|
| sound_id | 声音 ID |
| sound_name | 声音名称 |
| author_name | 声音作者 |
| use_count | 使用该声音的视频总数 |
| videos | 采集到的视频列表（见下）|
| page_state | `"healthy"` / `"login_required"` / `"verification_required"` |
| screenshot_path | page_state != healthy 时有值 |

每条 video：`video_id`, `video_title`, `like_count`, `video_url`, `author_id`, `author_name`

## State model
Browser Broker 至少维护：
- `owner_agent`
- `owner_task_id`
- `mode`: `hidden | visible`
- `state`: `idle | busy | paused_for_human`
- `pending`: 等待队列（list of `{owner_agent, owner_task_id}`）

## Minimal usage flow
1. 通过 broker 请求页面采集或详情补强
2. 若被拒绝（`retryable=true`）：调用 `register_pending` 后轮询 `status` 等 `state==idle`，再重试
3. 若被拒绝（`retryable=false`）：等待人工干预（`paused_for_human` 解除）
4. 命中验证则切 visible（broker 自动处理）
5. 人工处理完成后恢复（`resume_after_human`）
6. 释放浏览器资源（broker 在操作完成后自动释放）

## Notes
- 这是全局基础 skill，不属于某个单一业务域。
- 执行实现代码位于 `runtime/browser-broker/`；本 skill 只定义对外使用契约与边界。
- 不要在业务 skill 中直接引用 `runtime/browser-broker/` 的内部实现细节；通过 `BrokerCollector` 接入。
