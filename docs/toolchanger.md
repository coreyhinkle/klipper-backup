# Toolchanger Architecture & Operational Guide

This document describes the configuration, docking geometry, macros, and operational conventions for the **StealthChanger / TapChanger** multi-tool system implemented on this printer.

---

## 1. System Overview
- **Plugin:** `viesturz/klipper-toolchanger` (commit `94756df`).
- **Kinematics & Transformation:** Modern `ToolGcodeTransform` handles nozzle X/Y/Z offsets internally in Python, eliminating legacy `SET_GCODE_OFFSET` conflicts.
- **Dock Mounting:** Frame-mounted docks located on the top rear extrusion:
  - **Tool 0 (Purple):** Park position `X: 339.25, Y: 1.10, Z: 297.55` ([`sb-2.cfg`](../printer_data/config/sb-2.cfg))
  - **Tool 1 (Green):** Park position `X: 4.00, Y: 2.00, Z: 299.00` ([`ebb-sb-rp2040.cfg`](../printer_data/config/ebb-sb-rp2040.cfg))
- **Docking Paths:** Configured using the `rods_sb_path` spline in [`toolchanger-2.4.cfg`](../printer_data/config/tapchanger/toolchanger-2.4.cfg):
  ```ini
  params_type: 'rods_sb_path'
  params_rods_sb_path: [
    {'y': 9.5, 'z': 8},
    {'y': 9.5, 'z': 2},
    {'y': 5.5, 'z': 0},
    {'z': 0, 'y': 0, 'f': 0.5},
    {'z': -6, 'y': 0},
    {'z': -10, 'y': 3},
    {'z': -10, 'y': 16}
  ]
  ```

---

## 2. Speed & Safety Parameters
Defined in [`tapchanger/toolchanger-2.4.cfg`](../printer_data/config/tapchanger/toolchanger-2.4.cfg):
* **`params_safe_y`:** `90.0 mm` — Safe clearance distance in front of docks before initiating rapid cross-bed XY travels.
* **`params_close_y`:** `35.0 mm` — Close approach coordinate before engaging docking splines.
* **`params_fast_speed`:** `10000 mm/min` ($166.7\text{ mm/s}$) — Feedrate used for non-docking travels across the bed to/from the docks.
* **`params_path_speed`:** `600 mm/min` ($10.0\text{ mm/s}$) — Precision slow feedrate used while mating/unmating toolhead pins and pads.
* **Crash Detection:** Handled via `[tool_probe_endstop]`. Crash detection is armed **after** the toolhead lifts clear of the dock rods (`Z: +8` at `Y: 9.5`) to eliminate false triggers during pickup.

---

## 3. Operational Conventions

1. **Tool 0 is the Absolute Reference:**
   - `gcode_x_offset: 0`
   - `gcode_y_offset: 0`
   - `gcode_z_offset: 0`
   - All other tools are calibrated relative to T0. Tool 1 offsets are calibrated as:
     - `gcode_x_offset: -1.0`
     - `gcode_y_offset: 0.0`
     - `gcode_z_offset: 0.750`
2. **SpreadCycle Extrusion:**
   - Both extruders operate in pure SpreadCycle mode (`stealthchop_threshold: 0`) for maximum torque delivery and minimal MCU step-generation latency.
3. **Toolhead Lighting:**
   - T0 LEDs: `sb_leds_t0` (controlled via `stealthburner_leds_purple.cfg`)
   - T1 LEDs: `sb_leds_t1` (controlled via `stealthburner_leds_green.cfg`)
   - Status wrapper macros (`STATUS_READY`, `STATUS_HEATING`, `STATUS_PRINTING`) dynamically select the active toolhead LEDs.

---

## 4. Common Commands & Macros

### Tool Selection
* `T0` — Unselects active tool (if any) and mounts Toolhead 0.
* `T1` — Unselects active tool (if any) and mounts Toolhead 1.
* `UNSELECT_TOOL` — Parks the active tool in its dock and returns the shuttle empty.

### Alignment & Calibration Suite
* `TOOL_ALIGN_START` — Enters docking mode and brings the active toolhead to the dock approach point for manual inspection.
* `TOOL_ALIGN_TEST` — Tests docking engagement by dropping and re-picking the active toolhead.
* `TOOL_ALIGN_DONE` — Exits docking mode and restores normal operation.

### Maintenance
* `BELT_TUNING_POS` — Moves the gantry to the center of the chamber at a standard height for measuring belt tension.
* `CLEAN_NOZZLE` — Wipes the nozzle against the dock purge pad/brush.
