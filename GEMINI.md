# GEMINI.md - Klipper Toolchanger Context

This file provides instructional context for AI agents interacting with this repository.

## Remote Interaction & Access
- **Source of Truth:** The printer itself is the source of truth. NEVER modify the local repository's `printer_data/` files directly without syncing to the printer, as local-only changes will be overwritten by `klipper-backup`.
- **Modification Workflow:** To apply configuration changes:
  1. Retrieve the target files from the printer using `scp` to a temporary local directory.
  2. Modify the files locally.
  3. Upload the modified files back to the printer's `~/printer_data/config/` directory using `scp` (overwriting the remote files).
  4. Wait a few seconds for `klipper-backup-filewatch.service` (or run `~/klipper-backup/script.sh -c "message"` via SSH) and run `git pull` in this local repository to verify they took effect.
- **SSH Access:** The AI agent has access to the printer's host machine via SSH (`192.168.0.46` / `voron.local`) from the local environment. Authentication is handled via an SSH key in `~/.ssh`.
- **Static IP Configuration:** `wlan0` is configured with a persistent static IP **`192.168.0.46/22`** (gateway `192.168.0.1`) via NetworkManager (`Chez CoreCats` profile) so it stays outside the router's DHCP range (which starts at `192.168.1.1`).
- **Klipper-Backup Auth:** Configured on the printer in `~/klipper-backup/.env` using `git_protocol="ssh"` with a repository-scoped GitHub Deploy Key (`~/.ssh/id_ed25519` on `voron`). `inotify-tools` is installed so `klipper-backup-filewatch.service` automatically commits and pushes config edits.

## Project Overview
This repository contains the Klipper configuration for a **Voron 2.4 (350mm)** (Formbot base kit) 3D printer utilizing the **StealthChanger/TapChanger** multi-tool system.

### Key Technologies & Hardware Inventory:
- **Firmware:** Klipper (`v0.13.0-583-g594ec7e1`) with the `viesturz/klipper-toolchanger` plugin (`94756df`).
- **Host:** **Raspberry Pi Compute Module 5 Rev 1.0** (Debian Bookworm `6.12.75+rpt-rpi-2712`, upgraded from earlier BTT CB1).
- **Control Board:** BTT Manta M8P V2.0 (`canbus_uuid: 3d2cb7337ea8`, acting as a USB-to-CAN bridge, TMC2209 drivers).
- **Toolhead MCUs & Hotends:**
  - **Tool 0 (T0 - Reference Tool, Purple StealthBurner):**
    - **MCU:** BTT EBB SB2209 RP2040 (`[mcu purple]`, `canbus_uuid: 4445b54fe351`).
    - **Hotend:** Phaetus Rapido HF (currently in HF configuration; user also has an uninstalled UHF upgrade kit on hand) with PT1000/PT100 via MAX31865 (`0.4mm` nozzle).
    - **Extruder:** Clockwork 2 (CW2) with Amazon BMG-style dual-gear kit (`B09WQYHBZY` / Formbot BMG clone gears, `rotation_distance: 4.58` direct value, equivalent to `22.9` with `50:10` gear ratio), TMC2209 (`0.65A`).
    - **Probe:** OptoTap (`^purple:gpio22`, `z_offset: -0.450`).
  - **Tool 1 (T1 - Green StealthBurner):**
    - **MCU:** BTT EBB SB2209 RP2040 (`[mcu EBBCan]`, `canbus_uuid: a7f6c7cc913a`, from original Formbot kit).
    - **Hotend:** Voron / V6 clone hotend & nozzle (from Formbot kit) with PT1000/PT100 via MAX31865 (`0.4mm` nozzle).
    - **Extruder:** Clockwork 2 (CW2) with Formbot kit / Amazon (`B09WQYHBZY`) BMG dual-gear hardware (`rotation_distance: 22.9`, `gear_ratio: 50:10`), TMC2209 (`0.65A`).
    - **Probe:** OptoTap (`^EBBCan:gpio22`, `z_offset: -0.80`).
- **Accessories:** Nevermore carbon filter (`PF7`), BTT Mini12864 RGB LCD display, Skirt & Exhaust fans.
- **Communication (CAN Bus):**
  - Bitrate: `1000000` (1M), managed by `systemd-networkd` (`/etc/systemd/network/25-can.network`).
  - Queue Length: **`txqueuelen 128`** via `/etc/udev/rules.d/10-can.rules`.
  - *Note on `txqueuelen` (128 vs 1024):* Do **NOT** change `txqueuelen` to `1024` or create a custom `can0.service`. Current official Klipper & Esoterical CANBus documentation explicitly recommend `txqueuelen 128` to prevent kernel buffer bloat that causes `"Timer too close"` errors.
- **Topology & Termination:**
  - Star/Hub topology using a shared distribution board.
  - To achieve the required **60-ohm** bus resistance, 120-ohm termination jumpers are **ON** for **Toolhead 0 (T0)** and the **Manta M8P** mainboard, and **OFF** for **Toolhead 1 (T1)**.

## Directory Structure & Key Files
- `printer_data/config/printer.cfg`: Main entry point. Contains hardware limits, pin aliases, and includes.
- `printer_data/config/sb-2.cfg`: Configuration for **Tool 0 (T0)**.
- `printer_data/config/ebb-sb-rp2040.cfg`: Configuration for **Tool 1 (T1)**.
- `printer_data/config/led_status.cfg`: Logic wrapper for tool-aware LED status macros (`STATUS_READY`, `STATUS_HEATING`, etc.).
- `printer_data/config/tapchanger/`:
  - `toolchanger-2.4.cfg`: Main toolchanger parameters, safety heights, and pickup/dropoff paths.
  - `macros.cfg`: `PRINT_START` / `PRINT_END` logic and filament handling.
  - `homing.cfg`: Homing overrides for safe toolchanger operation.
  - `tool_detection.cfg`: `[tool_probe_endstop]` crash detection configuration.

