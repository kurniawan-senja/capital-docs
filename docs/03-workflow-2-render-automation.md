---
id: workflow-2
title: 3. Workflow 2 — Capital Render Automation v2
---

# 3. Workflow 2 — Capital Render Automation v2

## 3.1 Purpose & Scope

This workflow handles the actual rendering of each composition queued by Workflow 1. It reads render rows from the Queue tab, reconstructs the composition name, builds a job file, launches AfterFX to render, uploads output to Google Drive, tracks timing, and manages retries and Slack notifications.

:::info
This workflow runs per `render_id`. Multiple renders can be triggered simultaneously if multiple Queue rows are set to `READY` at the same time.
:::

## 3.2 Workflow Summary

| Property | Value |
| --- | --- |
| **Trigger** | Google Sheets polling — Queue tab, every minute |
| **Condition** | `render` column = `READY` |
| **Column watched** | `render` |
| **Output** | Rendered `.mp4` uploaded to Google Drive |
| **Retry logic** | Max 3 retries — requeue to `READY` then `FATAL` |
| **Notifications** | Slack webhook on DONE and FATAL |

### System Architecture — n8n Canvas Overview - Render Automation V2

![Workflow 2 Full Canvas (with Notification Nodes)](/img/screenshots/figure-001.png)

:::info Screencast Workflow 2 Full Demo
Set a Queue row render = READY, watch n8n trigger, AfterFX render, file upload to Drive, and status update to FINISHED. Duration: 5-8 min.
Link: https://streamable.com/1wj5fa
:::

## 3.3 Node Summary Table

| Node | Type | Purpose | Output |
| --- | --- | --- | --- |
| **Read Render Update** | Google Sheets Trigger | Polls Queue tab every minute watching render column | Changed rows |
| **If1** | IF Node | Routes rows with render = READY only | TRUE / FALSE |
| **Loop Over Items** | Split In Batches | Processes one `render_id` at a time | Single render item |
| **Read Queue Data** | Google Sheets | Fetches full Queue row by job_id + render_id | Complete render row |
| **Parse Job Format** | Code | Reconstructs comp name, builds nexrender job JSON | `nexrenderJob` object |
| **Encode Base64** | Code | Base64-encodes the job JSON for disk write | `jobB64` string |
| **Write Job to Disk** | Execute Command | Writes job JSON via PowerShell | File on disk |
| **Update Status RENDERING** | Google Sheets | Sets render = RENDERING before AE starts | Confirmation |
| **Render Start Timestamp** | Code | Captures ISO timestamp of render start | `render_start` string |
| **Write Start Timestamp** | Google Sheets | Writes `render_start` to Queue row | Confirmation |
| **Run Render** | Execute Command | Launches AfterFX with `Render.jsx` | stdout / exitCode |
| **If Render OK** | IF Node | Checks exitCode = 0 for success | TRUE / FALSE |
| **Read/Write Files from Disk**| Read/Write File | Reads rendered `.mp4` from output path | Binary file data |
| **Render End Timestamp** | Code | Captures end time and calculates `duration_sec` | `render_end` + duration |
| **Write End Timestamp** | Google Sheets | Writes `render_end` + `duration_sec` to Queue row | Confirmation |
| **Upload file** | Google Drive | Uploads `.mp4` to Render Output folder | `webContentLink` |
| **Update Job Link** | Google Sheets | Sets render = FINISHED + file_link | Confirmation |
| **Log Render FINISHED** | Google Sheets | Appends RENDER_FINISHED to Logs tab | Confirmation |
| **Build Notify Payload** | Code | Builds Slack success message | Slack payload JSON |
| **Notify Render Done** | HTTP Request | POSTs to Slack webhook on success | HTTP 200 |
| **Retry Counter** | Code | Reads retry_count, increments or marks FATAL | action: REQUEUE / FATAL |
| **If Requeue or Fatal** | IF Node | Routes REQUEUE vs FATAL | TRUE / FALSE |
| **Requeue to READY** | Google Sheets | Sets render = READY + increments retry_count | Confirmation |
| **Update Render FATAL** | Google Sheets | Sets render = FATAL + error_message | Confirmation |
| **Log Render FATAL** | Google Sheets | Appends RENDER_FATAL to Logs tab | Confirmation |
| **Build Fatal Alert Payload** | Code | Builds multi-channel alert message for FATAL | JSON payload |
| **Alert Systems (Slack/TG/Mail)** | HTTP / Mail | POSTs to Slack, Telegram, and Gmail webhooks on FATAL | HTTP 200 |
| **End Update** | Set | Terminal node for non-READY rows | No output |
| **Error Trigger** | Error Trigger | Global listener catching unhandled n8n failures | Exception data |
| **Build System Error Payload** | Code | Formats structured multi-channel alert for unhandled nodes | JSON payload |

