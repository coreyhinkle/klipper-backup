# Tuning Roadmap & Optimization Checklist

This document tracks completed milestones, active tuning tasks, and future optimization opportunities for the Voron 2.4 StealthChanger.

---

## 1. Speed & Acceleration Optimization Matrix

| Parameter | Current Setting | Target Setting | Tuning Plan & Status |
|:---|:---|:---|:---|
| **`max_z_velocity`** | `50.0 mm/s` | `50.0 mm/s` | **Completed & Qualified.** Slashes vertical travel time to top-frame docks to 11.8s. |
| **`max_z_accel`** | `500 mm/s²` | `500 mm/s²` | **Completed & Qualified.** Zero step loss, QGL converges to 5 microns. |
| **`max_velocity` (XY)** | `400 mm/s` | `400 mm/s` | **Completed & Qualified.** Velocity ladder passed cleanly up to **600 mm/s (900 RPM / 1273 RPM diagonal)** with 0 lost steps. Production at 400 mm/s maintains 50% safety margin. |
| **`max_accel` (XY)** | `5000 mm/s²` | `5000 mm/s²` | **Completed & Qualified.** Accel ladder passed up to **10,000 mm/s²**. Set to 5,000 mm/s² production baseline for ideal input shaper quality. |
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
  - Removed dead out-of-bounds wipe macros (`_PARK_ON_COOLING_PAD`, `_CLEAN_NOZZLE`, `_WIPE_NOZZLE`, `CLEAN_NOZZLE`).
  - Fixed crash detection in [`tapchanger/tool_detection.cfg`](../printer_data/config/tapchanger/tool_detection.cfg) to `PAUSE` instead of cutting motor power (`M84`).
  - Added 3-minute Nevermore post-print chamber venting to `PRINT_END`.
  - Added `CALIBRATE_TOOL_SHAPER` macro for multi-tool resonance testing via native Klipper `CHIPS=` parameter.
  - Raised T0 Rapido HF `max_temp` to 300°C in [`sb-2.cfg`](../printer_data/config/sb-2.cfg).
  - Optimized `PRINT_START` with parallel non-blocking `M104 S150` nozzle preheat alongside bed heating.

