# Tuning Roadmap & Optimization Checklist

This document tracks completed milestones, active tuning tasks, and future optimization opportunities for the Voron 2.4 StealthChanger.

---

## 1. Speed & Acceleration Optimization Matrix

| Parameter | Current Setting | Target Setting | Tuning Plan & Status |
|:---|:---|:---|:---|
| **`max_z_velocity`** | `50.0 mm/s` | `50.0 mm/s` | **Completed & Qualified.** Slashes vertical travel time to top-frame docks to 11.8s. |
| **`max_z_accel`** | `500 mm/s²` | `500 mm/s²` | **Completed & Qualified.** Zero step loss, QGL converges to 5 microns. |
| **`max_velocity` (XY)** | `400 mm/s` | `400 mm/s` | **Completed & Qualified.** 8-cycle diagonal torture test passed in 9.47s with zero step loss. |
| **`max_accel` (XY)** | `5000 mm/s²` | `5000 mm/s²` | **Completed & Qualified.** Input shaper frequencies cleanly support 5000 accel. |
| **`params_fast_speed`** | `15000 mm/min` ($250\text{ mm/s}$) | `15000 mm/min` | **Completed & Qualified.** Slashes toolchange duration to 18.90s. |
| **`run_current` (XY)** | `1.00 A` | `1.00 A` | **Completed & Qualified.** Provides high-speed torque margin on 24V bus. |

---

## 2. Todo Checklist

### Completed Milestones
- [x] Configure persistent static IP `192.168.0.46/22` outside router DHCP range.
- [x] Standardize CAN bus on `txqueuelen 128` and remove conflicting services.
- [x] Configure automated GitHub backups via `klipper-backup` with deploy key.
- [x] Eliminate toolchange coordinate corruption and double-offset application.
- [x] Fix premature crash detection in `pickup_gcode` during unpark lift.
- [x] Fix `PRINT_END` post-print dock parking collision risk.
- [x] Empirically benchmark Z-axis speed ladder from 15 mm/s up to 80 mm/s.
- [x] Qualify `max_z_velocity: 50.0 mm/s` and verify QGL convergence to 5 microns.
- [x] Verify cross-tool QGL agreement: T0 and T1 OptoTap probes share identical tilt plane down to 0.2 microns.
- [x] Measure live print-height toolchange stopwatch benchmarks (20.11s at 50 mm/s vs 46.2s stock).
- [x] Standardize `samples_tolerance: 0.008 mm` on both T0 and T1 OptoTap probes to eliminate retry loops.
- [x] Qualify CoreXY `max_velocity: 400 mm/s` and `max_accel: 5000 mm/s²` with `1.0A` driver current.
- [x] Increase toolchange transit `params_fast_speed` to `15000 mm/min` ($250\text{ mm/s}$), clocking live swaps at **18.90s**.
- [x] **Configuration Cleanup:**
  - Removed duplicate/dead `#[input_shaper]` and `G32` blocks in [`printer.cfg`](../printer_data/config/printer.cfg).
  - Removed obsolete `OLD_PRINT_START`, `OLD_PRINT_END`, and dead load/unload macros in [`printer.cfg`](../printer_data/config/printer.cfg).
  - Purged year-old backup files (`printer-*.cfg-old`, `moonraker-*.conf-old`) from `printer_data/config/`.
  - Verified sensorless homing currents in [`tapchanger/homing.cfg`](../printer_data/config/tapchanger/homing.cfg) cleanly drop to 0.5A on A/B steppers and restore to 1.0A.

### Active & Upcoming Tasks
- [ ] **Hardware:** Install overhead metal tension wire to keep toolhead umbilical wiring taut (eliminates T1 slack).
- [ ] **Software Package Maintenance:**
  - Run Update Manager for Moonraker, Mainsail, KAMP, and KlipperScreen.
- [ ] **Validation Print:**
  - Run multi-tool dual-color test cube or benchy to validate real-world extrusion and tool swapping during a continuous print job.
