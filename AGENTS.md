# AGENTS.md - Multi-Agent Operational Context & System Architecture

This repository contains the configuration, macro suite, and documentation for a **Voron 2.4 (350mm)** multi-tool 3D printer running the **StealthChanger / TapChanger** system with **OptoTap**.

---

## 1. Golden Rule: Source of Truth & Safe Modification Protocol

> [!IMPORTANT]
> **The physical printer (`192.168.0.46:~/printer_data/config/`) is the SOLE live runtime source of truth.**
> Automated backups via `klipper-backup` push changes to this repository.
> Never directly edit `printer_data/config/` files on a workstation without deploying them to the printer via `scp` and verifying them.

### Safe Config Deployment Protocol:
1. **Retrieve / Inspect:** Inspect config from the printer via `scp zyper@192.168.0.46:~/printer_data/config/<file> ...` or directly via Moonraker API (`http://192.168.0.46:7125`).
2. **Edit & Review:** Make minimal, incremental edits.
3. **Deploy:** Upload to `zyper@192.168.0.46:~/printer_data/config/<file>`.
4. **Restart & Verify:** Run `curl -s -X POST "http://192.168.0.46:7125/printer/gcode/script" -H "Content-Type: application/json" -d '{"script":"FIRMWARE_RESTART"}'` and verify `{"result":{"state":"ready"}}`.
5. **Sync:** Let `klipper-backup-filewatch.service` auto-commit to git, or pull in the repo.

---

## 2. Machine Hardware & Topology

- **Printer:** Voron 2.4 (350x350x340 mm) CoreXY with Flying Gantry.
- **Host Controller:** Raspberry Pi Compute Module 5 (CM5) Rev 1.0 (Linux 6.12, static IP `192.168.0.46/22`).
- **Mainboard:** BTT Manta M8P V2.0 (`canbus_uuid: 3d2cb7337ea8`).
- **CAN Bus Network:** 1 Mbps, `txqueuelen 128` (star topology via distribution board; 60Ω total bus resistance).
- **Toolhead 0 (T0 - Master Reference, Purple):**
  - EBB SB2209 RP2040 (`canbus_uuid: 4445b54fe351`).
  - Phaetus Rapido HF (0.4mm nozzle), MAX31865 PT100/PT1000 RTD.
  - CW2 Extruder, SpreadCycle (`stealthchop_threshold: 0`).
  - OptoTap sensor on `^purple:gpio22` (`[tool_probe T0]`).
  - Coordinate reference: `gcode_x_offset: 0`, `gcode_y_offset: 0`, `gcode_z_offset: 0`.
  - Dock: Top front frame `X: 339.25, Y: 1.10, Z: 297.55`.
- **Toolhead 1 (T1 - Secondary, Green):**
  - EBB SB2209 RP2040 (`canbus_uuid: a7f6c7cc913a`).
  - V6 hotend (0.4mm nozzle), MAX31865 RTD.
  - CW2 Extruder, SpreadCycle (`stealthchop_threshold: 0`).
  - OptoTap sensor on `^EBBCan:gpio22` (`[tool_probe T1]`).
  - Calibrated Offsets relative to T0: `gcode_x_offset: 1.447`, `gcode_y_offset: -2.236`, `gcode_z_offset: 0.750` (residual optical error $<50\ \mu\text{m}$).
  - Dock: Top front frame `X: 4.00, Y: 2.00, Z: 299.00`.
- **Enclosure:** **DraftChange front door extenders** installed. Hinges and frame spacers extend the front doors forward, clearing the front docks ($Y \approx 1\text{--}2\text{ mm}$, $Z \approx 298\text{ mm}$) and forward-shifted carriage shuttle.

---

## 3. Critical Upstream Quirks & Traps (Must Know!)

