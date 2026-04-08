---
id: system-overview
title: 1. System Overview
slug: /
---

# 1. System Overview

The Capital AE Automation system is a fully automated pipeline orchestrating Adobe After Effects rendering through n8n workflows, controlled via Google Sheets, with final video assets delivered to Google Drive.

## 1.1 Architecture

| Component | Role & Description |
| --- | --- |
| **n8n** | Workflow orchestrator — polls Google Sheets every minute, triggers all automation steps |
| **AfterFX.exe** | Adobe After Effects executable — invoked via Execute Command with `-s evalFile` |
| **JSX Scripts** | ExtendScript files controlling all AE operations: open, update, render, save, quit |
| **Google Sheets** | Job queue and status tracking — `Read_template` tab and `Queue` tab |
| **Google Drive** | Output storage for all rendered `.mp4` video files |

## 1.2 Pipeline Flow

The end-to-end automation follows this sequence:

1. Operator sets job status to `READY` in Google Sheets (Read_template tab)
2. n8n polls every minute and detects the `READY` status
3. n8n validates template and RW file paths on disk
4. If RW file exists: AfterFX runs `Auto_Update_RW.jsx` to replace old RW comp
5. AfterFX runs `Export_Comp.jsx` to extract all comp data to `project_data.json`
6. n8n reads JSON, filters regulatory comps, and populates the Queue tab
7. Queue entries trigger Workflow 2 (Render) per composition
8. AfterFX runs `Render.jsx` for each composition
9. Rendered `.mp4` is uploaded to Google Drive
10. Status updated to `FINISHED` / `FAILED` with full logging

## 1.3 Status Flow

**Read_template tab — job-level:**

| Status | Description |
| --- | --- |
| `READY` | Operator sets |
| `PROCESSING` | Job picked up |
| `RENDERING_RW` | RW updating |
| `RENDERING_EXPORT` | Exporting comps |
| `FINISHED` | All done |
| `FAILED` | Error occurred |

**Queue tab — render-level:**

| Status | Description |
| --- | --- |
| `READY` | Awaiting render |
| `RENDERING` | AE rendering |
| `FINISHED` | Uploaded to Drive |
| `FATAL` | Max retries hit |

## 1.4 System Paths

| Path | Purpose |
| --- | --- |
| `C:/Program Files/Adobe/Adobe After Effects 2026/Support Files/AfterFX.exe` | AE executable |
| `C:/ae-automated/scripts/` | JSX script files |
| `C:/ae-automated/exports/` | JSON exports and render logs |
| `C:/ae-automated/exports/project_data.json` | Stable JSON output from Export Comp |
| `C:/ae-automated/renders/out/` | Local rendered `.mp4` output |
| `C:/ae-automated/renders/temp_jobs/` | Temporary nexrender job JSON files |
| `C:/ae-automated/renders/render_history/` | Render job history folder |

### System Architecture — n8n Canvas Overview
![Automate Export Layers V2](/img/screenshots/figure-000.png)
![Render Automation V2](/img/screenshots/figure-001.png)

:::info
**Google Sheet ID:** 1BNGKNamOkjNvnR7DyDiqS0G3-qWzVdV6BlyzJ_RrLFk
**Google Drive Folder ID:** 1scgJBlL50Iaz19djbFC257dOghgpAzNh
:::
