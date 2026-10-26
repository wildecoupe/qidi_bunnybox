# HelixScreen "Cancel" Doesn't Stop a Print Started While It's Still Preparing

## Symptom

- Tap **Cancel** on HelixScreen's touchscreen while the printer is still in its pre-print phase (homing, heating, bed probing — before the actual print moves begin).
- HelixScreen acts like it worked: target temperatures drop to 0°C, no error shown.
- The printer keeps going anyway — finishes the bed probe, heats back up, and the real print starts right on schedule.
- Tapping Cancel again *after* the real print has visibly started (slicer progress / layer count showing) does work.
- Cancelling from Mainsail (or any other Moonraker client) works correctly regardless of phase.

This is **not** specific to BunnyBox or Happy Hare — it's a HelixScreen behavior that's easy to hit on this printer because the Qidi Q2's `PRINT_START` macro does a lot of work (homing, heating, bed probing, MMU initial tool load) before the real print begins, giving a wide window where it's reachable.

## Root Cause

Confirmed directly from a live `journalctl -u helixscreen -f` capture while reproducing it, cross-referenced against HelixScreen's source (`src/ui/print_control_buttons.cpp`).

The actual log sequence when Cancel is tapped during preparing:

```
[PrintControl] Stop confirmed while preparing - cancelling the start
[PrinterPrintState] Retiring preparing job '...': cancelled
[USER] Print cancelled
[Moonraker API] Setting extruder temperature to 0°C
[Moonraker API] Setting heater_bed temperature to 0°C
[PrintComplete] Start ended as cancelled - heaters dropped
[MoonrakerManager] PRINT_START collector stopped (retired without a print)
[Print Cancel] Hiding modal (programmatic)
...
[PrintStartNav] Auto-navigating to print status (print job active)
[MoonrakerManager] print_state transition: 0 -> 1 ...
[PrinterPrintState] Slicer progress active (M73 detected)
```

HelixScreen's Cancel confirmation callback branches on whether the current job is still "preparing" (i.e. hasn't been handed over to the printer as a real tracked job yet):

```cpp
// A job we are still preparing may not exist on the printer yet: a
// host-side pre-start block runs before the job is handed over, so
// print_stats is idle or still holds the PREVIOUS job. Routing that
// through AbortManager would send CANCEL_PRINT to an idle printer,
// which its own state watcher reads as an immediate success
// while the queued start_print fires anyway once the macro returns.
if (PrintControlButtons::instance().current_view().stop_retires_preparing) {
    state.print_state().retire_preparing(helix::PreparingExit::Cancelled);
    return;
}
// otherwise:
helix::AbortManager::instance().start_abort();
```

When `stop_retires_preparing` is true, Cancel takes the `retire_preparing()` path instead of the real `AbortManager`/`CANCEL_PRINT` path. That path only updates HelixScreen's own bookkeeping and zeroes target temperatures through the Moonraker API — it has no mechanism to interrupt a `PRINT_START` macro that's already running on Klipper's gcode queue. Once dispatched, Klipper runs that macro to completion regardless of what any UI client does afterward, and the slicer's actual print gcode follows immediately after, which is exactly what the log shows: `print_state transition: 0 -> 1` and `Slicer progress active` right behind the "cancelled" messages.

The second tap (after the real print had started) correctly took the `AbortManager::start_abort()` branch instead, which does work.

**Note:** HelixScreen's own documentation (`docs/user/TROUBLESHOOTING.md`, "Print preparing" section) states: *"Cancel is always available during preparation, and canceling there is a clean cancel, not a failed print: the print never starts..."* — that did not hold in this reproduction. Worth flagging as a discrepancy between documented and actual behavior, not just an edge case.

## Workaround

- **During the preparing phase** (homing/heating/bed probing, before the real print visibly starts): don't rely on a single tap of HelixScreen's Cancel. Either:
  - Cancel from Mainsail/Fluidd instead (unaffected — always goes through the real `printer.print.cancel` API), or
  - Run `CANCEL_PRINT` directly from the Klipper console, or
  - Wait for the real print to visibly start (layer/progress showing), then tap Cancel again — it works correctly once that happens.
- **If you're not sure whether a preparing-phase cancel actually took effect**, check Mainsail's print status directly rather than trusting HelixScreen's screen — in this reproduction HelixScreen showed "cancelled" messaging while the printer kept heating and probing underneath it.

## Upstream Status

Filed against `prestonbrown/helixscreen` — see [issue #1890](https://github.com/prestonbrown/helixscreen/issues/1890). Not something fixable from the BunnyBox/Happy Hare side; it's a HelixScreen UI/architecture gap in how it distinguishes a "preparing" job from a real tracked print job.
