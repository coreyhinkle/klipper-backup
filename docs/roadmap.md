# Tuning Roadmap & Optimization Checklist

This document tracks completed milestones, active tuning tasks, and future optimization opportunities for the Voron 2.4 StealthChanger.

---

## 1. Speed & Acceleration Optimization Matrix

| Parameter | Current Setting | Target Setting | Tuning Plan & Status |
|:---|:---|:---|:---|
| **`max_z_velocity`** | `50.0 mm/s` | `50.0 mm/s` | **Completed & Qualified.** Slashes vertical travel time to top-frame docks to 11.8s. |
| **`max_z_accel`** | `500 mm/s²` | `500 mm/s²` | **Completed & Qualified.** Zero step loss, QGL converges to 5 microns. |
| **`max_velocity` (XY)** | `300 mm/s` | `400 mm/s` | **Next Up.** 24V TMC2209s easily support 400 mm/s rapid travels. |
| **`max_accel` (XY)** | `3000 mm/s²` | `5000 – 6000 mm/s²` | Input shaper frequencies (T0: 62/88Hz; T1: 60/72Hz MZV) easily support 5000+ accel. |
| **`params_fast_speed`** | `10000 mm/min` ($166\text{ mm/s}$) | `15000 – 18000 mm/min` ($250–300\text{ mm/s}$) | Accelerates cross-bed transit during tool swaps. |
| **`run_current` (XY)** | `0.80 A` | `1.00 A` | Increases motor holding torque for 5000+ accel without overheating drivers. |

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

### Active & Upcoming Tasks
- [ ] **Config:** Relax `samples_tolerance` on `[tool_probe T1]` from `0.006` to `0.008 mm` to eliminate retry loops.
- [ ] **Hardware:** Install overhead metal tension wire to keep toolhead umbilical wiring taut (eliminates T1 slack).
- [ ] **XY Speed Tuning:**
  - Increase `stepper_x` and `stepper_y` `run_current` from `0.8A` to `1.0A` in [`printer.cfg`](../printer_data/config/printer.cfg).
  - Increase `max_velocity` to `400 mm/s` and `max_accel` to `5000 mm/s²`.
  - Increase `params_fast_speed` in [`toolchanger-2.4.cfg`](../printer_data/config/tapchanger/toolchanger-2.4.cfg) to `15000 mm/min`.
  - Execute automated XY step-loss test and benchmark cross-bed transit time.
- [ ] **Configuration Cleanup:**
  - Remove duplicate `[resonance_tester]` in [`ebb-sb-rp2040.cfg`](../printer_data/config/ebb-sb-rp2040.cfg).
  - Remove duplicate empty `[input_shaper]` in [`printer.cfg`](../printer_data/config/printer.cfg).
  - Equalize sensorless homing currents in [`tapchanger/homing.cfg`](../printer_data/config/tapchanger/homing.cfg).
- [ ] **Software Package Maintenance:**
  - Run Update Manager for Moonraker, Mainsail, KAMP, and KlipperScreen.
