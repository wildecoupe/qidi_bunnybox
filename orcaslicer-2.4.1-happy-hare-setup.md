# OrcaSlicer 2.4.1 Setup for Happy Hare / BunnyBox (Qidi Q2)

Settings checklist for configuring OrcaSlicer 2.4.1 to drive a Happy Hare MMU (BunnyBox) as a single-extruder-multi-material (SEMM) setup — i.e. one nozzle fed by multiple gates, selectable per print and usable for true multi-color.

> Field names below are current OrcaSlicer 2.4.1 terminology, verified against the official OrcaSlicer wiki — not guessed from a different slicer's UI.

## 1. Printer Settings → Multimaterial

| Setting | Value | Why |
|---|---|---|
| **Single Extruder Multi Material** | Enabled | Tells Orca this is a single physical nozzle fed by a multi-material unit (your MMU) — it will emit `Tn` tool-change commands instead of assuming separate physical nozzles. |
| **Extruders** | Leave as-is (likely locked to `1`) | This field means *physical extruders* (e.g. IDEX printers), not MMU gate count. It isn't the "number of slots" setting — don't try to force it to 4. |
| **Manual Filament Change** | Disabled | This is for M600/pause-based manual swaps. Happy Hare automates the tool change via G-code, so leave this off. |

**Where your 4 gates actually come from:** not a printer setting at all — in the main Plater window, use the `+` next to the filament color swatches at the top to add filament entries until you have 4 (one per gate). Orca generates `T0`-`T3` from however many filaments exist in the project, the same mechanism it uses for Bambu's AMS (also single-nozzle, multi-feed).

## 2. Printer Settings → Custom G-code

