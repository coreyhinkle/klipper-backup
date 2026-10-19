# Troubleshooting & Bug Fixes History

This document details root-cause analyses and resolutions for critical issues resolved on this printer.

---

## 1. Toolchange Coordinate Corruption & Double Offset Application

### Symptom:
When switching tools a second time during multi-material prints or after printing on Tool 1 (e.g. `T0 → T1 → T0`), the gantry would crash or jump to incorrect coordinates.

### Root Cause:
* Upstream `klipper-toolchanger` (commits `6188cd1` / `eb11cda`) replaced legacy `SET_GCODE_OFFSET`-based tool offsets with an internal Python `ToolGcodeTransform` class.
* [`tapchanger/homing.cfg`](../printer_data/config/tapchanger/homing.cfg) still contained legacy `_APPLY_ACTIVE_TOOL_GCODE_OFFSETS` macros invoking `SET_GCODE_OFFSET X=... Y=... Z=...`.
* When T1 was active, applying `SET_GCODE_OFFSET` on top of `ToolGcodeTransform` **double-applied tool offsets**. Additionally, `toolchanger.py` saved `homing_origin` into `_toolchange_state` and restored it via `RESTORE_GCODE_STATE NAME=_toolchange_state MOVE=0`, corrupting machine coordinates on subsequent toolchanges.

### Resolution:
Updated [`tapchanger/homing.cfg`](../printer_data/config/tapchanger/homing.cfg) to match modern upstream specifications:
* Removed legacy `SET_GCODE_OFFSET`, `_APPLY_ACTIVE_TOOL_GCODE_OFFSETS`, and `_ADJUST_Z_HOME_FOR_TOOL_OFFSET` macros.
* Utilized upstream `ADJUST_Z_AFTER_TOOL_NOZZLE_HOME` after homing Z.

---

## 2. Premature Crash Detection Arming in `pickup_gcode`

### Symptom:
During tool pickup (`pickup_gcode` in [`tapchanger/toolchanger-2.4.cfg`](../printer_data/config/tapchanger/toolchanger-2.4.cfg)), tool pickup would randomly halt with an emergency stop:
`Tool probe crash detected`.

### Root Cause:
Inside `pickup_gcode`, `START_TOOL_PROBE_CRASH_DETECTION` and `M400` were being called inside the unparking loop while the shuttle was still resting on the dock rods at `Z: 0`. Any microscopic compliance or deflection as the shuttle lifted off the rods immediately tripped the OptoTap sensor, causing an erroneous crash detection shutdown.

### Resolution:
Relocated `M400` and `START_TOOL_PROBE_CRASH_DETECTION` to immediately follow the unpark loop, ensuring crash detection is only armed after the toolhead has safely lifted to `Z: +8` at `Y: 9.5` and fully cleared the dock pins.
* **Verification:** Verified across 5 consecutive back-to-back toolchanges (`T0 ↔ T1 ↔ T0 ↔ T1 ↔ T0`) with zero false triggers.

---

## 3. Post-Print `M18` and Dock Park Collision Risk

### Symptom:
In historical logs (`klippy.log.2026-05-02`), `PRINT_END` parked T1 at `X=4, Y=90, Z=300` directly in front of T1's dock and executed `M18` (disabling all stepper motors). When a subsequent print was commanded, `on_axis_not_homed: home` triggered `G28` while the gantry was parked inches from the dock pins, leading to `No trigger on stepper_y after full movement` and a crash during `G28 X`.

### Resolution:
Updated `PRINT_END` logic in [`tapchanger/macros.cfg`](../printer_data/config/tapchanger/macros.cfg) to park the toolhead safely away from the docking zone before turning off steppers.

---

## 4. Probe Samples Exceeding Tolerance on T1 at Rear-Left Corner

### Symptom:
During `QUAD_GANTRY_LEVEL` with Toolhead 1, probing the rear-left corner (`X=50, Y=275`) repeatedly triggered:
`Probe samples exceed tolerance. Retrying...`

