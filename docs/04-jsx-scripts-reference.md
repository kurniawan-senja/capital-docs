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

<details>
<summary><b>View Full Auto_Update_RW.jsx Source Code</b></summary>

```javascript
(function () {

if (!$.global.templatePath)
    throw new Error("templatePath not provided");

if (!$.global.rwUpdatePath)
    throw new Error("rwUpdatePath not provided");

var projectFile = new File($.global.templatePath);
var rwFile = new File($.global.rwUpdatePath);

if (!projectFile.exists)
    throw new Error("Template file not found");

// ---------------- CLOSE ANY OPEN PROJECT ----------------

try {
    app.project.close(CloseOptions.DO_NOT_SAVE_CHANGES);
} catch (e) {}

// ---------------- LOGGING ----------------

var exportFolder = new Folder("C:/ae-automated/exports");
if (!exportFolder.exists) exportFolder.create();

var templateSlug = projectFile.name.toLowerCase().replace(/[^a-z0-9]/g, "_");
var logFile = new File(exportFolder.fullName + "/rw_update_" + templateSlug + ".txt");
logFile.open("w");

function log(msg) {
    logFile.writeln(msg);
}

log("RW UPDATE STARTED");
log("Template: " + projectFile.fullName);
log("RW Path: " + rwFile.fullName);

app.beginUndoGroup("RW Update");

try {

    // ---------------- OPEN TEMPLATE ----------------

    app.open(projectFile);
    log("Template opened.");

    if (!rwFile.exists || !rwFile.name.match(/^RW_.*\.aep$/i)) {
        log("Invalid RW file. Skipping.");
        app.project.save();
        return;
    }

    // ---------------- FIND PRJ ----------------

    function findFolder(name) {
        for (var i = 1; i <= app.project.numItems; i++) {
            var item = app.project.item(i);
            if (item instanceof FolderItem && item.name === name)
                return item;
        }
        return null;
    }

    var sourceFolder = findFolder("03_Source");
    var prjFolder = null;

    for (var i = 1; i <= sourceFolder.numItems; i++) {
        var item = sourceFolder.item(i);
        if (item instanceof FolderItem && item.name === "PRJ") {
            prjFolder = item;
            break;
        }
    }

    if (!prjFolder)
        throw new Error("PRJ folder not found");

    // ---------------- IMPORT RW ----------------

    log("Importing RW...");
    var importedRoot = app.project.importFile(new ImportOptions(rwFile));

    if (!(importedRoot instanceof FolderItem))
        throw new Error("Imported RW did not return folder");

    importedRoot.parentFolder = prjFolder;
    log("Imported folder moved into PRJ: " + importedRoot.name);

    importedRoot.name = "NEW_RW_TEMP";

    // ---------------- FIND OLD RW ----------------

    var oldRW = null;

    for (var i = 1; i <= prjFolder.numItems; i++) {
        var item = prjFolder.item(i);
        if (item instanceof FolderItem && item.name.match(/^RW_/i)) {
            oldRW = item;
            break;
        }
    }

    if (!oldRW)
        throw new Error("Old RW not found");

    var originalName = oldRW.name;

    // ---------------- REPLACE COMPS ----------------

    function getComps(folder) {
        var comps = [];
        for (var i = 1; i <= folder.numItems; i++) {
            var item = folder.item(i);
            if (item instanceof CompItem)
                comps.push(item);
            else if (item instanceof FolderItem)
                comps = comps.concat(getComps(item));
        }
        return comps;
    }

    var oldComps = getComps(oldRW);
    var newComps = getComps(importedRoot);

    for (var i = 0; i < oldComps.length; i++) {
        for (var j = 0; j < newComps.length; j++) {
            if (oldComps[i].name === newComps[j].name) {

                var usedIn = oldComps[i].usedIn;

                for (var k = 0; k < usedIn.length; k++) {
                    var parent = usedIn[k];
                    for (var l = 1; l <= parent.numLayers; l++) {
                        if (parent.layer(l).source === oldComps[i]) {
                            parent.layer(l).replaceSource(newComps[j], true);
                        }
                    }
                }
            }
        }
    }

    // ---------------- DELETE OLD ----------------

    oldRW.remove();
    log("Old RW removed.");

    // ---------------- RENAME BACK ----------------

    importedRoot.name = originalName;
    log("New RW renamed to: " + originalName);

    // ---------------- FIX CSV REFERENCE ----------------

    function fixSNCSVReference() {
        var targetCSVName = null;

        for (var i = 1; i <= app.project.numItems; i++) {
            var item = app.project.item(i);

            if (item instanceof FootageItem && item.file) {
                var filename = item.file.name;

                if (filename.match(/^RW-.*\.csv$/i)) {
                    targetCSVName = item.name;
                    break;
                }
            }
        }

        if (!targetCSVName) {
            throw new Error("RW CSV not found");
        }

        var compSN = app.project.itemByName("SN");

        if (compSN && compSN instanceof CompItem) {
            compSN.layer(1).name = targetCSVName;
        }
    }

    // ---------------- FIX CSV REFERENCE ----------------
    fixSNCSVReference();
    log("SN layer now points to correct RW CSV");

    // ---------------- BAKE TEXT ----------------
    function bakeAllText() {
        for (var i = 1; i <= app.project.numItems; i++) {
            var item = app.project.item(i);

            if (item instanceof CompItem) {
                for (var l = 1; l <= item.numLayers; l++) {
                    var layer = item.layer(l);

                    try {
                        var textProp = layer.property("Source Text");
                        if (textProp) {
                            var val = textProp.value;
                            textProp.setValue(val);
                        }
                    } catch (e) {}
                }
            }
        }
    }

    bakeAllText();
    log("All text baked (expressions converted to static)");

    // ---------------- SAVE ----------------
    app.project.save(new File(projectFile.fullName));
    $.sleep(500);
    log("Saved to: " + projectFile.fullName);

} catch (err) {

    log("ERROR: " + err.toString());
    throw err;


} finally {

    try {
        app.endUndoGroup();

        app.project.save(new File(projectFile.fullName));
        $.sleep(300);

        log("Final save completed");

    } catch (e) {
        log("Final save error: " + e.toString());
    }

    log("RW UPDATE FINISHED");
    logFile.close();

    app.quit();
}

})();
```

