---
id: workflow-1
title: 2. Workflow 1 — Capital Automate Export Layers v2
---

# 2. Workflow 1 — Capital Automate Export Layers v2

## 2.1 Purpose & Scope

This workflow prepares the AE project for batch rendering. It validates file paths, optionally replaces the RW comp, extracts composition metadata, indexes comps using the 12-token naming convention, and populates the Queue sheet with one row per composition ready to render.

:::tip
Workflow 1 must complete successfully before Workflow 2 can begin. The Queue tab is entirely populated by this workflow.
:::

## 2.2 Workflow Summary

| Property | Value |
| --- | --- |
| **Trigger** | Google Sheets polling — Read_template tab, every minute |
| **Condition** | Row status = `READY` |
| **Sheet column watched** | `status` |
| **Output** | Queue tab populated with render rows |
| **Success status** | `FINISHED` |
| **Failure status** | `FAILED` |

### System Architecture — n8n Canvas Overview

![Automate Export Layers V2](/img/screenshots/figure-000.png)

:::info Screencast Workflow 1 Full Demo
Set a job to READY in Google Sheets, watch n8n trigger, validate files, update RW, export comps, and populate the Queue tab. Duration: 3-5 min.
Link: https://streamable.com/9act30
:::

## 2.3 Node Summary Table

| Node | Type | Purpose | Output |
| --- | --- | --- | --- |
| **Read Job Queue** | Google Sheets Trigger | Polls Read_template every minute for status changes | All changed rows |
| **IF READY?** | IF Node | Routes only rows with status = READY | TRUE / FALSE branch |
| **Loop Update** | Split In Batches | Processes one job at a time | Single job item |
| **Wait** | Wait | Delays 1 min before fetching job data | Passes item through |
| **Get Path** | Google Sheets | Fetches full row data for the job_id | Full job row |
| **Update Processing** | Google Sheets | Sets status = PROCESSING | Confirmation |
| **Normalize Path** | Code | Converts backslashes to forward slashes in paths | Clean paths |
| **Check if Exist** | Execute Command | Checks if .aep template file exists on disk | OK / MISSING |
| **If Exist?** | IF Node | Routes based on template file existence | TRUE / FALSE |
| **If RW Exist** | Execute Command | Checks if RW .aep file exists on disk | OK / MISSING |
| **If RW** | IF Node | Routes: with RW vs skip to export | TRUE / FALSE |
| **Update RENDERING_RW** | Google Sheets | Sets status = RENDERING_RW | Confirmation |
| **Execute RW Update** | Execute Command | Runs AfterFX with Auto_Update_RW.jsx | stdout / exitCode |
| **Wait Update** | Wait | Waits 10 min for AE to complete RW update | Passes through |
| **Update RENDERING_EXPORT** | Google Sheets | Sets status = RENDERING_EXPORT | Confirmation |
| **Execute Export Comp** | Execute Command | Runs AfterFX with Export_Comp.jsx | stdout / exitCode |
| **Read JSON file** | Read/Write File | Reads `project_data.json` from disk | Binary file data |
| **Extract from File** | Extract From File | Parses binary to JSON object | JSON object |
| **Filter Only Regulatories** | Code | Filters comps by regulatory prefix codes | `businessComps[]` |
| **Index Compositions** | Code | Parses 12-token comp names into structured fields | `indexedComps[]` |
| **Flatten Data** | Code | Maps each comp to a render row with job metadata | Array of row items |
| **Update Comp Ready to Render**| Google Sheets | Appends all render rows to Queue tab | Confirmation |
| **Aggregate2** | Aggregate | Collects all `job_ids` from the batch | Aggregated array |
| **Update Status Finished** | Google Sheets | Sets job status = FINISHED | Confirmation |
| **Log FINISHED** | Google Sheets | Appends success event to Logs tab | Confirmation |
| **Update Failed** | Google Sheets | Sets status = FAILED with error details | Confirmation |
| **Log FAILED** | Google Sheets | Appends failure event to Logs tab | Confirmation |
| **End Run** | Set | Terminal node for non-READY rows | No output |

## 2.4 Node Detail — Key Nodes

### 2.4.1 Read Job Queue (Trigger)

Polls the Read_template sheet every minute watching the status column. When any row changes, the workflow fires.

| Setting | Value |
| --- | --- |
| **Sheet ID** | 1BNGKNamOkjNvnR7DyDiqS0G3-qWzVdV6BlyzJ_RrLFk |
| **Tab** | Read_template (gid=1093980937) |
| **Poll interval** | Every minute |
| **Column to watch**| `status` |

### 2.4.2 Normalize Path

Converts Windows backslashes to forward slashes in both `template_path` and `rw_path`, ensuring compatibility across all Execute Command nodes.

```javascript
template_path = $('Loop Update').first().json.template_path.replace(/\\/g, '/');
rw_path = $('Loop Update').first().json.rw_path.replace(/\\/g, '/');
```

### 2.4.3 Execute RW Update

Launches AfterFX.exe in script mode to run `Auto_Update_RW.jsx`. Both paths are injected as global variables before evalFile.

```bash
AfterFX.exe -s "$.global.templatePath='...'; $.global.rwUpdatePath='...'; $.evalFile('C:/ae-automated/scripts/Auto Update RW.jsx');"
```

:::warning
AfterFX.exe runs synchronously. The Wait node (10 min) after this step provides buffer time for AE to complete and save.
:::

### 2.4.4 Filter Only Regulatories

Filters compositions to include only those starting with valid regulatory prefix codes: `CRP, GOL, GAS, OIL, IND, FOX, PGE, COM, EQU, BND, OTH`.

### 2.4.5 Index Compositions

Parses the 12-token composition name into structured JSON fields. Each token is separated by underscore.

| # | Token | Example |
| --- | --- | --- |
| 1 | category | OTH |
| 2 | campaign | SM-instruments-charts-news-T |
| 3 | instrument | Tesla |
| 4 | version | v1 |
| 5 | regulation | BAH |
| 6 | rw | 0 |
| 7 | language | EN |
| 8 | creativeType | Video |
| 9 | creativeSize | 1080x1920 |
| 10 | creativeLength | 30sec |
| 11 | source | CT |
| 12 | acpm | 0.007 |

### 2.4.6 Error Handling — Update Failed and Log FAILED

When the template file is not found on disk (If Exist? FALSE branch), the workflow routes to Update Failed which sets status = FAILED and records the error step and message. Log FAILED appends the event to the Logs tab.
