# Happy Hare's Cutter Only Presses Once — Fails to Cleanly Cut PETG (BunnyBox / Qidi Q2)

## Symptom

- Filament fails to feed during loading; the extruder motor skips/grinds rather than feeding smoothly.
- Pulling the filament out of the extruder (box side) shows a **melted, malformed tip** instead of a clean cut.
- Happens after a toolchange/eject — the filament wasn't actually severed by the cutter, so the uncut tip gets dragged back through the hot zone, where it keeps softening/melting instead of being cleanly removed.
- Most noticeable on tougher/more elastic materials like **PETG** — PLA may not show this at all.

## Root Cause

Confirmed by comparing Happy Hare's own cutter config against QIDI's stock cutter macros directly (not guessed).

The Q2-specific `_MMU_CUT_TIP_VARS` config that BunnyBox installs already has the cutter's physical pin location configured **correctly** — it matches QIDI's own stock `BOX_MOVE_TO_CUTTER` target exactly:

```
variable_pin_loc_xy            : 15, 0.5     # matches BOX_MOVE_TO_CUTTER's G1 X15 / G1 Y0.5
variable_pin_loc_compressed_xy : -0.5, 0.5   # matches BOX_DO_CUT's G1 X-0.5
variable_cutting_axis          : "x"
variable_gantry_servo_enabled  : False       # correct — Q2 has no gantry servo
```

So this was never a wrong-location/miscalibration problem. The actual mismatch is in the **cutting motion itself**:

**QIDI's own `BOX_DO_CUT`** (the macro the stock firmware's `CUT_FILAMENT` uses, proven reliable):
```
G1 X-0.5 F5000
G4 P500
G1 X1 F1000
G1 X-0.5 F5000     ← presses a second time
G4 P500
G1 X1 F1000
G4 P1000
G1 X15 F3000       ← evacuates back to the pin location
```
It presses the blade **twice** before evacuating.

**Happy Hare's generic `_CUT_TIP_DO_CUT_MOTION`** (used by the default `_MMU_CUT_TIP` macro) does this motion only **once** — a single fast-approach + slow-cut pass, no repeat.

On this specific cutter hardware, one press isn't always enough to fully sever PETG — it's reliably sufficient on easier materials, which is presumably why this wasn't caught generically, but shows up specifically here.

## Fix

Have Happy Hare's cut step call QIDI's own proven `BOX_DO_CUT` motion directly, instead of its generic single-press motion — but **don't** swap in the full `CUT_FILAMENT` wrapper macro. `CUT_FILAMENT` → `BOX_MOVE_TO_CUTTER` starts with a full `BOX_HOME_XY` re-home every time, which is an unwanted, risky extra step mid-toolchange (first attempt at this fix did exactly that, and caused the toolhead to re-home instead of just cutting). Keep Happy Hare's own positioning logic — it's already correctly configured — and swap only the actual cut-motion call.

In `mmu_cut_tip.cfg` (under `CONFIG_DIR/mmu/base/mmu_cut_tip.cfg`), the `[gcode_macro _MMU_CUT_TIP]` macro's **"Step 2 - Perform the cut"** section currently reads:

```
# Step 2 - Perform the cut
_CUT_TIP_ADJUST_CURRENT
_CUT_TIP_MOVE_TO_CUTTER_PIN PIN_PARK_X_LOC={pin_park_x_loc} PIN_PARK_Y_LOC={pin_park_y_loc}
_CUT_TIP_GANTRY_SERVO_DOWN
_CUT_TIP_DO_CUT_MOTION PIN_PARK_X_LOC={pin_park_x_loc} PIN_PARK_Y_LOC={pin_park_y_loc} RIP_LENGTH={rip_length}
_CUT_TIP_GANTRY_SERVO_UP
_CUT_TIP_RESTORE_CURRENT
_MMU_EVENT EVENT="filament_cut"
```

Replace just the cut-motion line (`_CUT_TIP_DO_CUT_MOTION ...`) with `BOX_DO_CUT`:

```
# Step 2 - Perform the cut
_CUT_TIP_ADJUST_CURRENT
_CUT_TIP_MOVE_TO_CUTTER_PIN PIN_PARK_X_LOC={pin_park_x_loc} PIN_PARK_Y_LOC={pin_park_y_loc}
_CUT_TIP_GANTRY_SERVO_DOWN
BOX_DO_CUT
_CUT_TIP_GANTRY_SERVO_UP
_CUT_TIP_RESTORE_CURRENT
_MMU_EVENT EVENT="filament_cut"
```

Then `FIRMWARE_RESTART`.

Everything else in the macro — current boost/restore, servo no-ops, the park/pushback steps that follow — is untouched. Only the physical press motion changes, from one press to QIDI's own proven two-press sequence, and the toolhead ends up at X15 either way (matches `pin_loc_xy`), so nothing downstream needs to change.

## Verification

Tested via `MMU_EJECT` → physically inspect the cut tip → reinsert → `MMU_LOAD` → `MMU_EJECT` → `MMU_LOAD` round trip. Tip came out cleanly cut (not torn/melted) retracted into the PTFE tubing, and all load/unload/eject cycles completed successfully.

## Caveat

`mmu_cut_tip.cfg` is a file BunnyBox's installer manages — a future reinstall or update could overwrite this edit back to the single-press default. If the melted-tip/failure-to-feed symptom ever reappears after updating BunnyBox, re-check this file first before assuming it's a new problem.
