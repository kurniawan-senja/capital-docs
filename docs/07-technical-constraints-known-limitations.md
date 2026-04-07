---
id: technical-constraints
title: 7. Technical Constraints & Known Limitations
---

# 7. Technical Constraints & Known Limitations

## 7.1 Critical Render Constraints

:::danger CRITICAL
Nexrender and all headless renderers are NOT compatible. AfterFX.exe via evalFile is the only valid render method.
:::

| Constraint | Reason / Impact |
| --- | --- |
| No headless renderer (Nexrender) | AE expressions use `footage()`, `comp()`, and dropdown controls — incompatible with headless mode |
| No `throw new Error()` in expressions | Causes blank output in headless — must use fallback values instead |
| `footage()` requires exact name match | CSV filename in `footage()` must exactly match the item name in AE project panel |
| `totalRows` must equal `rowNames.length` | Never hardcode row count — derive dynamically from array length |
| Implicit return must be explicit | All AE expressions must end with explicit return: `var result = ...; result;` |
| Dynamic content via expressions only | All RW content is controlled by AE expressions — no external asset injection |

## 7.2 File Path Requirements

- Template path must use forward slashes after Normalize Path processes it
- RW file must be named `RW_*.aep` — the script validates this pattern
- Output directory is auto-created by `Render.jsx` if it does not exist
- `project_data.json` is always overwritten by `Export_Comp.jsx` on each run

## 7.3 Timing Considerations

| Wait Node | Duration & Reason |
| --- | --- |
| Wait (before Get Path) | 1 minute — allows sheet to fully update before reading |
| Wait Update (after RW Update) | 10 minutes — buffer for AE to complete RW replacement and save |

:::warning
If AE takes longer than the Wait duration, the workflow may read stale data. Monitor render times and adjust Wait durations for larger projects.
:::
