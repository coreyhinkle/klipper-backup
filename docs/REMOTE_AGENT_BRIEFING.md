# Remote Agent Complete System Briefing: Voron 2.4 StealthChanger Multi-Tool

> **Target Audience:** Any AI assistant, agent, or subagent interacting with Corey's 3D printing setup, repository (`coreyhinkle/klipper-backup`), or physical hardware (`voron.local` / `192.168.0.46`).

---

## 1. Machine Identity & Architecture
- **Printer Model:** Voron 2.4 (350mm build envelope: `X=350, Y=350, Z=340 mm`) CoreXY with Flying Gantry.
- **Multi-Tool Architecture:** **StealthChanger / TapChanger** (by Viesturs Zariņš / `klipper-toolchanger`) using kinematic shuttles on top-front frame-mounted docks.
- **Enclosure:** **DraftChange front door extenders** installed (hinge and perimeter spacers extend the front acrylic doors forward by ~30–40mm to clear the front-mounted docks at $Y \approx 1\text{--}2\text{ mm}$ and forward-shifted carriage).
- **Z-Probing:** **OptoTap** on each toolhead (`^gpio22` on toolhead RP2040 MCUs). The nozzle itself acts as the probe by physically contacting the bed and lifting the carriage along a linear rail against magnetic preloads to interrupt an optical sensor.

---

## 2. Electronics, Compute & Network Topology
- **Host System:** Raspberry Pi Compute Module 5 (CM5) Rev 1.0 (Linux kernel 6.12, static LAN IP `192.168.0.46/22`, gateway `192.168.0.1`).
- **Mainboard:** BigTreeTech (BTT) Manta M8P V2.0 (`canbus_uuid: 3d2cb7337ea8`).
- **CAN Bus Network:**
  - Standardized at **1 Mbps (1000000 baud)**.
  - **CRITICAL:** Queue length is strictly set to **`txqueuelen 128`** via `/etc/udev/rules.d/10-can.rules`. Never change this to 1024 or create conflicting custom systemd services.
  - Total bus resistance is **60Ω** ($120\Omega \parallel 120\Omega$ with termination jumpers ON on Mainboard and Toolhead 0, and OFF on Toolhead 1).

---

## 3. Toolhead Specifications & Offsets

### Toolhead 0 (T0 - Master Coordinate Reference, Purple StealthBurner):
- **MCU:** BTT EBB SB2209 RP2040 (`[mcu purple]`, `canbus_uuid: 4445b54fe351`).
- **Hotend:** Phaetus Rapido HF (0.4mm nozzle), MAX31865 PT100/PT1000 RTD (`purple:gpio9`), `max_temp: 300°C`.
- **Extruder:** Clockwork 2 (CW2) direct drive (`rotation_distance: 22.9`, `gear_ratio: 50:10`, TMC2209 in SpreadCycle mode `stealthchop_threshold: 0`).
- **Dock Position:** `X: 339.25, Y: 1.10, Z: 297.55` ([`sb-2.cfg`](../printer_data/config/sb-2.cfg)).
- **Coordinate Offsets:** **`gcode_x_offset: 0`**, **`gcode_y_offset: 0`**, **`gcode_z_offset: 0`** (T0 is the absolute machine reference).
- **Z Probe:** OptoTap on `^purple:gpio22`.
- **Input Shaper:** MZV (`shaper_freq_x: 62.4`, `shaper_freq_y: 88.6`).

### Toolhead 1 (T1 - Secondary Tool, Green StealthBurner):
- **MCU:** BTT EBB SB2209 RP2040 (`[mcu EBBCan]`, `canbus_uuid: a7f6c7cc913a`).
- **Hotend:** Voron / V6 clone (0.4mm nozzle), MAX31865 RTD (`EBBCan:gpio9`), `max_temp: 260°C`.
- **Extruder:** Clockwork 2 (CW2) direct drive (`rotation_distance: 22.9`, `gear_ratio: 50:10`, SpreadCycle mode).
- **Dock Position:** `X: 4.00, Y: 2.00, Z: 299.00` ([`ebb-sb-rp2040.cfg`](../printer_data/config/ebb-sb-rp2040.cfg)).
- **Calibrated Offsets relative to T0:**
  - `gcode_x_offset: 1.447`
  - `gcode_y_offset: -2.236`
  - `gcode_z_offset: 0.750`
  - *Note:* Optical alignment verified via upward-facing camera with sub-50 µm residual error.
- **Z Probe:** OptoTap on `^EBBCan:gpio22`.
- **Input Shaper:** MZV (`shaper_freq_x: 60.1`, `shaper_freq_y: 72.3`).

---

## 4. Software Baseline & Critical Quirks (MUST KNOW)

### A. Firmware Versions:
- **Klipper:** `v0.13.0-800-g9cb99754`.
- **Toolchanger Plugin:** `viesturz/klipper-toolchanger` patched with PR #174 (`0bded50`) to support modern Klipper probe interfaces.

