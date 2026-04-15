---
id: jsx-scripts
title: 4. JSX Scripts Reference
---

# 4. JSX Scripts Reference

Three ExtendScript (`.jsx`) files control all Adobe After Effects operations. All scripts are invoked via `AfterFX.exe -s evalFile` and receive parameters through `$.global` variables injected in the command string.

:::danger CRITICAL
JSX scripts via AfterFX.exe is the ONLY valid render method for this system. Nexrender or any headless renderer is NOT compatible.
:::

## 4.1 Auto_Update_RW.jsx

Replaces the old RW composition folder with a newly imported RW `.aep` file. Matches comps by name, replaces all layer sources across the project, renames the folder to the original name, and fixes the SN layer CSV reference.

| Parameter | Description |
| --- | --- |
| `$.global.templatePath` | Full path to the main `.aep` template file |
| `$.global.rwUpdatePath` | Full path to the new RW `.aep` file to import |

| Step | Description |
| --- | --- |
| 1. Validate inputs | Checks both paths are set and files exist |
| 2. Open template | Opens the main AEP file |
| 3. Validate RW file | Checks file is named `RW_*.aep` — skips if invalid |
| 4. Find PRJ folder | Locates 03_Source > PRJ folder in project panel |
| 5. Import new RW | Imports new RW `.aep` as folder item, moves to PRJ |
| 6. Match & replace comps | For each matching comp name: `replaceSource()` on all parent layers |
| 7. Delete old RW | Removes old RW folder from project panel |
| 8. Rename | Renames imported folder to original RW folder name |
| 9. Fix CSV reference | Updates SN comp layer name to match new RW CSV filename |
| 10. Bake text | Converts all text expressions to static values via `setValue()` |
| 11. Save + quit | Saves project to original path and calls `app.quit()` |

### Auto_Update_RW.jsx — Script in Text Editor

![Auto Update RW Script](/img/screenshots/figure-015.png)

**Log output:** `C:/ae-automated/exports/rw_update_{template_slug}.txt`

## 4.2 Export_Comp.jsx

Scans all compositions in the AE project, extracts metadata for comps matching the regulatory prefix filter, performs deep layer extraction including precomp traversal, and writes structured JSON to two output files.

| Parameter | Description |
| --- | --- |
| `$.global.templatePath` | Path to the `.aep` file to scan (optional — falls back to File.openDialog) |

:::warning
In Workflow 1, this script is called from `C:/ae-automated/test_src/Export Comp.jsx` — not the `/scripts/` folder. Verify this path matches your actual file location before deploying.
:::

### Export_Comp.jsx — Script in Text Editor

![Export Comp Script](/img/screenshots/figure-016.png)

**Output files:**
- Stable: `C:/ae-automated/exports/project_data.json`
- Cache: `C:/ae-automated/exports/cache/project_data_{template_slug}.json`

### project_data.json — Sample Output

![Project Data JSON](/img/screenshots/figure-017.png)

| JSON Field | Description |
| --- | --- |
| `schema_version` | Always 2.0 |
| `extracted_at` | ISO timestamp of extraction |
| `template.id` | Slug of the `.aep` filename |
| `template.file_path` | Full disk path of the `.aep` file |
| `compositions[]` | Array of all qualifying comp objects |
| `compositions[n].layers[]` | Top-level layers of the comp |
| `compositions[n].deep_layers[]` | All layers recursively including precomps |

## 4.3 Render.jsx

Opens the specified AE project, finds the target composition by name, clears the render queue, adds the comp with the specified output path, renders, saves, and quits. All operations are logged.

A delete check was added before the render starts. If the output file already exists on disk, the script automatically deletes it before rendering. This prevents the After Effects overwrite popup from appearing during automation, which would cause the workflow to hang indefinitely waiting for user input.

| Parameter | Description |
| --- | --- |
| `$.global.templatePath` | Full path to the `.aep` project file |
| `$.global.compName` | Exact composition name to render (12-token format) |
| `$.global.outputPath` | Full output path including filename and `.mp4` extension |

| Step | Description |
| --- | --- |
| 1. Open project | Opens the specified AE project |
| 2. Find composition | Locates the target composition by `compName` |
| 3. Clear render queue | Removes any pending jobs in the AE render queue |
| 4. Delete existing output | Checks if output file exists → deletes it silently → logs result |
| 5. Set output path | Adds the comp to queue and sets the output module and path |
| 6. Render | Executes the render |
| 7. Save + quit | Saves and closes the application |

### Render.jsx — Script in Text Editor

![Render.jsx Script](/img/screenshots/figure-018.png)

:::info
The script uses the output module already configured in the AE project. Ensure the project default render settings are correct before running.
:::

**Log output:** `C:/ae-automated/exports/render_{template_slug}.txt`