## Operational Conventions
- **T0 is the Reference Tool:** `gcode_x_offset`, `gcode_y_offset`, and `gcode_z_offset` for T0 should always be `0`. All other tools are calibrated relative to T0.
- **Extruder Mode:** Extruders run in **SpreadCycle** (`stealthchop_threshold: 0`) for maximum torque and reduced MCU latency.
- **LED Naming:**
  - T0 LEDs: `sb_leds_t0` (`stealthburner_leds_purple.cfg`)
  - T1 LEDs: `sb_leds_t1` (`stealthburner_leds_green.cfg`)

## Common Commands
- **Tool Selection:** `T0`, `T1`
- **Deselect Tool:** `UNSELECT_TOOL`
- **Alignment Suite:** `TOOL_ALIGN_START`, `TOOL_ALIGN_TEST`, `TOOL_ALIGN_DONE`
- **Maintenance:** `BELT_TUNING_POS`, `CLEAN_NOZZLE`

---

## Active Investigation / Known Issues (Second Toolchange Crash)

### Context from Logs (`klippy.log.2026-05-02` & `klipper-toolchanger` `94756df`)
When switching tools a second time (e.g., `T0 -> T1 -> T0` during a multi-color print or switching back to `T0` after finishing a print on `T1`), several bugs interact:

1. **Legacy `SET_GCODE_OFFSET` in `homing.cfg` Conflicts with `ToolGcodeTransform` in `toolchanger.py`**
   - Upstream `klipper-toolchanger` (commit `6188cd1` / `eb11cda` / `e8dbbdf`) replaced `SET_GCODE_OFFSET`-based tool offsets with an internal Python `ToolGcodeTransform` class.
   - `tapchanger/homing.cfg` still contains the legacy `_APPLY_ACTIVE_TOOL_GCODE_OFFSETS` (`SET_GCODE_OFFSET X=... Y=... Z=...`) and `_ADJUST_Z_HOME_FOR_TOOL_OFFSET` macros.
   - When `T1` is active or homed, applying `SET_GCODE_OFFSET` on top of `ToolGcodeTransform` **double-applies tool offsets**. Furthermore, `toolchanger.py`'s `_save_state()` / `_restore_state_and_transform()` saves `homing_origin` into `_toolchange_state` and restores it via `RESTORE_GCODE_STATE NAME=_toolchange_state MOVE=0`, corrupting coordinates on subsequent toolchanges.
   - **Fix:** Update `tapchanger/homing.cfg` to match upstream `klipper-toolchanger`: remove `SET_GCODE_OFFSET` / `_APPLY_ACTIVE_TOOL_GCODE_OFFSETS` / `_ADJUST_Z_HOME_FOR_TOOL_OFFSET`, and use `ADJUST_Z_AFTER_TOOL_NOZZLE_HOME` after `G28 Z`.

2. **Premature Z Restore in `pickup_gcode` (`toolchanger-2.4.cfg`)**
   - `[toolchanger]` sets `t_command_restore_axis: z` (which causes `toolchanger.py` to restore Z at the end of `select_tool`), **and** `pickup_gcode` in `toolchanger-2.4.cfg` also manually runs:
     ```gcode
     ROUNDED_G0 Y={tool.params_safe_y} F={fast} D=20
     {% if 'Z' in restore_position %}
       ROUNDED_G0 Z={restore_position.Z} F={fast} D=150
     {% endif %}
     ```
   - Because `restore_position` only contains `Z` (when `t_command_restore_axis: z`) and `X`/`Y` are not in `restore_position`, `pickup_gcode` drops the nozzle down to print height (`restore_position.Z`) while still parked at `Y=90, X=park_x` near the docks, before the slicer's next `G1 X... Y...` move!

3. **Post-Print `M18` + Dock Park Position (`PRINT_END` in `tapchanger/macros.cfg`)**
   - In `klippy.log.2026-05-02` (lines 9145–9509), `PRINT_END` parked `T1` at `X=4, Y=90, Z=300` (`X={tool.params_park_x}`) and ran `M18` (disabling motors and marking axes unhomed).
   - When `T0` was subsequently commanded, `on_axis_not_homed: home` triggered `G28` while the gantry was at `X=4, Z=300` directly in front of T1's dock, resulting in `No trigger on stepper_y after full movement` and an emergency stop during `G28 X`.

4. **Additional Config Cleanups Identified**
   - `tapchanger/macros.cfg`: `START_TOOL_PROBE_CRASH_DETECTION` and `SET_GCODE_VARIABLE MACRO=PRINT_START VARIABLE=printing VALUE=True` are commented out in `PRINT_START`, which causes `after_change_gcode` in `toolchanger-2.4.cfg` to immediately disable crash detection after every tool swap.
   - `tapchanger/homing.cfg`: `_SENSORLESS_HOME_X` lowers `stepper_x` current to `0.5A` during `G28 X` but forgets to lower `stepper_y` current (on CoreXY, both A and B motors drive X).
   - `ebb-sb-rp2040.cfg` & `printer.cfg`: Duplicate `[resonance_tester]` (with invalid `accel_chip: adxl345` in `ebb-sb-rp2040.cfg`), duplicate empty `[input_shaper]` sections in `printer.cfg`, and duplicate `LOAD_FILAMENT` / `UNLOAD_FILAMENT` macros.