### Active & Upcoming Tasks
- [ ] **Toolhead 2 (`T2` - `cyan` Dragon Burner) & Toolhead 3 (`T3` - `yellow` Dragon Burner) Bring-Up:**
  - [x] Flash Katapult + Klipper (`v0.13.0-800-g9cb99754`, 8KiB offset, 1 Mbps CAN on `PB0/PB1`) to both BTT EBB36 v1.2 boards (`[mcu cyan]` `108d45e46fd2` and `[mcu yellow]` `ce0655e3d001`).
  - [x] Verify `T2` (`cyan`) fans (`PA0` part fan, `PA1` hotend fan), thermistor (`PA3`, `ATC Semitec 104NT-4-R025H42G`), heater (`PB13`), TMC2209 (`PA15`), ADXL345 (`PB12`), and RGBW NeoPixels (`PD3`, `color_order: GRBW`).
  - [x] Run `T2` hotend PID calibration at 240°C (`pid_Kp: 40.092, pid_Ki: 13.364, pid_Kd: 30.069`).
  - [x] Verify `T2` OptoTap sensor (`^cyan:PB6`) and complete 200-tap break-in (`0.99 µm` std dev across taps 101–200).
  - [ ] **T2 Extruder Calibration (`[extruder2]` - Sherpa Mini):**
    - Verify motor rotation direction (`dir_pin: !cyan:PD1` vs `cyan:PD1`) so positive `+E` pulls filament into the hotend.
    - Confirm pinion gear tooth count (`gear_ratio: 50:10` vs `50:8`) and run 100 mm filament extrusion calibration to dial in `rotation_distance` (currently `22.67895` in [`T2.cfg`](../printer_data/config/toolchanger/tools/T2.cfg)).
  - [ ] **T2 Physical Assembly & Hardware Tuning:**
    - Install support brace across the back of `T2` (`cyan`).
    - Finish assembling and mounting `T2`'s DraftShift Modular Dock (`Top_Crossbar.stl` / `Bottom_Crossbar.stl`) on the front 2020 crossbar.
    - Calibrate and save `params_park_x`, `params_park_y`, `params_park_z` in `[tool T2]` ([`T2.cfg`](../printer_data/config/toolchanger/tools/T2.cfg)) and verify clean dropoff/pickup cycles. *(Do NOT run `T0`, `T1`, or `UNSELECT_TOOL` while `T2` is on the shuttle until `T2`'s dock is mounted and calibrated!)*
  - [ ] **Recalculate Nudge Probe Position (`_CALIBRATION_SWITCH`):**
    - Re-measure `variable_x`, `variable_y`, and `variable_z` in [`toolchanger-config.cfg`](../printer_data/config/toolchanger/toolchanger-config.cfg) using master reference tool `T0` since the physical Nudge mount location was moved slightly.
  - [ ] **T2 Probe Z-Offset & Tool X/Y/Z Offset Calibration:**
    - Calibrate `[tool_probe T2]` `z_offset` (write manually into [`T2.cfg`](../printer_data/config/toolchanger/tools/T2.cfg); never use `SAVE_CONFIG`).
    - Run Nudge calibration to set `gcode_x_offset`, `gcode_y_offset`, and `gcode_z_offset` on `T2` relative to `T0`.
  - [ ] **T2 Input Shaper Calibration:**
    - Run resonance testing using `[adxl345 T2]` and update `params_input_shaper_*` in [`T2.cfg`](../printer_data/config/toolchanger/tools/T2.cfg).
  - [ ] **T3 (`yellow`) Assembly & Bring-Up:**
    - Assemble `T3` hardware, uncomment [`T3.cfg`](../printer_data/config/toolchanger/tools/T3.cfg), and repeat peripheral verification, PID, extruder calibration, dock parking, and Nudge offset calibration.
- [ ] **Thermal / Electronics Cooling:** Install top-mount heatsink / active cooler on Raspberry Pi CM5 SoC. Revert/automate skirt fans in [`printer.cfg`](../printer_data/config/printer.cfg#L239-L251) (currently set to temporary `fan_generic` for manual cooling).
- [ ] **Umbilical Management & CAN Hub Upgrade (Top Frame & Rear):**
  - Print and install DraftShift/N3MI **Voron 350 5-Tool Umbilical Plate** on top rear 2020 extrusion with ECAS04 PTFE collets and 0.3x3mm spring steel relief anchors.
  - Print and mount **Fanny Pack** enclosure on rear frame to house the 5-port CAN distribution board, routing wiring safely down via the wire duct.
  - Fabricate universal 350mm-length umbilicals (~740mm) using 0.3x3mm spring steel spines to eliminate all OptoTap vertical drag and slack variance on T1.
  - Terminate umbilicals with modular toolhead strain relief (CW2/SB2209 for current tools; EBB36/DragonBurner compatible for expansion).
- [ ] **Multi-Tool Fleet Expansion (Hybrid StealthBurner + DragonBurner):**
  - Maintain T0 (Rapido HF) and T1 (V6) as primary StealthBurners with Clockwork 2 extruders and OptoTap.
  - Plan expansion of T2–T4 using lightweight DragonBurners (Sherpa Mini / Sharkpa + EBB36) for 5-tool CMYKW Prusa ColorMix or multi-material support.
- [x] **Software Package Maintenance:**
  - Upgraded Mainsail (`v2.19.0`), KlipperScreen (`v0.4.7-196`), KAMP (`v1.1.2-14`), Sonar (`v0.2.0-2`), and `klipper-backup` (`v1.3.6-7`).
- [x] **MCU Firmware Upgrade (Katapult / CANBus):**
  - Upgraded Mainboard (STM32H723), Toolhead 0 (RP2040), and Toolhead 1 (RP2040) to `v0.13.0-800-g9cb99754`.
  - Patched `klipper-toolchanger` with upstream PR #174 for latest Klipper probe API compatibility.
  - Fully qualified with live `G28` homing and complete round-trip toolchange (`T0` $\to$ `T1` $\to$ `T0`).
- [x] **QGL & Bed Mesh Qualification:**
  - `QUAD_GANTRY_LEVEL` converged to 7.5 microns (`0.0075 mm`) in 2 iterations.
  - `BED_MESH_CALIBRATE` verified under updated Klipper probe helpers; cold plate variance measured at `0.210 mm` across the 350x350 mm bed.
- [x] **Enclosure Clearance:** Installed **DraftChange front door extenders** to provide clearance for the front-mounted StealthChanger docks ($Y \approx 1\text{--}2\text{ mm}$, $Z \approx 298\text{ mm}$) and forward-shifted carriage shuttle.
- [x] **Optical Tool Offset Calibration (T0 vs T1):** Calibrated XY nozzle offsets using an upward-facing USB macro camera with centroid sub-pixel tracking. Updated T1 in [`ebb-sb-rp2040.cfg`](../printer_data/config/ebb-sb-rp2040.cfg) to `gcode_x_offset: 1.447`, `gcode_y_offset: -2.236` (residual error $< 50\ \mu\text{m}$, repeatability $\sigma < 20\ \mu\text{m}$).
- [x] **Nozzle Brush Alignment Audit:**
  - Evaluated rear bed brush at `X=265, Y=350, Z=15`. Confirmed toolchanger shuttle geometry places the nozzle ~15mm forward of gantry travel limits, leaving insufficient clearance behind a 350x350 mm bed for a rear purge bucket.
  - **Front Bed Brush Feasibility:** With DraftChange front door extenders installed, a front-bed mounted brush/wiper (attached to the front bed extrusion) is physically viable and avoids rear-travel limits with minimal transit distance to front docks.
  - Standardized on KAMP adaptive bed purge for print start, and OrcaSlicer / PrusaSlicer Prime Tower for toolchange transitions.
- [ ] **Dock-Integrated Wipers & Ooze Blockers (Future Hardware Upgrade):**
  - Print and assemble DraftShift Modular Dock open-hardware nozzle wipers (PTFE tube squeegee / generic high-temp silicone flap) and spring-loaded silicone cup ooze blockers (cast with automotive high-temp RTV silicone). Strictly open-source / generic materials (zero proprietary/Bambu components).
- [x] **Filament Handling & Macro Modernization:**
  - Standardized `M600` on upstream Mainsail `PAUSE X=175 Y=20 Z_MIN=50` pattern (preserves bed heater and active nozzle temp).
  - Consolidated filament loading to lean, tool-aware macros (`LOAD_FILAMENT` and `UNLOAD_FILAMENT`) calibrated for Clockwork 2 direct-drive paths (~50-60mm).
  - Added carriage mount safety guards (`printer.toolchanger.tool_number < 0`) to prevent cold or dock extrusion.
  - Eliminated hazardous legacy macros (`UNLOAD_ALL_FILAMENT` dock extrusion risk, `CHANGE_NOZZLE`, and `Z20` collision moves).
- [x] **Klipper `v0.13.0-800` & `klipper-toolchanger` PR #174 Config Audit & Homing Fix:**
  - Fixed `_ADJUST_Z_HOME_FOR_TOOL_OFFSET` and `TOOL_BED_MESH_CALIBRATE` in [`tapchanger/homing.cfg`](../printer_data/config/tapchanger/homing.cfg) to use native `ADJUST_Z_AFTER_TOOL_NOZZLE_HOME` without double-applying `probe_z_offset` or overwriting active bed mesh Z compensation.
  - Updated `[bed_mesh]` in [`printer.cfg`](../printer_data/config/printer.cfg) to standard Voron 350mm bounds (`mesh_min: 40, 40`, `mesh_max: 310, 310`) to prevent extreme rear-left umbilical pull (`X=5, Y=340`), and added `fade_start: 1.0` / `fade_end: 10.0` so bed mesh Z offsets fade out completely well below top-docking height (`Z ≈ 298 mm`).
  - Added `_CLIENT_VARIABLE` and `_ON_PRINT_CANCEL` in [`printer.cfg`](../printer_data/config/printer.cfg) to disarm crash detection on print cancel and park safely at rear center (`X=175, Y=300`).
  - Hardened `PRINT_START` in [`tapchanger/macros.cfg`](../printer_data/config/tapchanger/macros.cfg) to always select master reference tool `T0` for QGL, Z homing, and bed meshing, and added a `1.0°C` PID tolerance to `pickup_gcode` `TEMPERATURE_WAIT` in [`tapchanger/toolchanger-2.4.cfg`](../printer_data/config/tapchanger/toolchanger-2.4.cfg).
- [x] **Validation Print:**
  - **Passed & Qualified:** Executed multi-tool dual-color validation print (`micro+bitonal+whistle+color`) with active tool swapping across all print layers. Verified zero lost steps, seamless docking transitions at 250 mm/s transit (18.9s toolchange duration), accurate thermal preheating, and precise layer alignment with KAMP adaptive bed mesh.
  - **Reversion Anchor:** Tagged in git as `multitool-validated` on commit `6183d9b`.

