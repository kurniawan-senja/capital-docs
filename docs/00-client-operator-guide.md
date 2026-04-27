---
id: client-operator-guide
title: 0. Client Operator Guide (SOP)
---

# 0. Client Operator Guide

Welcome to the **Capital After Effects Render Automation** system. This Standard Operating Procedure (SOP) is written specifically for Daily Operators (Video Editors, Traffic Managers, and Production Coordinators). 

This system acts as a direct bridge between your Google Sheet Data Entry and a dedicated After Effects rendering server. By following these operational procedures, you can command the server to extract templates, update regional variants, and render hundreds of videos completely hands-free.

---

## 1. Triggering an Extraction (Phase 1)

Before videos can be rendered, the system must read your Master After Effects Template and extract its data into the queue.

### Step-by-Step Instructions:
1. Open the **Layer Control Google Sheet**.
2. Navigate to the **`Read_template`** tab.
3. In a new or existing row, fill in the **`template_path`**. 
   - **Format Rules:** The path must be a complete Windows directory path starting with the drive letter.
   - **Correct:** `C:\ae-automated\01_Projects\Campaign_A\Master_Template.aep`
   - **Incorrect:** `Campaign_A/Master_Template.aep` (Missing drive letter and using wrong slashes).
4. If you are updating a Regional Variant, fill in the **`rw_path`**. If not, leave it completely blank.
   - **Format Rules:** Must point directly to an `.aep` file starting with `RW_`.
5. Finally, type the word **`START`** (all caps) into the **`trigger_export`** column.

### Verification:
Within 60 seconds, you will see the `trigger_export` cell change automatically to `PROCESSING`. This means the server has picked up your command and is actively opening After Effects in the background. Do not edit the row while it is processing.

---

## 2. Managing the Render Queue (Phase 2)

Once Phase 1 finishes (the status turns to `FINISHED`), navigate to the **`Queue`** tab in your Google Sheet.

1. You will notice that the system has automatically populated new rows based on the data extracted from your template.
2. The `render` column for these new rows will automatically be set to **`READY`**.
3. **No action is required from you.** The render server checks the `Queue` tab every minute. It will grab the top-most row marked `READY` and begin rendering it.

---

## 3. Comprehensive Status Glossary

As the automation progresses, the Google Sheet is updated in real-time. Understanding these statuses is critical to monitoring the health of the queue.

### Extraction Statuses (`Read_template` tab)
| Status | Meaning | Estimated Time |
| :--- | :--- | :--- |
| **`START`** | Job is queued. Server has not seen it yet. | 1 minute |
| **`PROCESSING`** | Server is scanning the `.aep` file and extracting layer data. | 2 - 5 minutes |
| **`RENDERING_RW`** | Server is actively importing and replacing the new Regional Variant assets. | 3 - 6 minutes |
| **`FINISHED`** | Success! The data has been pushed to the `Queue` tab. | N/A |
| **`FATAL`** | System crashed or file not found. Extraction aborted. | N/A |

### Render Statuses (`Queue` tab)
| Status | Meaning | Estimated Time |
| :--- | :--- | :--- |
| **`READY`** | Video is in line waiting to be rendered. | Varies by queue size |
| **`RENDERING`** | After Effects is currently rendering this specific video. | Depends on comp length |
| **`FINISHED`** | Success! Video is uploaded. See `file_link` column. | N/A |
| **`FATAL`** | Render failed 3 consecutive times. System skipped this video. | N/A |

---

## 4. Troubleshooting & Alert Interpretation

If something goes wrong, the system will instantly send a notification payload to your designated **Slack** or **Telegram** channels. It will also write the error directly into the Google Sheet.

### Invisible Errors vs. FATAL
> [!NOTE]
> **The Automatic Retry System:** If After Effects crashes unexpectedly mid-render, the system will silently log an **`ERROR`** and automatically switch the status back to **`READY`**. It will retry up to **3 times**. You do not need to intervene. It only becomes **`FATAL`** if it fails 3 times in a row.

### Common Error Messages & Solutions

| Error Message in Sheet/Slack | Cause | Operator Solution |
| :--- | :--- | :--- |
| `Error: Template file not found` | The `template_path` has a typo, or the file was moved/deleted from the server. | Verify the exact spelling of the `.aep` file and ensure the file exists on the `C:\` drive. |
| `Error: PRJ folder not found` | Your Regional Variant update failed because the master template does not have a `03_Source/PRJ` folder structure. | Open the template manually, fix the folder structure to match the standard, and try again. |
| `Error: Comp not found: [Name]` | The composition name built by the Google Sheet does not match any composition inside the `.aep` file. | Check your Naming Conventions. Ensure the 12-segment name in the sheet perfectly matches the AE comp name. |

---

## 5. Safe Recovery Protocol (SOP)

If a job hits **`FATAL`** and you have identified and fixed the underlying issue (e.g., you fixed the typo in the file path), follow this strict sequence to recover the job without duplicating data:

### Recovering a Failed Extraction (`Read_template`):
1. Fix the typo in the `template_path` or `rw_path`.
2. Delete the text in the `error_message` column for that row so it is blank.
3. Type **`START`** into the `trigger_export` column.
4. Press Enter. The system will re-attempt the extraction.

### Recovering a Failed Render (`Queue`):
1. Fix the issue (e.g., relink missing footage inside the master AE project).
2. Go to the failed row in the `Queue` tab.
3. Delete the text in the `error_message` column.
4. Delete the number in the `retry_count` column.
5. Type **`READY`** into the `render` column.
6. Press Enter. The server will pick it up on the next minute cycle.

---

## 6. Strict Governance: What NOT to Touch

The Google Sheet is deeply connected to an SQL-like mapping system. Modifying structural elements will permanently break the automation pipeline.

> [!CAUTION] STRICTLY FORBIDDEN
> - **Do NOT rename the Column Headers (Row 1).** The server relies on these exact names (e.g., `template_path`, `render_id`).
> - **Do NOT modify the `job_id` or `render_id` columns.** These are unique cryptographic hashes. Changing one character will sever the connection between the Queue and the uploaded video.
> - **Do NOT manually insert empty rows in the middle of the `Queue` data.** The system reads data sequentially. Blank rows may cause the server to stop parsing.
> - **Do NOT alter the Naming Conventions of your compositions.** The 12-segment naming format is strictly enforced by the extraction script.

---

## 7. Account & Infrastructure Maintenance

This system relies on specific API credentials, webhooks, and Service Accounts. **Operators should not attempt to modify server settings or n8n workflows.**

Please contact the **Capital Automation Engineering Team** if you require any of the following:
- Changing the target Google Drive destination folder.
- Adding or changing Slack/Telegram notification channels.
- Renewing expired Google OAuth2 API credentials.
- Modifying the underlying ExtendScript (`.jsx`) rendering logic.
- Altering the baseline After Effects Render Settings (currently locked to `H.264 - Match Render Settings - 15 Mbps`).
