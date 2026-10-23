# Qidi Q2 + BunnyBox/Happy Hare + HelixScreen: Install & Calibration Guide (Plus4 Hub)

This guide walks through installing BunnyBox (Happy Hare MMU) and HelixScreen on a Qidi Q2 using the [ChanceVegas/Qidi-Q2-superuser](https://github.com/ChanceVegas/Qidi-Q2-superuser) AIO installer, and documents every real-world problem hit along the way — including a defective stock hub that required swapping to a **Plus4 hub** with an onboard encoder. If you're repeating this on another Q2 with the same Plus4 hub, you can likely skip straight to [Step 7](#step-7-plus4-hub-specific-config).

## Prerequisites

- Qidi Q2 running the legacy `mks` firmware layout (not the 1.1.2/`qidi` layout — the AIO installer blocks mutating actions on that layout).
- Qidi Box (AMS-style unit) with a Plus4 replacement hub installed, or planning to install one.
- SSH access to the printer.

---

## Step 1: Fix the printer's `apt` sources (if installs fail with 404s)

Qidi's stock image points `apt` at a regional Debian mirror. Debian 11 (bullseye) has aged out of normal support, and mirrors have started pruning old `bullseye-security`/`bullseye-backports` binaries while their index files go stale — `apt-get update` partially fails, then any `apt-get install` 404s on packages that *look* available in the index but no longer exist in the mirror's pool.

**Fix** — point at the permanent archive instead of a live mirror:

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo sed -i \
  -e 's|mirrors.ustc.edu.cn/debian-security|archive.debian.org/debian-security|' \
  -e 's|mirrors.ustc.edu.cn/debian|archive.debian.org/debian|' \
  -e '/bullseye-backports/d' \
  /etc/apt/sources.list

echo 'Acquire::Check-Valid-Until "false";' | sudo tee /etc/apt/apt.conf.d/99no-check-valid-until
sudo apt-get update
sudo apt-get install -y git   # sanity check — should now install cleanly
```

(Adjust the mirror hostname in the `sed` commands to whatever your `sources.list` actually points at.)

## Step 2: Fix GitHub connectivity issues (if DNS/connections fail)

Some networks have unreliable or selectively-poisoned DNS for `github.com` and its subdomains (`raw.githubusercontent.com`, `codeload.github.com`). Symptoms: `Could not resolve host`, or the installer just hangs with no output.

**Quick diagnosis:**
```bash
curl -v --connect-timeout 8 --max-time 20 -fsSL https://raw.githubusercontent.com/<any-known-good-path> -o /tmp/test.sh
```
- Fails fast with "Could not resolve host" → DNS problem.
- Hangs the full timeout → connection/TLS problem, not DNS.

**Fix for DNS resolution** — add known-good IPs to `/etc/hosts` (safe: HTTPS certificate validation still protects you even with a hardcoded IP):
```bash
sudo tee -a /etc/hosts <<'EOF'
140.82.114.4 github.com
185.199.108.133 raw.githubusercontent.com
140.82.113.9 codeload.github.com
EOF
```
(These IPs can rotate occasionally — re-resolve via `https://cloudflare-dns.com/dns-query?name=<host>&type=A` with the `accept: application/dns-json` header if they stop working months later.)

**Alternative if the launch command itself hangs** — use jsdelivr's GitHub mirror instead of `raw.githubusercontent.com` directly:
```bash
bash <(curl -fsSL https://cdn.jsdelivr.net/gh/Camden-Winder/Qidi-Q2-superuser@main/Q2/aio_menu.sh)
```

## Step 3: Run the AIO installer

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/Camden-Winder/Qidi-Q2-superuser/main/Q2/aio_menu.sh)
```
Select **option 1: Install BunnyBox & HelixScreen**. This delegates the BunnyBox install to `Camden-Winder/Bunny-Box`'s installer, which in turn clones `Wazzup77/Happy-Hare` (branch `bunnybox`) and runs its own `install.sh`.

If `git` wasn't already on the image, this is where Step 1 matters — the dependency check will try to `apt-get install git`.

## Step 4: Gear calibration (gate 0)

`MMU_CALIBRATE_GEAR` does **not** drive filament itself — it only records a measurement. The actual test feed is a separate command:

```
MMU_SELECT GATE=0
MMU_TEST_MOVE MOVE=50          # repeat until filament is visible past the hub/exit reference point
```
Cut the filament flush with the reference point, then do the real test move:
```
MMU_TEST_MOVE MOVE=100
```
Physically measure the emitted filament with calipers/a ruler, then record the *real* measured value:
```
MMU_CALIBRATE_GEAR MEASURED=<your measured mm>
```
This seeds every gate's `rotation_distance` with gate 0's value as a baseline — not a full per-gate calibration, but enough to pass Happy Hare's "essential calibration" check and start loading. See [Step 10](#step-10-encoder--autotune) for finishing per-gate accuracy via autotune instead of repeating this manually for gates 1-3.

> **Caution:** Don't use `MMU_TEST_MOVE` as a substitute for real load/unload — it's diagnostic-only and never updates Happy Hare's filament-position state, which will desync it from physical reality (see Step 5).

## Step 5: State-recovery issues you'll likely hit

These come from Happy Hare's internal state machine (`filament_pos`, per-gate `AVAILABLE` status) disagreeing with physical reality — common right after fresh installs or manual `MMU_TEST_MOVE` use.

| Symptom | Cause | Fix |
|---|---|---|
| `Operation not possible, filament is loaded` (when nothing is loaded) | Stale internal state | `MMU_RECOVER LOADED=0` |
| `Gate 0 is empty` on `T0` (right after filament was just detected/parked) | The no-gear-sensor preload path detects/parks filament but doesn't flip the gate's `AVAILABLE` flag | `MMU_CHECK_GATE GATE=0` (or `MMU_GATE_MAP GATE=0 AVAILABLE=1` to force it) |
| Filament preload shows `Loading... Parking...` — this is normal | Pre-gate sensor trips → Happy Hare does a confirm-load then parks back — by design | Not a bug; `MMU_STATUS` to confirm gate shows available afterward |

## Step 6: Known software bug — `MMU_SENSORS` crashes Klipper

On this installed version of the `bunnybox` branch, `MMU_SENSORS` throws:
```
AttributeError: 'MmuSensorManager' object has no attribute '_get_all_sensors'
```
This is a genuine code defect (version skew in `mmu_sensor_manager.py`), not anything you did, and it will crash Klipper (`Internal error... shutdown`) every time. Recover with `FIRMWARE_RESTART`.

**Workaround** — Happy Hare auto-registers each sensor as a standard Klipper object, so query them directly instead:
```
QUERY_FILAMENT_SENSOR SENSOR=mmu_gate_sensor
QUERY_FILAMENT_SENSOR SENSOR=mmu_pre_gate_0
```
or get everything at once:
```
QUERY_ENDSTOPS
```

## Step 7: Plus4 hub-specific config

If your stock hub's shared gate sensor (`mmu_gate`, on `gate_switch_pin` in `mmu_hardware.cfg`) is unreliable — symptoms: bowden calibration's forward pass succeeds, but the retract/unload phase always fails with `did not home to sensor 'mmu_gate'` even with a large `gate_homing_max`, and `QUERY_FILAMENT_SENSOR SENSOR=mmu_gate_sensor` reports "detected" even with **zero filament anywhere near it** — that's a mechanically stuck switch (sticks after first trigger, doesn't release), not a config or calibration problem. Swapping to a **Plus4 hub** replaces this switch (and adds a real rotary encoder on the same signal pin).

Config changes for the Plus4 hub, in `mmu_hardware.cfg`:

```
# The tension-switch pin is repurposed as the encoder signal on the Plus4 hub — disable it:
#sync_feedback_tension_pin: ^mmu:PB0