</details>

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

<details>
<summary><b>View Full Export_Comp.jsx Source Code</b></summary>

```javascript
/* =====================================================
   SAFE PROJECT EXTRACTION SCRIPT
   - Folder-based ID (EN->CySec)
   - Safe export directory creation
   - Manual fallback if templatePath not provided
===================================================== */

// =====================================================
// JSON STRINGIFY POLYFILL
// =====================================================
if (typeof JSON === "undefined") {
    JSON = {};
}

if (typeof JSON.stringify !== "function") {
    JSON.stringify = function (obj, indent) {
        function stringify(value, level) {
            var pad = "";
            for (var i = 0; i < level; i++) pad += "  ";

            if (value === null) return "null";
            if (typeof value === "number" || typeof value === "boolean")
                return value.toString();
            if (typeof value === "string")
                return "\"" + value.replace(/\\/g, "\\\\").replace(/"/g, '\\"').replace(/\r/g, "\\r").replace(/\n/g, "\\n")+ "\"";

            if (value instanceof Array) {
                var arr = [];
                for (var i = 0; i < value.length; i++) {
                    arr.push(stringify(value[i], level + 1));
                }
                return "[\n" + pad + "  " + arr.join(",\n" + pad + "  ") + "\n" + pad + "]";
            }

            if (typeof value === "object") {
                var props = [];
                for (var key in value) {
                    props.push(
                        "\"" + key + "\": " +
                        stringify(value[key], level + 1)
                    );
                }
                return "{\n" + pad + "  " + props.join(",\n" + pad + "  ") + "\n" + pad + "}";
            }

            return "null";
        }

        return stringify(obj, 0);
    };
}

// =====================================================
// PROJECT INPUT
// =====================================================
var projectFile;

if (typeof $.global.templatePath === "undefined" || !$.global.templatePath) {
    projectFile = File.openDialog("Select AEP file");
    if (!projectFile) {
        throw new Error("No project selected.");
    }
} else {
    projectFile = new File($.global.templatePath);
}

if (!projectFile.exists) {
    throw new Error("AEP file not found: " + projectFile.fullName);
}

// =====================================================
// EXPORT FOLDER SAFETY
// =====================================================
var exportFolder = new Folder("C:/ae-automated/exports/");
if (!exportFolder.exists) {
    exportFolder.create();
}

// =====================================================
// LOGGING SETUP
// =====================================================
var templateSlug = projectFile.name.toLowerCase().replace(/[^a-z0-9]/g, "_");
var logFile = new File(exportFolder.fullName + "/log_" + templateSlug + ".txt");
logFile.open("w");

function log(msg) {
    logFile.writeln(msg);
}

log("=== EXTRACTION STARTED ===");
log("Opening project: " + projectFile.fullName);

app.open(projectFile);

// =====================================================
// UTILITIES
// =====================================================
function slug(str) {
    return str.toLowerCase().replace(/[^a-z0-9]/g, "_");
}

function getTimestamp() {
    var d = new Date();
    function pad(n) { return n < 10 ? "0" + n : n; }

    return d.getFullYear() + "-" +
        pad(d.getMonth() + 1) + "-" +
        pad(d.getDate()) + "T" +
        pad(d.getHours()) + ":" +
        pad(d.getMinutes()) + ":" +
        pad(d.getSeconds());
}

// =====================================================
// FILTER RULES
// =====================================================
var ALLOWED_PREFIXES = [
    "CRP","GOL","GAS","OIL","IND",
    "FOX","PGE","COM","EQU","BND","OTH"
];

function hasAllowedPrefix(name) {
    if (!name) return false;
    var upper = name.toUpperCase();
    for (var i = 0; i < ALLOWED_PREFIXES.length; i++) {
        var prefix = ALLOWED_PREFIXES[i] + "_";
        if (upper.indexOf(prefix) === 0) {
            return true;
        }
    }
    return false;
}

// =====================================================
// LAYER DETECTION
// =====================================================
function detectLayerType(layer) {

    if (layer instanceof TextLayer) return "TEXT";
    if (layer.nullLayer) return "NULL";
    if (layer.adjustmentLayer) return "ADJUSTMENT";
    if (layer.matchName === "ADBE Vector Layer") return "SHAPE";
    if (layer.hasAudio && !layer.hasVideo) return "AUDIO";

    if (layer instanceof AVLayer) {
        if (layer.source instanceof CompItem) return "PRECOMP";
        if (layer.source && layer.source.file) return "FOOTAGE";
    }

    return "OTHER";
}

function detectRole(name, type) {

    var n = name.toLowerCase();

    if (n.indexOf("headline") !== -1) return "headline";
    if (n.indexOf("title") !== -1) return "title";
    if (n.indexOf("subtitle") !== -1) return "subtitle";
    if (n.indexOf("cta") !== -1) return "cta";
    if (n.indexOf("logo") !== -1) return "logo";
    if (n.indexOf("bg") !== -1 || n.indexOf("background") !== -1) return "background";
    if (n.indexOf("chart") !== -1) return "chart";
    if (n.indexOf("data") !== -1) return "chart_data";
    if (n.indexOf("disclaimer") !== -1) return "disclaimer";

    if (type === "TEXT") return "text_generic";
    if (type === "FOOTAGE") return "media_generic";

    return "structural";
}

// =====================================================
// GET FOLDER PATH (EXCLUDES OUT)
// =====================================================
function getFolderPath(item) {
    var folders = [];
    var parent = item.parentFolder;

    while (parent && parent.name !== "Root") {
        if (parent.name !== "OUT") {
            folders.unshift(parent.name);
        }
        parent = parent.parentFolder;
    }

    return folders.join("->");
}

// =====================================================
// RECURSIVE EXTRACTION
// =====================================================
function extractComp(comp, parentPath, rootCompId, resultArray) {

    var currentPath = parentPath ? parentPath : comp.name;

    for (var l = 1; l <= comp.numLayers; l++) {

        var layer = comp.layer(l);
        var type = detectLayerType(layer);

        var layerPath = currentPath + "->" + layer.name;

        var layerEntry = {
            root_comp_id: rootCompId,
            comp_name: comp.name,
            layer_index: l,
            layer_name: layer.name,
            layer_type: type,
            path: layerPath,
            is_automation_target: (
                type === "TEXT" ||
                type === "FOOTAGE"
            ),
            automation_role: detectRole(layer.name, type)
        };

        if (type === "TEXT") {
            var textProp = layer.property("Source Text");
            if (textProp) {
                layerEntry.text_default = textProp.value.text;
            }
        }

        if (type === "FOOTAGE" && layer.source && layer.source.file) {
            layerEntry.source_path = layer.source.file.fullName;
        }

        resultArray.push(layerEntry);

        if (type === "PRECOMP" && layer.source instanceof CompItem) {
            extractComp(layer.source, layerPath, rootCompId, resultArray);
        }
    }
}

// =====================================================
// MAIN STRUCTURE
// =====================================================
var data = {
    schema_version: "2.0",
    extracted_at: getTimestamp(),
    template: {
        id: "tpl_" + slug(app.project.file.name),
        name: app.project.file.name,
        file_path: app.project.file.fullName,
        ae_version: app.version
    },
    compositions: []
};

log("Scanning project items...");

// =====================================================
// MAIN LOOP
// =====================================================
for (var i = 1; i <= app.project.numItems; i++) {

    var item = app.project.item(i);

    if (item instanceof CompItem && hasAllowedPrefix(item.name) === true) {

        log("Exporting root comp: " + item.name);

        var compId = getFolderPath(item);

        var compData = {
            id: compId,
            name: item.name,
            width: item.width,
            height: item.height,
            duration: item.duration,
            frame_rate: item.frameRate,
            is_render_target: true,
            layers: [],
            deep_layers: []
        };

        for (var l = 1; l <= item.numLayers; l++) {

            var layer = item.layer(l);
            var type = detectLayerType(layer);

            compData.layers.push({
                id: slug(compId) + "_L" + l,
                index: l,
                name: layer.name,
                type: type,
                automation_role: detectRole(layer.name, type)
            });
        }

        extractComp(item, item.name, compId, compData.deep_layers);

        data.compositions.push(compData);
    }
}

// =====================================================
// EXPORT JSON (STABLE + CACHE)
// =====================================================

// Ensure export folder exists
var exportFolder = new Folder("C:/ae-automated/exports/");
if (!exportFolder.exists) {
    exportFolder.create();
}

// Ensure cache folder exists
var cacheFolder = new Folder("C:/ae-automated/exports/cache/");
if (!cacheFolder.exists) {
    cacheFolder.create();
}

// -----------------------------------------------------
// Stable file for n8n to always read
// -----------------------------------------------------
var stableOutputFile = new File(
    exportFolder.fullName + "/project_data.json"
);

log("Writing stable JSON...");
stableOutputFile.open("w");
stableOutputFile.write(JSON.stringify(data));
stableOutputFile.close();

// -----------------------------------------------------
// Cache file with template slug
// -----------------------------------------------------
var cacheOutputFile = new File(
    cacheFolder.fullName +
    "/project_data_" +
    templateSlug +
    ".json"
);

log("Writing cache JSON...");
cacheOutputFile.open("w");
cacheOutputFile.write(JSON.stringify(data));
cacheOutputFile.close();

log("JSON written successfully (stable + cache).");
log("=== EXTRACTION COMPLETE ===");
```