## 3.4 Node Detail — Key Nodes

### 3.4.1 Parse Job Format

Reconstructs the full 12-token composition name from Queue row data and builds the nexrender job object with template source, composition name, output module, and post-render copy action.

### Parse Job Format — Code Panel

![Parse Job Format](/img/screenshots/figure-010.png)

### 3.4.2 Run Render

Launches AfterFX.exe in script mode passing three global variables: `templatePath`, `compName`, and `outputPath`. AfterFX executes `Render.jsx` which opens the project, finds the comp, renders, saves, and quits.

```bash
AfterFX.exe -s "$.global.templatePath='...'; $.global.compName='...'; $.global.outputPath='...'; $.evalFile('Render.jsx');"
```

### Run Render — Command Field

![Run Render](/img/screenshots/figure-011.png)

:::warning
AfterFX.exe must exit with code 0 for the workflow to continue to upload. A non-zero exit code routes to Retry Counter.
:::

### 3.4.3 If Render OK + Retry Counter

After Run Render, `exitCode` is checked. If not `0`, Retry Counter reads `retry_count`. Below 3: requeues with `READY` and increments counter. At 3: marks `FATAL` and builds the **Fatal Alert Payload**.

Instead of a single Slack notification, the pipeline now fans out a comprehensive FATAL alert (containing the explicit error `$json.message`) concurrently to **Slack**, **Telegram**, and **Gmail**, allowing the operational team to respond rapidly to broken render chains.

| Scenario | Action |
| --- | --- |
| `exitCode = 0` | Continue to file read and upload |
| `exitCode != 0, retry_count < 3` | Set render = READY, increment retry_count |
| `exitCode != 0, retry_count = 3` | Set render = FATAL, log, send Slack alert |

### Retry Logic — If Render OK + Retry Counter + If Requeue or Fatal

![Retry Logic](/img/screenshots/figure-023.png)

### 3.4.4 Render Timestamps

The workflow captures render start and end timestamps. Duration in seconds is calculated and written to the Queue row for render time tracking.

| Node | Column Written | Format |
| --- | --- | --- |
| **Render Start Timestamp** | `render_start` | `YYYY-MM-DD HH:MM:SS UTC` |
| **Render End Timestamp** | `render_end` + `duration_sec` | Same format + integer seconds |

### Render Start Timestamp — Code Panel

![Render Start Timestamp](/img/screenshots/figure-012.png)

### 3.4.5 Upload File + Update Job Link

After render, the `.mp4` is read from disk and uploaded to the designated Google Drive folder. The returned `webContentLink` is stored in `file_link`, and render status is updated to `FINISHED`.

### Upload File — Google Drive Settings

![Upload File](/img/screenshots/figure-013.png)

### Update Job Link — Column Mapping

![Update Job Link](/img/screenshots/figure-014.png)

### 3.4.6 Notification Nodes

The following notification nodes were added to handle various states of rendering events (success, errors, fatal limits, and system crashes) automatically:

- **Build Success Payload** (Set node) — triggered after `Log Render FINISHED`, sends `render_id`, `job_id`, `file_link`, and `duration_sec`.
- **Build Error Payload** (Set node) — triggered after `Requeue to READY`, sends `render_id`, `retry_count`, and `error_message`.
- **Build Fatal Payload** (Set node) — triggered after `Log Render FATAL`, sends `render_id`, `job_id`, and reason (`max retry reached`).
- **Slack, Telegram, Gmail, Discord** — individual routing nodes configured for each payload type (Success, Error, Fatal).
- **Error Trigger node** — global listener for unexpected application failures.
- **Build System Error Payload** (Set node) — constructs the exact system crash output.
- **Slack, Telegram, Gmail, Discord** — individual routing nodes for the System Error payload.

### Success Notification Branch

![Success Notification Branch](/img/screenshots/figure-027.png)

### Error / Retry Notification Branch

![Error Notification Branch](/img/screenshots/figure-028.png)

### Fatal Notification Branch

![Fatal Notification Branch](/img/screenshots/figure-029.png)

### Error Trigger Area

![Error Trigger Area](/img/screenshots/figure-030.png)