### Root Cause:
* OptoTap measures carriage displacement against magnetic preloads.
* T1's overhead umbilical wiring currently has slight physical slack compared to T0.
* At the far rear-left corner (maximum distance from the umbilical anchor), the loose cable creates minor variable vertical drag, altering the nozzle contact force and shifting consecutive readings by $0.0025\text{ mm}$ (1 microstep).
* Because [`ebb-sb-rp2040.cfg`](../printer_data/config/ebb-sb-rp2040.cfg#L113) set an ultra-strict `samples_tolerance: 0.006` (6 microns), a single microstep shift triggered retry loops.

### Resolution:
1. **Hardware Fix:** Planned installation of overhead metal tension wire to maintain consistent cable tension across both toolheads.
2. **Software Bridge:** Standardized `samples_tolerance: 0.008 mm` (8 microns) on both `[tool_probe T0]` and `[tool_probe T1]`, completely eliminating retry loops while preserving sub-10-micron precision.

---

## 5. Klipper `v0.13.0-800` + PR #174 Z-Offset Double Application & Post-Mesh Overwrite

### Symptom:
After upgrading Klipper to `v0.13.0-800` and merging `klipper-toolchanger` PR #174 (`0bded50`), the KAMP purge line (`Z=0.8`) printed cleanly, but as soon as Layer 1 started (`Z=0.2`), the nozzle lowered into the PEI build plate and triggered OptoTap crash detection (`Toolhead collision or probe trigger detected! Pausing print.`).

### Root Cause:
1. **Upstream Klipper & Toolchanger Change (`5cbebdc3` / `5a56830a` / `0bded50`):**
   - Prior to Klipper `5cbebdc3` and PR #174, `EndstopRouter.get_position_endstop()` returned `0.0`, so `G28 Z` did **not** apply `probe_z_offset`. [`_ADJUST_Z_HOME_FOR_TOOL_OFFSET`](../printer_data/config/tapchanger/homing.cfg) had to manually apply `probe_z_offset` via `SET_KINEMATIC_POSITION Z={80.0 + tool_z_offset + probe_z_offset}`.
   - In Klipper `v0.13.0-800` + PR #174 (`probe.HomingViaProbeHelper`), `G28 Z` runs a native probe session and **automatically applies `probe_z_offset` (`-0.450 mm`) during `G28 Z`**.
2. **Bug #1 — Double-Application of `probe_z_offset` in `_ADJUST_Z_HOME_FOR_TOOL_OFFSET`:**
   - Running `SET_KINEMATIC_POSITION Z={80.0 + tool_z_offset + probe_z_offset}` after `G28 Z` applied `probe_z_offset` (`-0.450 mm`) a **second time**, shifting Klipper's Z coordinate system so the true bed surface read at $Z = -0.485\text{ mm}$ during `BED_MESH_CALIBRATE ADAPTIVE=1`.
3. **Bug #2 — `SET_KINEMATIC_POSITION` After Mesh Activation in `TOOL_BED_MESH_CALIBRATE`:**
   - Immediately after `BED_MESH_CALIBRATE ADAPTIVE=1` activated the $-0.485\text{ mm}$ mesh, `TOOL_BED_MESH_CALIBRATE` ran `G0 Z10` (moving kinematic Z to $10.0 - 0.485 = 9.515\text{ mm}$, physically $10.000\text{ mm}$ above the bed) and then called `SET_KINEMATIC_POSITION Z={10.0 + tool_z_offset}` (`10.000 mm`) while the mesh was active.
   - Overwriting kinematic Z from `9.515` to `10.000` shifted G-code Z to $10.485\text{ mm}$ when the nozzle was physically at $10.000\text{ mm}$, driving all subsequent G-code Z moves **$0.485\text{ mm}$ lower than commanded** ($Z=0.8$ purge at $+0.315\text{ mm}$; $Z=0.2$ first layer at **$-0.285\text{ mm}$**).

### Resolution:
* Updated [`_ADJUST_Z_HOME_FOR_TOOL_OFFSET`](../printer_data/config/tapchanger/homing.cfg) to call upstream `ADJUST_Z_AFTER_TOOL_NOZZLE_HOME` (which only applies `tool.gcode_z_offset` and never re-applies `probe_z_offset`).
* Updated [`TOOL_BED_MESH_CALIBRATE`](../printer_data/config/tapchanger/homing.cfg) so `SET_KINEMATIC_POSITION` is skipped when `tool_z_offset == 0.0` (T0) and, if ever invoked with a non-zero offset tool, adjusts relative to `printer.toolhead.position.z` so active bed mesh compensation is never overwritten.
* Removed the dead/ignored `#*# [tool_probe_endstop] z_offset = -0.810` block from `SAVE_CONFIG` in [`printer.cfg`](../printer_data/config/printer.cfg) (`ToolProbeEndstop` does not read `z_offset`; each `[tool_probe T0]` / `[tool_probe T1]` reads `z_offset` from its own section).

---

## 6. Post-Upgrade Config Audit Hardening (`v0.13.0-800`)

### Additional Traps Identified & Resolved:
1. **Guaranteed Master Reference Tool (`T0`) for `PRINT_START` Probing ([`tapchanger/macros.cfg`](../printer_data/config/tapchanger/macros.cfg)):**
   - Previously, if a job started while `T1` was left mounted on the carriage from a prior print, `M104 S150` before `INITIALIZE_TOOLCHANGER` heated `T0` in the dock and `G32` / `TOOL_BED_MESH_CALIBRATE` probed with `T1`. `PRINT_START` now initializes the toolchanger first and explicitly selects `T0` after `G28` so QGL, Z homing, and bed meshing always use the calibrated master probe on `T0`.
2. **Bed Mesh Bounds (`40, 40` to `310, 310`) & Fade Out Before Top-Docking Height ([`printer.cfg`](../printer_data/config/printer.cfg)):**
   - Updated `mesh_min: 40, 40` and `mesh_max: 310, 310` (standard Voron 2.4 350mm specification) from `5, 5` / `340, 340`. Probing at the extreme rear-left corner (`X=5.0, Y=340.0`) stretched T0's umbilical cable tight enough to pull the OptoTap carriage upward in mid-air and trip `"Probe triggered prior to movement"`.
   - Added `fade_start: 1.0` and `fade_end: 10.0` (and matched `speed: 400` to `max_velocity`) in `[bed_mesh]`. Previously, `fade_end` defaulted to `0.0` (disabled), causing front-edge bed mesh Z offsets to remain active at `Z = 297.55–299.0 mm` during tool docking/undocking.
3. **Crash Detection & State Cleanup on `CANCEL_PRINT` ([`printer.cfg`](../printer_data/config/printer.cfg)):**
   - Added `[gcode_macro _CLIENT_VARIABLE]` with `variable_user_cancel_macro: "_ON_PRINT_CANCEL"` to disarm `STOP_TOOL_PROBE_CRASH_DETECTION` and reset `PRINT_START.printing = False` when a print is cancelled, and configured `PAUSE` to park safely at the rear center (`X=175, Y=300, dz=10`).
4. **PID Oscillation Tolerance in `pickup_gcode` ([`tapchanger/toolchanger-2.4.cfg`](../printer_data/config/tapchanger/toolchanger-2.4.cfg)):**
   - Updated `TEMPERATURE_WAIT` during tool pickup to use `SENSOR={tool.extruder}` with a `1.0°C` tolerance (`MINIMUM={[printer[tool.extruder].target - 1.0, 0]|max}`), preventing multi-second stalls inside the dock when PID hovers at `0.1°C` below target.

