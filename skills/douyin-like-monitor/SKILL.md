---
name: douyin-like-monitor
description: Use when you need to monitor Douyin homepage videos within a rolling 7-day window, track homepage like growth with low-risk collection, and output Chinese JSON event arrays. Trigger when input contains homepage_url, group_key, target_peer_id, run_id, and optional threshold_profile, and when event types include 首次爆款、增长爆款、登录页需扫码、验证页需人工、恢复、异常.
metadata:
  openclaw:
    emoji: 🎵
    requires:
      bins: []
---

# Douyin Like Monitor

Parse monitoring input, maintain a task-scoped rolling 7-day video window for a Douyin homepage, collect homepage like counts with low-risk actions, evaluate event rules, and return a Chinese JSON event array via the `run()` function.

**Execution model**: Tasks flow through a unified queue (`monitor_task_queue` table). Cron enqueues `incremental_update` tasks; new accounts get `full_rebuild` immediately. The queue worker consumes one task at a time, executes it, and reports results.

Do not send Feishu messages directly. Do not schedule workflows.
Do not define Playwright / browser startup details here; browser access must go through the global `browser-broker` skill and the browser-broker implementation layer.

## Input Contract

Required fields:
- `homepage_url`
- `group_key`
- `target_peer_id`
- `run_id`