### A. KTC-Easy & Virtual Probe Homing
- The system runs **`jwellman80/klipper-toolchanger-easy` (KTC-Easy)** tracking `main` in `~/klipper-toolchanger-easy/`.
- Toolchanger configurations are organized under `printer_data/config/toolchanger/`:
  - `readonly-configs/` (symlinked to KTC-Easy examples for clean Moonraker updates).
  - `tools/` (active tools like `T0.cfg`; inactive/isolated tools named `T*.cfg.disabled`).
  - `toolchanger-config.cfg` (user-level overrides for dock coordinates, paths, and homing parameters).
- In Klipper `v0.13.0-800+` (`5cbebdc3`) with KTC-Easy's updated probe API, `G28 Z` **automatically applies `probe_z_offset`** during virtual endstop homing.
- **NEVER** use `SET_KINEMATIC_POSITION Z={... + probe_z_offset}` in homing macros. That will double-apply the offset and drive the nozzle into the build plate!

### B. Never Run `SAVE_CONFIG` after `PROBE_CALIBRATE`
- In `klipper-toolchanger`, each tool probe reads its offset from its own section (`[tool_probe T0]` in `sb-2.cfg` and `[tool_probe T1]` in `ebb-sb-rp2040.cfg`).
- Standard `SAVE_CONFIG` saves `z_offset` under `#*# [tool_probe_endstop]`, where **`klipper-toolchanger` completely ignores it**.
- Always manually write calibrated Z offsets into `[tool_probe T0]` inside `sb-2.cfg`.

### C. OptoTap Deadband Warning
- OptoTap only trips after the nozzle touches the bed AND the carriage slides up the linear rail against its magnets by the full trigger travel (~0.45–0.81 mm).
- Between physical contact and electrical trigger, `QUERY_PROBE` reports `0` (`open`). Never assume the nozzle is clear of the bed just because `QUERY_PROBE` is open.

### D. Bed Mesh Bounds
- Standard Voron 350mm mesh bounds are **`mesh_min: 40, 40`** and **`mesh_max: 310, 310`** with `fade_start: 1.0` / `fade_end: 10.0`.
- Do NOT probe at `X=5, Y=340` (the rear-left corner), where extreme umbilical cable stretch pulls the OptoTap carriage upward in mid-air and causes `"Probe triggered prior to movement"`.

---

## 4. Key G-Code Macros & Commands

- `T0` / `T1`: Select active tool.
- `UNSELECT_TOOL`: Park active tool into its dock and return empty carriage.
- `G32`: Levels gantry via `QUAD_GANTRY_LEVEL` and homes Z.
- `PRINT_START`: Hardened to always initialize toolchanger, preheat T0, home, and select T0 for QGL and adaptive mesh.
- `PRINT_END`: Parks nozzle safely at rear center (`X=175, Y=300`), runs 3-minute Nevermore venting, and turns off heaters.
- `CANCEL_PRINT`: Configured via `_CLIENT_VARIABLE` to disarm crash detection, reset `PRINT_START.printing = False`, and park at rear center.

---

## 5. Network Endpoints & Credentials

- **Printer IP:** `192.168.0.46` (port 80 for Mainsail, port 7125 for Moonraker).
- **SSH Access:** `ssh zyper@192.168.0.46` (passwordless key access configured).
- **Documentation:** Full technical guides live in [`docs/`](docs/) (`hardware.md`, `toolchanger.md`, `benchmarks.md`, `troubleshooting.md`, `roadmap.md`).

---

## 6. Slicer Profiles & Integration

Tuned PrusaSlicer and SuperSlicer profiles recovered from the Windows workstation are stored in:
- **Directory**: \`slicer_profiles/\`
- **Documentation & Review Context**: \`slicer_profiles/README.md\`
- **Key Profiles**:
  - \`slicer_profiles/prusa_windows_backup/printer/VORON V2 350mm - 0.4mm Nozzle (High Flow) - Custom.ini\` (Active dual-extruder StealthChanger configuration).
  - \`slicer_profiles/prusa_windows_backup/print/0.20mm Standard @VORON2 - HF0.4mm - Copy.ini\` (Active high-speed Arachne profile).
  - \`slicer_profiles/whistle_voron_full_sliced_config.ini\` (Full configuration dump from actual Voron slice).
