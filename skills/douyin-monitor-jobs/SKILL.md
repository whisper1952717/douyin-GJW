---
name: douyin-monitor-jobs
description: Manage task-scoped Douyin monitoring jobs for each Feishu group. Use when users need to query, add, pause, resume, remove, or update the accounts currently being tracked by their own group, including listing current jobs and changing monitored homepage URLs or task status without crossing group boundaries.
---

# Douyin Monitor Jobs

Manage the set of Douyin homepage monitoring jobs that belong to the current group/task scope.

Use this skill to:
- list the accounts currently tracked by a group
- add a new tracked Douyin homepage
- pause or resume an existing tracked job
- remove a tracked job
- update a tracked homepage URL

When handling natural-language requests, read `INTENTS.md` to map user phrasing to actions. Use `scripts/render_jobs_text.py` when you want a concise Chinese summary for group replies.

## Scope Rules

- Always scope operations by `group_key + target_peer_id`
- Never list or modify jobs from other groups
- Treat `(homepage_url, group_key, target_peer_id)` as the natural task identity

## Storage

Reuse the same SQLite database used by `douyin-like-monitor`:
- `skills/douyin-like-monitor/data/monitor.db`

Primary table:
- `monitor_job`

Optional related tables:
- `job_video_state`
- `job_event_history`

## Recommended Actions

### List jobs
Return current jobs for one group, including:
- `job_id`
- `homepage_url`
- `account_id`
- `account_name`
- `status`
- `last_run_at`
- `last_successful_run_at`
- `last_full_rebuild_at`

### Add job
When adding a job:
1. normalize homepage URL
2. create job if not exists for this group
3. default to `status=healthy`
4. do not create duplicate jobs for the same `(homepage_url, group_key, target_peer_id)`

### Pause / Resume job
Suggested status values:
- `paused`
- `healthy`

When paused:
- the scheduler/executor should skip normal collection for that job
- historical state remains intact

### Remove job
When removing a job:
- remove the `monitor_job` row for that group-scoped task
- optionally cascade delete `job_video_state` and `job_event_history`
- never remove jobs belonging to another group

### Update job
Allow safe updates such as:
- homepage URL
- account display fields if needed
- status changes

## Implementation Guidance

Keep the implementation small and deterministic:
- prefer a single Python script under `scripts/`
- return JSON for all actions
- validate required arguments strictly
- normalize URLs before persistence

## Output Pattern

Return JSON objects like:

```json
{
  "ok": true,
  "action": "list",
  "jobs": []
}
```

or

```json
{
  "ok": true,
  "action": "add",
  "job": { ... }
}
```

For errors:

```json
{
  "ok": false,
  "error": "..."
}
```