</details>

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

<details>
<summary><b>View Full Render.jsx Source Code</b></summary>

```javascript
(function () {

if (!$.global.templatePath)
    throw new Error("templatePath not provided");

if (!$.global.compName)
    throw new Error("compName not provided");

if (!$.global.outputPath)
    throw new Error("outputPath not provided");

var projectFile = new File($.global.templatePath);

if (!projectFile.exists)
    throw new Error("Template file not found");

// ---------------- CLOSE ANY OPEN PROJECT ----------------

try {
    app.project.close(CloseOptions.DO_NOT_SAVE_CHANGES);
} catch (e) {}

// ---------------- LOGGING ----------------

var exportFolder = new Folder("C:/ae-automated/exports");
if (!exportFolder.exists) exportFolder.create();

var templateSlug = projectFile.name.toLowerCase().replace(/[^a-z0-9]/g, "_");
var logFile = new File(exportFolder.fullName + "/render_" + templateSlug + ".txt");
logFile.open("w");

function log(msg) {
    logFile.writeln(msg);
}

log("RENDER STARTED");
log("Template: " + projectFile.fullName);
log("Comp: " + $.global.compName);
log("Output: " + $.global.outputPath);

app.beginUndoGroup("Render");

try {

    // ---------------- OPEN TEMPLATE ----------------

    var proj = app.open(projectFile);
    log("Template opened.");

    if (!proj)
        throw new Error("Failed to open project");

    // ---------------- FIND COMP ----------------

    var targetComp = null;

    for (var i = 1; i <= proj.numItems; i++) {
        var item = proj.item(i);
        if (item instanceof CompItem && item.name === $.global.compName) {
            targetComp = item;
            break;
        }
    }

    if (!targetComp)
        throw new Error("Comp not found: " + $.global.compName);

    log("Comp found: " + targetComp.name);

    // ---------------- CLEAR RENDER QUEUE ----------------

    while (proj.renderQueue.numItems > 0) {
        proj.renderQueue.item(1).remove();
    }

    log("Render queue cleared");

    // ---------------- ADD TO RENDER QUEUE ----------------

    var rqItem = proj.renderQueue.items.add(targetComp);

    var om = rqItem.outputModule(1);

    var outFile = new File($.global.outputPath);

    if (!outFile.parent.exists)
        outFile.parent.create();

    om.file = outFile;

    log("Output set: " + outFile.fullName);

    // ---------------- RENDER ----------------

    log("Rendering started...");
    proj.renderQueue.render();
    log("Rendering finished");

    // ---------------- SAVE ----------------

    proj.save(new File(projectFile.fullName));
    $.sleep(300);

    log("Project saved");

} catch (err) {

    log("ERROR: " + err.toString());
    throw err;

} finally {

    try {
        app.endUndoGroup();

        app.project.save(new File(projectFile.fullName));
        $.sleep(300);

        log("Final save completed");

    } catch (e) {
        log("Final save error: " + e.toString());
    }

    log("RENDER FINISHED");
    logFile.close();

    app.quit();
}

})();
```

</details>

### Render.jsx — Script in Text Editor

![Render.jsx Script](/img/screenshots/figure-018.png)

:::info
The script uses the output module already configured in the AE project. Ensure the project default render settings are correct before running.
:::

**Log output:** `C:/ae-automated/exports/render_{template_slug}.txt`
