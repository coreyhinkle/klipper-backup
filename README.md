# Voron 2.4 (350mm) StealthChanger 🚀

Configuration repository and comprehensive technical documentation for a **Voron 2.4 (350mm)** multi-tool 3D printer running the **StealthChanger / TapChanger** system.

Automated backups are powered by [Klipper-Backup](https://github.com/Staubgeborener/klipper-backup).

---

## 📖 Technical Documentation Index

All technical documentation has been organized into modular guides:

| Guide | Description |
|:---|:---|
| 🔧 **[Hardware & Electronics](docs/hardware.md)** | Machine specs, CM5 host, Manta M8P V2, Toolheads (T0 Rapido HF / T1 V6), OptoTap probes, and CAN bus star topology. |
| 🚀 **[Kinematics & Speed Benchmarks](docs/benchmarks.md)** | Empirical Z ladder test (15–80 mm/s), 50 mm/s qualification, top-dock travel math, and live stopwatch toolchange benchmarks. |
| 🔄 **[Toolchanger Architecture & Macros](docs/toolchanger.md)** | StealthChanger docking splines, top-frame park coordinates, macro suite (`T0`, `T1`, `UNSELECT_TOOL`), and operational conventions. |
| 🛠️ **[Troubleshooting & Bug Fixes](docs/troubleshooting.md)** | Root-cause write-ups and permanent fixes for toolchange coordinate corruption, pickup crash detection timing, and probe sample drift. |
| 📋 **[Tuning Roadmap & Audit Checklist](docs/roadmap.md)** | Completed milestones, active tasks (XY speed tuning to 400 mm/s, umbilical tension wire), and future optimizations. |

---

## ⚡ Quick Reference

### Network & Access
- **Printer Static IP:** `192.168.0.46/22` (persistent via NetworkManager `Chez CoreCats` profile)
- **Web Interfaces:** Mainsail / Moonraker on port `80` / `7125` (`http://192.168.0.46`)
- **CAN Bus:** Bitrate `1000000` (1 Mbps), `txqueuelen 128`

### Tuned Kinematics Baseline
- **`max_z_velocity`:** **`50.0 mm/s`** (3.3x speedup over 15 mm/s stock default)
- **`max_z_accel`:** **`500.0 mm/s²`**
- **Print-Height Toolchange Time:** **`20.11 seconds`** (measured live from $Z = 5\text{ mm}$ to top-frame docks and back, saving **26.1s per swap** over stock)
- **`QUAD_GANTRY_LEVEL`:** Converges cleanly within **`0.0050 mm` (5 microns)** at 50 mm/s

### Essential Toolchanger Commands
- `T0` / `T1` — Select Toolhead 0 (Purple) or Toolhead 1 (Green)
- `UNSELECT_TOOL` — Safely dock the active toolhead and return shuttle empty
- `TOOL_ALIGN_START` / `TOOL_ALIGN_TEST` / `TOOL_ALIGN_DONE` — Interactive docking inspection suite
- `QUAD_GANTRY_LEVEL` — Levels the flying gantry parallel to the bed (agrees within 0.2 µm across both tools)

---

## ⚠️ Active Hardware & Configuration Notes

> [!WARNING]
> **Temporary Fan Configuration (Electronics Bay Cooling):**
> - The skirt fans (`skirt1` on `PF6` and `skirt2` on `PF9`) are temporarily configured in [`printer.cfg`](printer_data/config/printer.cfg#L239-L251) as `[fan_generic]` instead of `[heater_fan]` to allow manual 100% cooling during offline calibration.
> - **Action Required:** Once a dedicated top-mount heatsink / cooler is installed on the **Raspberry Pi Compute Module 5 (CM5)**, revert `skirt1` and `skirt2` in [`printer.cfg`](printer_data/config/printer.cfg#L239-L251) back to their original configuration or permanently convert them to automated `[temperature_fan]` tracking `temperature_host` (target 50–55°C) so the CM5 remains cool regardless of whether the bed heater is active.

