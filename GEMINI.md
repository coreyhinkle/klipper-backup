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
    - **Physical Harness/Umbilical:** Overhead wire/umbilical currently has slight slack compared to T0 (user plans to wire with steel tension wire to keep toolhead cables taut). This physical slack directly correlates with the minor ~2.5 µm OptoTap trigger variance observed at the far rear-left bed corner (`X=50, Y=275`).
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

## Resolved Issues & Bug Fixes

### 1. Toolchange Coordinate Corruption & Double Offset Application
- **Issue:** Legacy `_APPLY_ACTIVE_TOOL_GCODE_OFFSETS` (`SET_GCODE_OFFSET X=... Y=... Z=...`) in `tapchanger/homing.cfg` conflicted with upstream `klipper-toolchanger` (`ToolGcodeTransform`). When switching tools a second time (e.g. `T0 -> T1 -> T0`), offsets were applied twice, corrupting machine coordinates upon restoring gcode state.
- **Resolution:** Updated `tapchanger/homing.cfg` to adhere to modern upstream toolchanger specifications: eliminated legacy `SET_GCODE_OFFSET` and `_APPLY_ACTIVE_TOOL_GCODE_OFFSETS` macros, utilizing `ADJUST_Z_AFTER_TOOL_NOZZLE_HOME` after homing Z.

### 2. Premature Crash Detection Arming in `pickup_gcode`
- **Issue:** During tool pickup (`pickup_gcode` in `tapchanger/toolchanger-2.4.cfg`), `START_TOOL_PROBE_CRASH_DETECTION` and `M400` were being called inside the dock loop while the shuttle was resting on the dock rods at `Z: 0`. Any slight resistance or deflection during the vertical lift immediately triggered a crash detection halt (`Tool probe crash detected`).
- **Resolution:** Relocated `M400` and `START_TOOL_PROBE_CRASH_DETECTION` to immediately follow the unpark loop, arming crash detection only after the toolhead has safely lifted to `Z+8` at `Y+9.5` and cleared the dock pins.
- **Physical Verification:** Verified through 5 consecutive back-to-back toolchanges (`T0 ↔ T1 ↔ T0 ↔ T1 ↔ T0`) with zero false triggers, no coordinate drift, and clean shuttle docking/undocking.

---

## Kinematics & Speed Benchmarks

### Empirical Z-Axis Speed Ladder Test (24V Voron 2.4 350mm)
To establish maximum reliable Z travel speeds without microstep loss or motor stall, an automated rapid ladder routine was run using 5 full travel cycles ($Z 15 \leftrightarrow 200\text{ mm}$, 1.85 m total travel per step) with sub-micron optical OptoTap probe checks after each step:

| Z Velocity (mm/s) | Z Accel (mm/s²) | Stepper RPM | OptoTap Measured Drift ($\Delta Z$) | Result / Notes |
|:---|:---|:---|:---|:---|
| **30 mm/s** | 500 mm/s² | 225 RPM | `+0.0025 mm` | Pass (1 microstep, sensor baseline) |
| **35 mm/s** | 500 mm/s² | 262.5 RPM | `+0.0075 mm` | Pass (3 microsteps, well within tolerance) |
| **40 mm/s** | 500 mm/s² | 300 RPM | `+0.0025 mm` | Pass (1 microstep) |
| **45 mm/s** | 500 mm/s² | 337.5 RPM | `+0.0000 mm` | Pass (Exact zero drift) |
| **50 mm/s** | 500 mm/s² | 375 RPM | `+0.0050 mm` | Pass (2 microsteps) |
| **55 mm/s** | 500 mm/s² | 412.5 RPM | `+0.0025 mm` | Pass (1 microstep) |
| **60 mm/s** | 500 mm/s² | 450 RPM | `+0.0050 mm` | Pass (2 microsteps) |
| **65 mm/s** | 500 mm/s² | 487.5 RPM | `+0.0025 mm` | Pass (1 microstep, highest zero-loss speed) |
| **70 mm/s** | 500 mm/s² | 525 RPM | `+0.0100 mm` | Marginal Slip (4 microsteps, back-EMF onset) |
| **80 mm/s** | 500 mm/s² | 600 RPM | `+0.0125 mm` | Slip (5 microsteps, back-EMF torque loss) |

### Top-Frame Docks & Selection of 50 mm/s Production Setting
- **Top-Mounted Dock Mechanics:** Unlike gantry-mounted toolchangers, the StealthChanger docks are mounted to the upper frame extrusions at $Z \approx 298–299\text{ mm}$ (`params_park_z`). When changing tools during a print near the bed ($Z \approx 5\text{ mm}$), the gantry must climb $\approx 295\text{ mm}$ to the dock, swap, and dive $\approx 295\text{ mm}$ back down—totalling **$\approx 590\text{ mm}$ of vertical travel per toolchange**.
- **Empirical Toolchange Benchmark at Print Height ($Z = 5\text{ mm}$):**
  - **15 mm/s (Stock):** $\approx 46.2\text{ s}$ total (39.2s in pure vertical travel).
  - **35 mm/s (Option A):** Measured at **23.53 s** (16.8s pure vertical travel).
  - **50 mm/s (Option B - Production Selected):** Measured live at **20.11 s** (11.8s pure vertical travel).
  - **Net Gain:** 50 mm/s saves **26.1 seconds per toolchange** over stock (and an extra 3.42s over 35 mm/s). Over a 500-toolchange print, this saves **over 3.6 hours total**.
- **Quality & QGL Convergence at 50 mm/s:**
  - `QUAD_GANTRY_LEVEL` verified at 50 mm/s with `retry_tolerance: 0.0075 mm`.
  - Converged to **0.0050 mm (5 microns)** across all four corners.
- **Cross-Tool QGL Comparison (T0 vs T1):**
  - Probed plane tilt adjustments between T0 OptoTap and T1 OptoTap were **virtually identical down to 0.2 microns**:
    - Both converged with adjustments: `stepper_z = -0.000200 mm`, `stepper_z1 = -0.004500 mm`, `stepper_z2 = +0.000200 mm`, `stepper_z3 = +0.004500 mm`.
    - Confirms exceptional rigidity and zero tilt skew between the two toolhead mounts.

---

## Remaining Cleanups & Tuning Roadmap
See [`config_audit_and_todos.md`](file:///home/zyper/.gemini/antigravity-cli/brain/6e52ba8d-f478-4c25-ba15-7d58f0a28b79/config_audit_and_todos.md) for the complete audit:
1. **X/Y Speed Tuning:**
   - Stepper X and Y are currently conservative at `run_current: 0.8A`, `max_velocity: 300`, `max_accel: 3000`.
   - Planned tuning: Increase current to `1.0A`, test `max_velocity: 400 mm/s`, `max_accel: 5000–6000 mm/s²`.
2. **Toolchanger Transit Speeds:**
   - `params_fast_speed` is currently `10000` (166 mm/s); can be increased to `15000–18000` (250–300 mm/s) for faster tool swap transit across the bed.
3. **Cross-Tool Probe Repeatability Fine-Tuning:**
   - On T1, `samples_tolerance: 0.006` caused retries when single-microstep variance (0.0025 mm) occurred on consecutive taps. Consider relaxing to `0.008 mm` to eliminate unnecessary sampling retries while maintaining sub-10-micron precision.
4. **Configuration Cleanups:**
   - Remove duplicate `[resonance_tester]` in `ebb-sb-rp2040.cfg`.
   - Remove duplicate empty `[input_shaper]` in `printer.cfg`.
   - Equalize sensorless homing currents in `tapchanger/homing.cfg` (`_SENSORLESS_HOME_X`).
