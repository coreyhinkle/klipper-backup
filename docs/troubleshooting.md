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
2. **Software Bridge:** Relaxing `samples_tolerance` on `[tool_probe T1]` from `0.006` to `0.008 mm` (8 microns) eliminates retry loops while preserving sub-10-micron precision.