### B. Virtual Probe Homing (`probe:z_virtual_endstop`):
- In Klipper `v0.13.0-800+` (commit `5cbebdc3`), `G28 Z` **automatically applies `probe_z_offset`** directly during the homing move.
- **Trap Avoided:** Do NOT use legacy homing correction macros that invoke `SET_KINEMATIC_POSITION Z={... + probe_z_offset}`. This double-applies the probe offset and drives the nozzle into the build plate!
- In [`tapchanger/homing.cfg`](../printer_data/config/tapchanger/homing.cfg), `_ADJUST_Z_HOME_FOR_TOOL_OFFSET` uses `klipper-toolchanger`'s native command `ADJUST_Z_AFTER_TOOL_NOZZLE_HOME`.

### C. `PROBE_CALIBRATE` & `SAVE_CONFIG` Trap:
- In `klipper-toolchanger`, each tool probe reads its Z offset strictly from its own configuration block (`[tool_probe T0]` in `sb-2.cfg` and `[tool_probe T1]` in `ebb-sb-rp2040.cfg`).
- Standard Klipper `SAVE_CONFIG` writes `z_offset` under `#*# [tool_probe_endstop]`, which `klipper-toolchanger` **completely ignores**.
- **Rule:** Never rely on `SAVE_CONFIG` for tool probe offsets. Calibrated values must be manually written into `[tool_probe T0]` inside `sb-2.cfg`.

### D. OptoTap Deadband Physics:
- OptoTap has mechanical overtravel (~0.45mm to ~0.81mm).
- When the nozzle touches the bed, `QUERY_PROBE` **still reads `open` (0)** until the gantry moves down far enough to trip the optical flag.
- Never rely on `QUERY_PROBE` reading `open` to conclude that the nozzle is safely above the bed.

### E. Bed Mesh Geometry & Top-Frame Docks:
- `[bed_mesh]` in [`printer.cfg`](../printer_data/config/printer.cfg) is configured with `mesh_min: 40, 40` and `mesh_max: 310, 310`.
- **Why:** Probing at extreme rear-left `X=5, Y=340` causes excessive umbilical harness tension, pulling the toolhead upward on its OptoTap rail and triggering `"Probe triggered prior to movement"`.
- **Mesh Fade:** Configured with `fade_start: 1.0` and `fade_end: 10.0`. This guarantees that by the time the gantry lifts to the docks at `Z ≈ 298 mm`, bed mesh Z compensation has faded to strictly **`0.000 mm`**, preventing dock alignment deviation.

### F. Print Cancel & Safe Parking:
- Mainsail's `CANCEL_PRINT` macro is hooked into `_CLIENT_VARIABLE` (`_ON_PRINT_CANCEL`) to immediately call `STOP_TOOL_PROBE_CRASH_DETECTION` and reset `PRINT_START.printing = False`.
- Toolhead parks safely at the rear center (`X=175, Y=300`) with a 10mm hop (`variable_custom_park_dz: 10.0`).

---

## 5. Operations & Macros Summary

| Macro | Description |
|:---|:---|
| `T0` / `T1` | Mounts specified toolhead, restores gcode offsets, arms crash detection. |
| `UNSELECT_TOOL` | Safely parks current tool in dock and returns carriage empty. |
| `G32` | Clears mesh, levels gantry (`QUAD_GANTRY_LEVEL`), and homes Z. |
| `PRINT_START` | Initializes toolchanger, preheats T0, homes XYZ, selects master tool T0 for QGL and adaptive mesh, preheats hotends, and purges via KAMP. |
| `PRINT_END` | Disarms crash detection, retracts filament, lifts nozzle, parks at rear center (`X=175, Y=300`), vents chamber via Nevermore for 3 minutes, and powers down heaters. |
| `LOAD_FILAMENT` / `UNLOAD_FILAMENT` | Direct-drive Clockwork 2 load/unload routines with carriage mount safety checks. |
| `CALIBRATE_TOOL_SHAPER` | Multi-tool ADXL resonance calibration using native Klipper `CHIPS=` parameter (`adxl345 T{tool}`). |

---

## 6. Access Protocols & Source of Truth

- **Source of Truth:** Live runtime files reside on `zyper@192.168.0.46:~/printer_data/config/`.
- **Backup Git Repository:** `git@github.com:coreyhinkle/klipper-backup.git` (tracked locally on workstations under `klipper-backup/`).
- **REST / WebSocket API:** Moonraker API available at `http://192.168.0.46:7125/` (Mainsail UI on port `80`).
- **Detailed Reference Guides:**
  - [`docs/hardware.md`](hardware.md) — Electronics, wiring, termination jumpers, CM5 host.
  - [`docs/benchmarks.md`](benchmarks.md) — Z-axis 50mm/s tuning, CoreXY 400mm/s velocity ladder, toolchange timing.
  - [`docs/toolchanger.md`](toolchanger.md) — Docking paths, speed profiles, optical offset qualification.
  - [`docs/troubleshooting.md`](troubleshooting.md) — Forensic root-cause history (coordinate corruption, OptoTap false crash, Klipper v0.13.0-800 homing double-offset).
  - [`docs/roadmap.md`](roadmap.md) — Active roadmap, upcoming optimizations, and hardware upgrades.
