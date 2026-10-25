# GEMINI.md - Klipper Toolchanger AI Operational Context

This file provides core operational rules, remote interaction protocols, and system context for AI agents working in this repository.

---

## 1. Remote Interaction & Source of Truth Rules

> [!IMPORTANT]
> **The physical printer (`192.168.0.46:~/printer_data/config/`) is the SOLE source of truth.**
> NEVER modify files in `printer_data/config/` directly inside the local workstation repository, as automated backups from `klipper-backup` will overwrite them.

### Configuration Modification Workflow:
1. **Retrieve:** Use `scp` to copy target configuration files from `zyper@192.168.0.46:~/printer_data/config/<file>` to a local scratch directory (`scratch/<file>`).
2. **Modify:** Make and review edits locally.
3. **Deploy:** Upload modified files back to the printer via `scp scratch/<file> zyper@192.168.0.46:~/printer_data/config/<file>`.
4. **Backup & Sync:** Let `klipper-backup-filewatch.service` auto-commit (or run `~/klipper-backup/script.sh -c "commit message"` via SSH), then run `git pull` in this workstation repository.

### Network & Authentication:
- **Host Access:** Passwordless SSH and `sudo` access are configured for `zyper@192.168.0.46` (`voron.local`).
- **Static IP:** `wlan0` is permanently configured at **`192.168.0.46/22`** (gateway `192.168.0.1`) via NetworkManager (`Chez CoreCats` profile) to remain outside the router's DHCP range (`192.168.1.1+`).
- **Backup Service:** `klipper-backup` uses an SSH deploy key (`~/.ssh/id_ed25519` on `voron`) pushed to `git@github.com:coreyhinkle/klipper-backup.git`.
- **Preserved Documentation:** `~/klipper-backup/script.sh` on the printer is configured to preserve `README.md`, `GEMINI.md`, `*.md`, and the `docs/` directory during backup pull/push cycles.

---

## 2. Technical Documentation Index

For detailed technical specifications and data tables, consult the modular documentation in `docs/`:

- **Hardware & Electronics:** See [`docs/hardware.md`](docs/hardware.md) for CM5 host specs, Manta M8P V2 wiring, CAN bus star topology, 60-ohm termination jumper settings, and toolhead specs.
- **Kinematics & Speed Benchmarks:** See [`docs/benchmarks.md`](docs/benchmarks.md) for the empirical Z speed ladder data (15 to 80 mm/s), 24V back-EMF physics, top-dock 600mm travel math, stopwatch toolchange benchmarks, and QGL verification.
- **Toolchanger Architecture:** See [`docs/toolchanger.md`](docs/toolchanger.md) for StealthChanger docking splines, park coordinates, macros, and operational conventions.
- **Troubleshooting & Fixes:** See [`docs/troubleshooting.md`](docs/troubleshooting.md) for root-cause analyses on resolved toolchange bugs (offset doubling, premature pickup crash detection, and OptoTap cable drag).
- **Tuning Roadmap & Audit:** See [`docs/roadmap.md`](docs/roadmap.md) for prioritized upcoming tasks and speed optimization targets.

---

## 3. Key Operational Conventions

1. **Reference Tool:** Toolhead 0 (T0 - Purple Rapido HF) is the absolute coordinate reference (`gcode_x_offset: 0`, `gcode_y_offset: 0`, `gcode_z_offset: 0`). Toolhead 1 (T1 - Green V6) offsets are applied relative to T0 (`gcode_x_offset: 1.447`, `gcode_y_offset: -2.236`, `gcode_z_offset: 0.750`), calibrated optically with $<50\ \mu\text{m}$ residual error.
2. **SpreadCycle Mode:** All extruder steppers run with `stealthchop_threshold: 0` for maximum torque and minimal step-generation latency.
3. **Probe Clearance:** When running manual probe commands (`PROBE`, `PROBE_ACCURACY`), always retract to `Z >= 10` immediately afterward to prevent restarting Klipper while the OptoTap nozzle is mechanically depressed against the bed.
4. **CAN Bus Queue Length:** `txqueuelen` must remain **`128`** via `/etc/udev/rules.d/10-can.rules`. Never change to 1024.

---

## 4. Current Machine Status & Active Baseline
- **Firmware:** Klipper `v0.13.0-800-g9cb99754` with `jwellman80/klipper-toolchanger-easy` (`main` branch, KTC-Easy).
- **Toolchanger Configs:** Standardized in `printer_data/config/toolchanger/` (`readonly-configs/`, `tools/T0.cfg`, `toolchanger-config.cfg`).
- **Z-Axis Speed Baseline:** `max_z_velocity: 50.0 mm/s`, `max_z_accel: 500.0 mm/s²`.
- **Leveling:** `QUAD_GANTRY_LEVEL` qualified at 50 mm/s to `0.0050 mm` tolerance.
- **Enclosure:** DraftChange front door extenders installed (clears front-mounted StealthChanger docks at $Y \approx 1\text{--}2\text{ mm}$ and forward-shifted carriage).
- **Toolhead Offsets:** Optically verified with sub-50 µm residual error across repetitive docking cycles.
- **Active State:** System homed, parked safely at bed center ($X=175, Y=175, Z=80$).