# Gate sensor stays on its original (correct) inverted polarity — the old stuck switch
# just made it look wrong; the wiring convention itself was fine:
gate_switch_pin: ^!mmu:PB1

# Enable the encoder on the now-freed pin:
[mmu_encoder mmu_encoder]
encoder_pin: ^mmu:PB0
encoder_resolution: 3.0    # starter value — overwritten by calibration below
```

`FIRMWARE_RESTART` after any hardware config change.

> **Diagnostic tip for future reference:** if you ever suspect a sensor's polarity is simply backwards (vs. physically stuck), it's cheap to test: temporarily remove/add the `!` modifier on the pin, restart, and press/release the switch by hand a few times. If it now toggles cleanly and repeatably, it was a polarity issue. If it's still stuck on one value no matter what, the switch itself is bad — inverting a signal can't fix a switch that never actually changes state.

## Step 8: Calibrate the encoder

```
MMU_CALIBRATE_ENCODER
```
This measures actual resolution (e.g. `3.1621` vs the `3.0` placeholder) from repeated load/unload counts and saves it.

## Step 9: Bowden calibration

```
MMU_RECOVER LOADED=0
MMU_CALIBRATE_BOWDEN BOWDEN_LENGTH=3500
```
With a working hub and encoder in place, this should complete cleanly. Example successful output:
```
Filament homed to extruder after 1089.5mm movement
(encoder measured 1049.4mm)
Calibrated bowden length is 1095.5mm. Recommended flowguard_encoder_max_motion (clog detection length): 21.9mm
Calibrated bowden length 1095.5mm has been saved for all gates
```
Note the encoder catching a real ~40mm discrepancy between commanded gear movement and actual filament movement — this is exactly what the encoder is for: without it, Happy Hare would have silently trusted the gear's commanded distance even if the filament were slipping (e.g. from a dragging spool — see note below).

> **Also check your spool holders for drag.** A spool that doesn't spin freely (too small a diameter for the holder, binding on an axle, etc.) can cause the gear to lose real distance during the long bowden retract specifically, since that move is a blind/calculated distance with no live sensor feedback mid-move (only the encoder, if enabled, can catch it after the fact). A 3D-printed spacer to properly seat the spool fixed this on our build.

## Step 10: Encoder + autotune

With the encoder calibrated, enable autotune in `mmu_parameters.cfg` instead of manually repeating gear calibration for gates 1-3:
```
autotune_rotation_distance: 1
```
Happy Hare will now continuously fine-tune each gate's `rotation_distance` from encoder feedback during real use, rather than requiring a manual `MMU_TEST_MOVE` → measure → `MMU_CALIBRATE_GEAR` pass per gate.

The encoder also unlocks:
- **Slip detection** during the bowden retract (`Excess slippage was detected in bowden tube unload` warnings instead of silent mismeasurement).
- **`MMU_CALIBRATE_GATES`** for automated multi-gate rotation-distance calibration.
- **Clog/runout detection (FlowGuard)** during printing, monitoring real filament movement vs. commanded.

## Step 11: Verify other gates

Load filament into each remaining slot by hand. Each slot has its own `pre_gate` sensor; insertion should trigger the same `Loading... Parking...` sequence (confirm reaching the shared hub sensor, then retract to the parked position) — this is generic per-gate logic, not special-cased to gate 0. After parking, filament should sit a small, consistent distance back from the hub body in that slot's own PTFE tube — this is the normal **UNLOADED** ("parked in gate") state, not a fault. "Loaded" specifically means filament has advanced *past* the hub through the shared bowden into the extruder/toolhead — a distinct, later operation (triggered by `T0`-`T3` or `MMU_LOAD`), not something that happens on mere insertion.

Confirm actual state anytime with:
```
MMU_STATUS
```

---

## Quick troubleshooting reference

| Command | Purpose |
|---|---|
| `MMU_STATUS` | Shows what Happy Hare currently believes about gate/tool/filament state |
| `MMU_RECOVER LOADED=0` | Force-clear a stale "filament loaded" state without moving anything |
| `MMU_CHECK_GATE GATE=<n>` | Physically re-inspect a gate and correct its availability flag |
| `QUERY_FILAMENT_SENSOR SENSOR=<name>` | Direct Klipper sensor query — use instead of the broken `MMU_SENSORS` |
| `QUERY_ENDSTOPS` | One-shot dump of every endstop's live state, MMU sensors included |
| `MMU_TEST_MOVE MOVE=<mm>` | Raw diagnostic gear move — **never** a substitute for real load/unload |

---

*Written up from a live troubleshooting session on a Qidi Q2 with a Plus4 hub replacement. If something here doesn't match your hardware revision, the Happy Hare upstream docs are at [github.com/moggieuk/Happy-Hare/wiki](https://github.com/moggieuk/Happy-Hare/wiki).*