Optional fields:
- `threshold_profile` (default: `default`)
- `forced_run_mode` — override auto-detection of run mode: `"full_rebuild"` or `"incremental_update"`
- `resume_checkpoint` — integer checkpoint (from previous `full_rebuild`'s `checkpoint` field); causes `fill_publish_dates` to skip already-processed candidates

## Output Contract

`run()` returns a `dict` (not a raw list) for checkpoint support:
```json
{
  "events": [Event, ...],
  "checkpoint": 5
}
```

Each **Event** object:
- `事件类型`
- `账号名`
- `账号ID`
- `群名`
- `视频ID`
- `视频标题`
- `视频链接`
- `当前点赞数`
- `上次检测点赞`
- `上次汇报点赞`
- `新增点赞`
- `增长比例`
- `触发原因`
- `触发时间`
- `作者名`  *(新增)*
- `发布时间` *(新增，Unix timestamp 或 null)*
- `screenshot_path` *(仅登录页需扫码/验证页需人工事件有值)*

The `checkpoint` field in the result is the number of candidate videos processed by `fill_publish_dates`. The queue worker saves this to `monitor_task_queue.checkpoint` so that if the worker crashes or is paused, the next run can resume from where it left off by passing it as `resume_checkpoint`.

## Monitoring Scope

- A task is identified by `homepage_url + group_key + target_peer_id`
- Each task maintains its own rolling 7-day video window
- Video state and emitted events are isolated per task
- Events from account A must only go to the task/group bound to account A; do not mix groups
- Task queue keys include `group_key` and `target_peer_id`, so the same account monitored in different groups gets separate queue entries

## Queue-Based Execution

Tasks are stored in `monitor_task_queue` with these statuses:
- `PENDING` — waiting to be claimed
- `RUNNING` — currently executing (set by worker)
- `DONE` — completed successfully
- `FAILED` — failed after retries
- `PAUSED_FOR_HUMAN` — blocked on login/verification; requires human intervention
- `RESUMING` — transitioning from PAUSED_FOR_HUMAN back to RUNNING

Priority values: `full_rebuild` = 200 (highest), `incremental_update` = 50 (normal).

## Run Modes

### 1. `full_rebuild`
Use when:
- this task runs for the first time
- the task has not completed successfully for more than 7 days
- cached 7-day window is missing / invalid
- or when `forced_run_mode="full_rebuild"` is passed

Actions:
1. Normalize `homepage_url`
2. Open homepage and detect page status
3. If login / verification / exception is detected, emit the corresponding event immediately and return
4. If page is healthy, rebuild the rolling 7-day window
5. Collect candidate videos from homepage, resolve publish dates via detail pages (`fill_publish_dates`)
6. If `resume_checkpoint > 0`, skip the first N candidates (resume from saved checkpoint)
7. Prefer homepage-level date signals; if unavailable, use limited detail-page checks to locate the 7-day boundary
8. Store only videos within the last 7 days
9. Drop videos older than 7 days from realtime state
10. Evaluate event rules on newly collected/updated videos
11. Persist new state; return `{"events": [...], "checkpoint": N}` where N is the number of candidates processed

### 2. `incremental_update`
Use when:
- the task has a valid rolling 7-day window
- the last successful run is within 7 days
- or when `forced_run_mode="incremental_update"` is passed

Actions:
1. Normalize `homepage_url`
2. Open homepage and detect page status
3. If login / verification / exception is detected, emit the corresponding event immediately and return
4. If page is healthy, scan from the top of the homepage and incrementally update:
   - newly appeared high-priority videos
   - known in-window videos that are easy to refresh from homepage cards
5. Do not rebuild the full 7-day boundary on normal hourly runs
6. Drop videos older than 7 days from realtime state
7. Evaluate event rules on updated videos
8. Persist new state; `checkpoint` in result is always 0 for incremental_update

## Video State (per task)

For each in-window video, persist at least:
- `video_id`
- `title`
- `video_url`
- `publish_ts`
- `publish_time_text`
- `current_likes`
- `last_detected_likes`
- `last_reported_likes`
- `last_detected_at`
- `last_reported_at`
- `priority` (`high | medium | low`)

Notes:
- Do not keep realtime state for videos older than 7 days
- Event history (`job_event_history`) retains `author_name`, `author_id`, and `publish_ts` for audit

## Event Rules

### `首次爆款`
Emit when all conditions hold:
- video is inside the rolling 7-day window
- `当前点赞数 >= 100`
- `上次汇报点赞 == 0`

After emit:
- set `上次汇报点赞 = 当前点赞数`
- update `last_reported_at`

### `增长爆款`
Emit when all conditions hold:
- video is inside the rolling 7-day window
- `上次汇报点赞 > 0`
- `当前点赞数 - 上次汇报点赞 >= 100`

After emit:
- set `上次汇报点赞 = 当前点赞数`
- update `last_reported_at`

### `登录页需扫码`
When detected:
- capture screenshot
- emit status event with `screenshot_path`
- task status → `login_required`
- queue status → `PAUSED_FOR_HUMAN`
- do not overwrite the last healthy window with empty/partial video results

### `验证页需人工`
When detected:
- capture screenshot
- emit status event with `screenshot_path`
- task status → `verification_required`
- queue status → `PAUSED_FOR_HUMAN`

### `恢复`
Emit when:
- previous task status was `login_required`, `verification_required`, or `exception`
- current run returns to a healthy collectible homepage

### `异常`
Emit when collection cannot continue reliably, for example:
- homepage open timeout
- broken page structure
- missing required fields after reasonable retries
- failure to rebuild the 7-day window

## Priority Rules

Priority controls collection effort, not message thresholds.

### `high`
Typical cases:
- newly seen videos near first report threshold
- videos close to the next `+100` report boundary
- recently growing videos worth refreshing every run

### `medium`
Typical cases:
- in-window videos with some growth but still far from the next report boundary

### `low`
Typical cases:
- low-like new videos
- in-window videos with weak recent growth

## Scanning Stop Conditions

For normal hourly incremental runs, stop when any condition is met:
- all high-priority videos for this run have been refreshed
- encountered enough consecutive known old videos
- clearly reached videos older than 7 days
- several scroll attempts produced no new cards
- run budget / time budget is exhausted

## Workflow (queue-based)

1. Cron trigger (整点): enqueue `incremental_update` for all healthy jobs; assign `full_rebuild` for jobs where 7+ days since last rebuild
2. New account added: immediately enqueue `full_rebuild` (priority 200)
3. Worker claims one PENDING task at a time (ordered by priority desc, created_at asc)
4. Worker calls `DouyinLikeMonitor.run(payload)` with `forced_run_mode` and `resume_checkpoint`
5. Worker dispatches alert events (登录页/验证页/异常) via `dispatch_login_event.py`
6. Worker dispatches normal events (首次爆款/增长爆款) via `dispatch_normal_event.py`
7. Worker marks task DONE / FAILED / PAUSED_FOR_HUMAN
8. For PAUSED_FOR_HUMAN: separate polling loop (`--poll` mode) detects when homepage becomes healthy and resets task to PENDING

## Event Priority

Highest to lowest:
1. `登录页需扫码` / `验证页需人工`
2. `异常`
3. `恢复`
4. `首次爆款`
5. `增长爆款`

## Threshold Profile

`default`:
- 首次爆款: 当前点赞 >= 100 and 上次汇报点赞 == 0
- 增长爆款: 当前点赞 - 上次汇报点赞 >= 100
- Window retention: only keep videos whose publish time is within the last 7 days in realtime state
- Normal hourly runs: use incremental homepage updates instead of full boundary rebuilds