| Field | Content |
|---|---|
| **Change filament G-code** | `T[next_extruder]` |
| **Machine start G-code** | See [Section 4](#4-full-start-g-code-replacement) below — replace the whole box |
| **Machine end G-code** | Add `MMU_END` as the *first* line, before your existing end-print macro |
| **Layer change G-code** | Add `_MMU_UPDATE_HEIGHT` (this is the field that fires *after* the Z-lift — not "Before layer change G-code", which fires before it) |

## 3. Turning off double tip-forming

Happy Hare's own cutter/tip-forming macros handle filament tip prep on a tool change. If Orca *also* does a retract-and-cool sequence before each `Tn`, you get it done twice (causing blobs/jams). Zero out:
- **Filament Settings → Advanced** — cooling tube length/retraction distances for toolchange
- **Printer Settings → Extruder N → Retraction when switching material** — set lengths to `0`

**Verify it actually took effect** by slicing and inspecting the exported G-code around a `Tn` line — it should go straight from end-of-layer into temperature/park commands and `Tn`, with no `G1 ... E-15`-style retraction immediately before it. If you see that, tip-forming is still happening in the slicer too.

Leave the **wipe tower** (Multimaterial Settings → Prime Tower) **enabled** unless you've confirmed BunnyBox's own nozzle-clean macros (`bunnybox_macros.cfg`) are handling purge disposal some other way.

## 4. Full Start G-code replacement

Based on your current Machine start G-code, here's the same script with Happy Hare's required setup/check/load steps inserted in the correct order (setup → check → your existing print-start call → load the initial tool → *then* your purge/prime-line logic, since that needs filament actually in the nozzle first). Nothing about your existing `PRINT_START` call or purge-line logic was changed — just reordered and the raw `T[initial_tool]` line was replaced with the proper Happy Hare macro that uses the tool/color/material info `MMU_START_SETUP` just parsed out.

Replace your entire **Machine start G-code** box with:

```gcode
MMU_START_SETUP INITIAL_TOOL=[initial_tool] TOTAL_TOOLCHANGES=!total_toolchanges! REFERENCED_TOOLS=!referenced_tools! TOOL_COLORS=!colors! TOOL_TEMPS=!temperatures! TOOL_MATERIALS=!materials! FILAMENT_NAMES=!filament_names! PURGE_VOLUMES=!purge_volumes!
MMU_START_CHECK
PRINT_START BED=[bed_temperature_initial_layer_single] HOTEND=[nozzle_temperature_initial_layer] CHAMBER=[chamber_temperature] EXTRUDER=[initial_no_support_extruder]
MMU_START_LOAD_INITIAL_TOOL
M83
M140 S[bed_temperature_initial_layer_single]
M104 S[nozzle_temperature_initial_layer]
M141 S[chamber_temperature]
G4 P3000
G1 X108 Y1 F30000
G0 Z[initial_layer_print_height] F600
G0 X128 E8 F{outer_wall_volumetric_speed/(24/20) * 60}
G0 X133 E.3742 F{outer_wall_volumetric_speed/(0.3*0.5)/4 * 60}
G0 X138 E.3742 F{outer_wall_volumetric_speed/(0.3*0.5) * 60}
G0 X143 E.3742 F{outer_wall_volumetric_speed/(0.3*0.5)/4 * 60}
G0 X148 E.3742 F{outer_wall_volumetric_speed/(0.3*0.5) * 60}
G0 X153 E.3742 F{outer_wall_volumetric_speed/(0.3*0.5)/4  * 60}
G1 X154 Z-0.1
G1 X158
G1 Z1 F600
G1 X108 Y2.5 F30000
G0 Z[initial_layer_print_height] F600
G0 X128 E10 F{outer_wall_volumetric_speed/(24/20) * 60}
G0 X133 E.3742 F{outer_wall_volumetric_speed/(0.3*0.5)/4 * 60}
G0 X138 E.3742 F{outer_wall_volumetric_speed/(0.3*0.5) * 60}
G0 X143 E.3742 F{outer_wall_volumetric_speed/(0.3*0.5)/4 * 60}
G0 X148 E.3742 F{outer_wall_volumetric_speed/(0.3*0.5) * 60}
G0 X153 E.3742 F{outer_wall_volumetric_speed/(0.3*0.5)/4 * 60}
G1 X154 Z-0.1
G1 X158
G1 Z1 F600
SET_PRINT_STATS_INFO TOTAL_LAYER=[total_layer_count]
```

**What changed vs. your original, and why:**
1. Added `MMU_START_SETUP ...` as the very first line — parses the slicer's tool/color/material/purge-volume data into Happy Hare's "Slicer Tool Map" before anything else happens.
2. Added `MMU_START_CHECK` right after — confirms filament is actually available in every gate the print needs *before* homing/heating wastes time on a print that can't complete.
3. Kept your `PRINT_START ...` call as-is, right where it was.
4. Replaced the bare `T[initial_tool]` with `MMU_START_LOAD_INITIAL_TOOL` — this performs the actual initial tool load using the data `MMU_START_SETUP` already parsed, rather than a raw, context-free tool change. This is also why it now happens *after* `PRINT_START`: your original script loaded filament before homing/heating ran, which works but means the MMU gate-availability check wasn't done first.
5. Everything from `M83` through the two purge/prime-line blocks is **unchanged**, just shifted down to run after the tool is actually loaded (filament has to be in the nozzle before you can purge/prime with it).
6. Moved `SET_PRINT_STATS_INFO TOTAL_LAYER=[total_layer_count]` to the end, matching Happy Hare's recommended pattern — this is a minor, non-functional reorder, not a required fix.

## 5. Machine end G-code

Add as the first line, before whatever you currently have:
```gcode
MMU_END
```

## 6. Choosing the slot

- **Single color from a specific gate**: assign one filament to one of the slots in the top filament bar, then right-click the object/plate and set its extruder to that slot number.
- **Multi-color**: assign different filaments to different slots, then use Orca's multi-material painting brush (or split the model into parts/modifiers) to assign different extruder numbers to different regions. Orca emits `Tn` changes automatically at the right points.

## 7. After slicing, verify

```
MMU_SLICER_TOOL_MAP PURGE_MAP=1
```
on the printer console after uploading — confirms Happy Hare correctly parsed which gates/tools/colors the sliced file actually references.
