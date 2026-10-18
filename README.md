# Klipper-Backup 💾 
Klipper backup script for manual or automated GitHub backups 

This backup is provided by [Klipper-Backup](https://github.com/Staubgeborener/klipper-backup).

## Hardware Notes
- **Printer:** Voron 2.4 (350mm, Formbot base kit) with StealthChanger / TapChanger
- **Host:** Raspberry Pi Compute Module 5 Rev 1.0 (Static IP: `192.168.0.46/22`)
- **Mainboard:** BTT Manta M8P V2.0 (USB-to-CAN bridge, `1000000` bitrate, `txqueuelen 128`)
- **Toolheads (StealthBurner + CW2 on both):**
  - **T0 (`purple`, `sb-2.cfg`):** BTT EBB SB2209 RP2040, Phaetus Rapido HF (0.4mm nozzle; UHF upgrade kit available on hand), CW2 with BMG dual-gear kit (Amazon `B09WQYHBZY` / Formbot, `rotation_distance: 4.58`), OptoTap
  - **T1 (`EBBCan`, `ebb-sb-rp2040.cfg`):** BTT EBB SB2209 RP2040 (from Formbot kit), Formbot Voron/V6 clone hotend & 0.4mm nozzle, CW2 with BMG dual-gear kit (Formbot / Amazon `B09WQYHBZY`, `rotation_distance: 22.9`, `gear_ratio: 50:10`), OptoTap
- **CAN Bus Termination:** Toolhead 0 (T0) has the 120-ohm resistor jumper installed. Toolhead 1 (T1) does not. The Manta M8P mainboard also has the 120-ohm resistor jumper installed, resulting in the required 60-ohm total bus resistance.

## Tuned Kinematics & Speed Benchmarks
- **Z-Axis Speed & Acceleration:**
  - `max_z_velocity: 35.0` mm/s (up from 15.0 mm/s stock default, 2.3x speedup)
  - `max_z_accel: 500.0` mm/s² (up from 350.0 mm/s²)
  - **Empirically Benchmarked:** Tested via high-speed vertical ladder routines (15 mm/s to 80 mm/s) using sub-micron optical OptoTap probe drift checks. Zero lost steps measured up to 65 mm/s; first microstep slip onset at 70 mm/s (525 RPM), full back-EMF slip at 80 mm/s (600 RPM).
  - **Selected Baseline (Option A):** 35 mm/s @ 500 mm/s² captures 90% of travel time reduction for typical Voron 2.4 Z-hops (e.g. 8mm toolchange hop takes 0.29s vs 0.26s at 50 mm/s) while preserving a robust 45% torque safety margin below the 24V back-EMF limit.
  - Fully verified with `QUAD_GANTRY_LEVEL` converging within 0.0045 mm.
- **X/Y Speeds:**
  - Currently conservative at `max_velocity: 300` mm/s, `max_accel: 3000` mm/s² (X/Y tuning roadmap: 400 mm/s @ 5000–6000 mm/s² with `run_current` raised to 1.0A).

## Documentation & Debugging Context
See [`GEMINI.md`](./GEMINI.md) for full architecture notes, operational conventions, empirical benchmark tables, and resolved bug fixes (including toolchanger crash detection and offset transformation fixes).
