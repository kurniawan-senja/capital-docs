---
id: google-sheets
title: 5. Google Sheets Structure
---

# 5. Google Sheets Structure

The Layer Control sheet is the central control plane. All job orchestration, status tracking, render queuing, and logging occurs here.

:::info
**Sheet ID:** 1BNGKNamOkjNvnR7DyDiqS0G3-qWzVdV6BlyzJ_RrLFk
:::

## 5.1 Read_template Tab (gid=1093980937)

The job queue. One row per job. Operator creates rows here to trigger automation.

### Google Sheets — Read_template Tab

![Google Sheets Read Template Tab](/img/screenshots/figure-018.png)

| Column | Type | Description |
| --- | --- | --- |
| `job_id` | String | Unique identifier for the job (e.g. `job_001`) |
| `template_path` | String | Full Windows path to the `.aep` template file |
| `rw_path` | String | Full Windows path to the RW `.aep` file (optional) |
| `status` | String | `READY / PROCESSING / RENDERING_RW / RENDERING_EXPORT / FINISHED / FAILED` |
| `eror_step` | String | Error step identifier if status = FAILED |
| `eror_message` | String | Error message detail if status = FAILED |

## 5.2 Queue Tab (gid=956294243)

The render queue. One row per composition per job. Populated automatically by Workflow 1.

### Google Sheets — Queue Tab

![Google Sheets Queue Tab](/img/screenshots/figure-019.png)

| Column | Type | Description |
| --- | --- | --- |
| `render_id` | String | Unique render identifier (e.g. `render_001`) |
| `job_id` | String | Parent `job_id` from Read_template |
| `template_path` | String | Path to `.aep` file for this render |
| `category` | String | Comp token 1 (e.g. OTH, CRP) |
| `campaign` | String | Comp token 2 — campaign slug |
| `instrument` | String | Comp token 3 — financial instrument |
| `variant` | String | Comp token 4 — version (v1, v2...) |
| `regulation` | String | Comp token 5 — regulatory body code |
| `rw` | String | Comp token 6 — RW index |
| `language` | String | Comp token 7 — language code (EN, DE, AR...) |
| `creative_type` | String | Comp token 8 — Video / Banner etc. |
| `creative_size` | String | Comp token 9 — dimensions (1080x1920) |
| `creative_length` | String | Comp token 10 — duration (30sec) |
| `source` | String | Comp token 11 — source identifier |
| `ACPM` | String | Comp token 12 — ACPM value |
| `render` | String | `READY / RENDERING / FINISHED / FATAL` |
| `retry_count` | Number | Number of render attempts (max 3) |
| `render_start` | String | Timestamp when render started |
| `render_end` | String | Timestamp when render completed |
| `duration_sec` | Number | Render duration in seconds |
| `file_link` | String | Google Drive `webContentLink` to uploaded `.mp4` |

## 5.3 Logs Tab (gid=395354427)

Append-only event log. Every status transition and error is recorded here.

### Google Sheets — Log Tab

![Google Sheets Log Tab](/img/screenshots/figure-020.png)

| Column | Type | Description |
| --- | --- | --- |
| `timestamp` | String | ISO timestamp of the event |
| `job_id` | String | Associated `job_id` |
| `event` | String | `FINISHED / FAILED / RENDER_FINISHED / RENDER_FATAL` |
| `detail` | String | Human-readable description of the event |
