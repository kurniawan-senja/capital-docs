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

### System Architecture — n8n Canvas Overview - Automate Export Layers V2

![Automate Export Layers V2 (with Notification Nodes)](/img/screenshots/figure-000.png)

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
| **Build Error Payload** | Code | Formats structured message for logical failure alert | JSON payload |
| **Alert Systems (Slack/TG/Mail)** | HTTP / Mail | Dispatches failure alerts to multiple channels | HTTP 200 |
| **Error Trigger** | Error Trigger | Global listener catching unhandled n8n failures | Exception data |
| **Build System Error Payload** | Code | Formats structured multi-channel alert for unhandled nodes | JSON payload |

## 2.4 Node Detail — Key Nodes

### 2.4.1 Read Job Queue (Trigger)

Polls the Read_template sheet every minute watching the status column. When any row changes, the workflow fires.

| Setting | Value |
| --- | --- |
| **Sheet ID** | 1BNGKNamOkjNvnR7DyDiqS0G3-qWzVdV6BlyzJ_RrLFk |
| **Tab** | Read_template (gid=1093980937) |
| **Poll interval** | Every minute |
| **Column to watch**| `status` |

### Read Job Queue — Node Settings

![Read Job Queue](/img/screenshots/figure-003.png)

### 2.4.2 Normalize Path

Converts Windows backslashes to forward slashes in both `template_path` and `rw_path`, ensuring compatibility across all Execute Command nodes.

```javascript
const template_path = $('Loop Update1').first().json.template_path.replace(/\\/g, "/");
const rw_path = $('Loop Update1').first().json.rw_path.replace(/\\/g, "/");

return [{
  json: {
    ...$json,
    template_path: template_path,
    rw_path: rw_path
  }
}];
```

**Code Explanation:**
- `$('Loop Update1').first()`: Dynamically fetches the active row item passing through the `Loop Update1` node.
- `.replace(/\\/g, "/")`: Uses a global Regular Expression (RegEx) to find all Windows backslashes (`\`) from the Google Sheets input and replaces them with standard forward slashes (`/`). This prevents file-path escaping errors when passed into the *Execute Command* nodes.
- `...$json`: Returns the original input data intact, but overrides the `template_path` and `rw_path` properties with our sanitized string variables.

### Normalize Path — Code Panel

![Normalize Path](/img/screenshots/figure-004.png)

### 2.4.3 Execute RW Update

Launches AfterFX.exe in script mode to run `Auto_Update_RW.jsx`. Both paths are injected as global variables before evalFile.

```bash
AfterFX.exe -s "$.global.templatePath='...'; $.global.rwUpdatePath='...'; $.evalFile('C:/ae-automated/scripts/Auto Update RW.jsx');"
```

:::warning
AfterFX.exe runs synchronously. The Wait node (10 min) after this step provides buffer time for AE to complete and save.
:::

### Execute RW Update — Command Field

![Execute RW Update](/img/screenshots/figure-005.png)

### 2.4.4 Filter Only Regulatories

Filters compositions to include only those starting with valid regulatory prefix codes: `CRP, GOL, GAS, OIL, IND, FOX, PGE, COM, EQU, BND, OTH`.

```javascript
const comps = $json.data.compositions || [];

const business = comps.filter(c =>
  /^(CRP|GOL|GAS|OIL|IND|FOX|PGE|COM|EQU|BND|OTH)_/.test(c.name)
);

return [{
  json: {
    ...$json,
    businessComps: business
  }
}];
```

**Code Explanation:**
- `const comps`: Retrieves the array of compositions parsed from the `project_data.json` file.
- `comps.filter(...)`: Uses a Regular Expression (`test(c.name)`) to safely keep only compositions whose name starts with one of the approved regulatory prefixes (e.g., CRP, GOL).
- `...$json`: Returns the payload with the newly created `businessComps` array containing only the matched compositions.

### Filter Only Regulatories — Code Panel

![Filter Only Regulatories](/img/screenshots/figure-006.png)

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

```javascript
function parse(comp) {
  const tokens = comp.name.split("_");

  if (tokens.length < 12) {
    throw new Error("Invalid comp naming format: " + comp.name);
  }

  return {
      category: tokens[0],
      campaign: tokens[1],
      instrument: tokens[2],
      version: tokens[3],
      regulation: tokens[4],
      rw: tokens[5],
      language: tokens[6],
      creativeType: tokens[7],
      creativeSize: tokens[8],
      creativeLength: tokens[9],
      source: tokens[10],
      acpm: tokens[11]
  };
}

const indexed = $json.businessComps.map(parse);

return [{
  json: {
    indexedComps: indexed
  }
}];
```

**Code Explanation:**
- `function parse(comp)`: A custom parser function that breaks down a single composition object.
- `comp.name.split("_")`: Slices the long file name into an array of segments using the underscore character as a delimiter.
- `if (tokens.length < 12)`: A strict validation check. If a designer missed an underscore, it forcefully throws an Error to halt the workflow and prevent garbage data from entering Google Sheets.
- `return { category: tokens[0]... }`: Maps each segment by its index to a structured JSON property.
- `map(parse)`: Loops the parser over every composition inside the `businessComps` array, resulting in a fully structured `indexedComps` payload.

### Index Compositions — Code Panel

![Index Compositions](/img/screenshots/figure-007.png)

### 2.4.6 Error Handling & Notifications

When the template file is not found on disk (If Exist? FALSE branch), the workflow routes to Update Failed which sets status = FAILED and records the error step and message. Log FAILED appends the event to the Logs tab.

Once the failure is logged, **Build Error Payload** constructs a formatted message containing `job_id`, `eror_step`, and `eror_message`. This alert is then broadcast to the team across three channels simultaneously:
- **Slack** (via Webhook)
- **Telegram** (via Bot API)
- **Gmail** (via SMTP routing)

### Error Path — Update Failed + Log FAILED

![Error Path](/img/screenshots/figure-008.png)

### 2.4.7 Notification Nodes

The following notification nodes were appended to the end of the existing automation logic to handle success, failure, and system-level exceptions without modifying the core business logic.

- **Build Success Payload** (Set node) — triggered after `Log FINISHED`, sends `job_id`, status, and timestamp.
- **Build Error Payload** (Set node) — triggered after `Log FAILED`, sends `job_id`, `step_failed`, and `error_message`.
- **Slack, Telegram, Gmail** — individual nodes routing the Success payload.
- **Slack, Telegram, Gmail** — individual nodes routing the Error payload.
- **Error Trigger node** — catches global, system-level crashes (unhandled n8n application exceptions) not covered by business logic.
- **Build System Error Payload** (Set node) — formats the system error message from the Error Trigger.
- **Slack, Telegram, Gmail** — individual nodes routing the System Error payload.

### Success Notification Branch

![Success Notification Branch](/img/screenshots/figure-024.png)

### Error Notification Branch

![Error Notification Branch](/img/screenshots/figure-025.png)

### Error Trigger Area

![Error Trigger Area](/img/screenshots/figure-026.png)
