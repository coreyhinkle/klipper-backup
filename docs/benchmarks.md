# Kinematics Benchmarks & Speed Qualification

This document records the empirical testing, kinematic calculations, and real-world timing benchmarks conducted to tune vertical travel and toolchange transit speeds on this **Voron 2.4 (350mm)** printer.

---

## 1. Top-Mounted Frame Docks & The 600 mm Z-Travel Reality

On standard toolchangers (e.g. Jubilee, IDEX, or gantry-docked Vorons), tool docks ride directly on the moving Z-gantry, meaning a toolchange only requires an 8 mm Z-hop to clear dock pins.

On this printer, the StealthChanger docks are fixed to the **upper frame extrusions**:
* **T0 Park Coordinates:** `X = 339.25, Y = 1.10, Z = 297.55` ([`sb-2.cfg`](../printer_data/config/sb-2.cfg#L86))
* **T1 Park Coordinates:** `X = 4.00, Y = 2.00, Z = 299.00` ([`ebb-sb-rp2040.cfg`](../printer_data/config/ebb-sb-rp2040.cfg#L94))

### The Vertical Journey per Toolchange:
When executing a toolchange during a print near the bed (e.g. $Z = 5\text{ mm}$):
1. **Vertical Climb:** The gantry must lift $294\text{ mm}$ from print height to the ceiling dock.
2. **Docking Path:** The shuttle unmates the active tool and engages the new tool.
3. **Vertical Descent:** The gantry must dive $294\text{ mm}$ back down to the print layer.

**Every single toolchange demands nearly $590\text{ mm}$ of vertical travel.** 

Because of this massive vertical distance, Z velocity directly dictates total toolchange duration.

---

## 2. Empirical Z-Axis Speed Ladder Test

To establish the maximum safe operating speed for the Z gantry without risking step loss, an automated torture routine was executed. At each speed tier (+5 mm/s increments), the gantry ran **5 consecutive full-stroke cycles** ($Z 15 \leftrightarrow 200\text{ mm}$, totaling $1.85\text{ m}$ of high-speed vertical travel per step) followed by an optical OptoTap probe measurement against the bed:

| Z Velocity | Z Accel | Stepper RPM | Measured $\Delta Z$ Drift | Status / Observation |
|:---|:---|:---|:---|:---|
| **30 mm/s** | 500 mm/s² | 225 RPM | `+0.0025 mm` | Pass (1 microstep, sensor baseline) |
| **35 mm/s** | 500 mm/s² | 262.5 RPM | `+0.0075 mm` | Pass (3 microsteps, well within tolerance) |
| **40 mm/s** | 500 mm/s² | 300 RPM | `+0.0025 mm` | Pass (1 microstep) |
| **45 mm/s** | 500 mm/s² | 337.5 RPM | `+0.0000 mm` | Pass (Exact zero drift) |
| **50 mm/s** | 500 mm/s² | 375 RPM | `+0.0050 mm` | Pass (2 microsteps, robust margin) |
| **55 mm/s** | 500 mm/s² | 412.5 RPM | `+0.0025 mm` | Pass (1 microstep) |
| **60 mm/s** | 500 mm/s² | 450 RPM | `+0.0050 mm` | Pass (2 microsteps) |
| **65 mm/s** | 500 mm/s² | 487.5 RPM | `+0.0025 mm` | Pass (Highest zero-loss speed) |
| **70 mm/s** | 500 mm/s² | 525 RPM | `+0.0100 mm` | Marginal Slip (4 microsteps, back-EMF onset) |
| **80 mm/s** | 500 mm/s² | 600 RPM | `+0.0125 mm` | Slip (5 microsteps, back-EMF torque loss) |

### Drive Mechanics & 24V Electrical Limits:
- **Gear Reduction:** The Voron 2.4 Z-axis utilizes an 80:16 (5:1) gear ratio with 20T pulleys (2mm GT2 pitch $\implies 40\text{ mm}$ belt pitch per revolution).
  $$\text{Lead} = \frac{40\text{ mm}}{5} = 8.0\text{ mm / rev}$$
- **Back-EMF Boundary:** On a 24V supply (TMC2209), standard NEMA 17 steppers experience rapid torque degradation beyond 500 RPM due to back-EMF voltage approaching the rail voltage. At 70 mm/s (525 RPM), holding torque during deceleration begins to slip microsteps.
- **Operating Baseline:** **`max_z_velocity: 50.0 mm/s`** operating at 375 RPM preserves a substantial safety cushion below the 65 mm/s zero-loss ceiling.

---

## 3. Live Stopwatch Toolchange Benchmarks

A real-world test script was executed directly on the printer, positioning the toolhead at print height ($Z = 5.0\text{ mm}$) and recording elapsed time for the full climb, tool swap, and descent:

| Speed Profile | Pure Z Travel Time (590mm round trip) | Measured Total Toolchange Time | Time Saved per Toolchange | 500-Swap Print Total |
|:---|:---|:---|:---|:---|
| **15 mm/s Z / Stock XY** | 39.2s | $\approx 46.2\text{ s}$ | — | **6.4 hours** |
| **35 mm/s Z / Stock XY** | 16.8s | **23.53 s** | **22.7s saved** | **3.26 hours** |
| **50 mm/s Z / Stock XY** | 11.8s | **20.11 s** | **26.1s saved** | **2.79 hours** |
| **50 mm/s Z / 400 mm/s XY (15k fast)** | 11.8s | **18.90 s** | **27.3s saved** | **2.62 hours** |

* Moving from stock defaults to tuned production speeds **slashed toolchange time from 46.2s to 18.90s (59% reduction)**, saving **3.78 hours** on a 500-toolchange multi-color print.

---

## 4. CoreXY Velocity & Acceleration Empirical Ladder Tests

To establish the maximum safe operating envelope for the CoreXY horizontal kinematics (X and Y axes) without risking step loss, motor stall, or driver overheating, an automated multi-tier stress routine was executed in the safe open-air volume ($X: 40 \to 310\text{ mm}, Y: 50 \to 300\text{ mm}, Z = 50.0\text{ mm}$, totaling $>4\text{ meters}$ of continuous full-stroke moves per tier).

Each tier executed 2 perimeter boxes (pure X & Y moves), 4 full-stroke diagonal crosses (isolating Motor A and Motor B at $v \cdot \sqrt{2}$), and alternating high-speed diamond vectors. After each tier, the toolhead re-homed to measure raw MCU step drift against the hardware endstops:
* **Sensorless X:** Baseline jitter $\pm 1$ full step (16 microsteps = $0.20\text{ mm}$).
* **Physical Microswitch Y:** Baseline jitter $\pm 2$ microsteps ($0.025\text{ mm}$).

### Phase 1: Velocity Ladder (Fixed Acceleration = 5,000 mm/s², 1.0A Driver Current)

| Velocity | Stepper RPM (Pure XY) | Stepper RPM (Diagonal $\sqrt{2}$) | Run Time | Measured $\Delta X$ Drift | Measured $\Delta Y$ Drift | Driver Temp | Status / Observation |
|:---|:---|:---|:---|:---|:---|:---|:---|
| **300 mm/s** | 450 RPM | 636 RPM | 35.64s | `0 steps (0.0000 mm)` | `0 steps (0.0000 mm)` | 62.0°C | **PASS** (Zero Step Loss) |
| **350 mm/s** | 525 RPM | 742 RPM | 31.39s | `0 steps (0.0000 mm)` | `0 steps (0.0000 mm)` | 62.4°C | **PASS** (Zero Step Loss) |
| **400 mm/s** | 600 RPM | 848 RPM | 28.30s | `1 step (0.0125 mm)` | `1 step (0.0125 mm)` | 62.7°C | **PASS** (1 microstep, baseline noise) |
| **450 mm/s** | 675 RPM | 954 RPM | 25.98s | `1 step (0.0125 mm)` | `1 step (0.0125 mm)` | 63.2°C | **PASS** (1 microstep) |
| **500 mm/s** | 750 RPM | 1,060 RPM | 24.19s | `0 steps (0.0000 mm)` | `0 steps (0.0000 mm)` | 63.2°C | **PASS** (Exact zero drift) |
| **550 mm/s** | 825 RPM | 1,167 RPM | 22.81s | `0 steps (0.0000 mm)` | `2 steps (0.0250 mm)` | 63.7°C | **PASS** (2 microsteps, endstop jitter) |
| **600 mm/s** | 900 RPM | 1,273 RPM | 21.70s | `1 step (0.0125 mm)` | `1 step (0.0125 mm)` | 63.7°C | **PASS** (Highest tested zero-loss speed) |

### Phase 2: Acceleration Ladder (Tested at 400 mm/s Velocity, 1.0A Driver Current)

| Acceleration | Motion Profile | Run Time | Measured $\Delta X$ Drift | Measured $\Delta Y$ Drift | Driver Temp | Status / Observation |
|:---|:---|:---|:---|:---|:---|:---|
| **4,000 mm/s²** | Conservative Baseline | 29.03s | `1 step (0.0125 mm)` | `1 step (0.0125 mm)` | 64.1°C | **PASS** (Zero Step Loss) |
| **5,000 mm/s²** | Current Production Baseline | 28.30s | `1 step (0.0125 mm)` | `1 step (0.0125 mm)` | 64.2°C | **PASS** (Zero Step Loss) |
| **6,000 mm/s²** | High Responsiveness | 27.82s | `1 step (0.0125 mm)` | `1 step (0.0125 mm)` | 64.5°C | **PASS** (Zero Step Loss) |
| **7,000 mm/s²** | High Dynamic Torque | 27.47s | `0 steps (0.0000 mm)` | `0 steps (0.0000 mm)` | 64.7°C | **PASS** (Exact zero drift) |
| **8,000 mm/s²** | Extreme Dynamic Load | 27.21s | `0 steps (0.0000 mm)` | `2 steps (0.0250 mm)` | 64.8°C | **PASS** (2 microsteps) |
| **9,000 mm/s²** | Extreme Dynamic Load | 27.01s | `0 steps (0.0000 mm)` | `0 steps (0.0000 mm)` | 64.9°C | **PASS** (Exact zero drift) |
| **10,000 mm/s²** | Maximum Torture Limit | 26.85s | `0 steps (0.0000 mm)` | `2 steps (0.0250 mm)` | 65.3°C | **PASS** (Zero Step Loss across all passes) |

### Engineering Takeaways & Production Recommendations:
1. **Torque Margin & Thermal Stability:** TMC2209 stepper drivers operating at `1.0A` run current stayed exceptionally cool (rising only from 62.0°C to 65.3°C throughout 14 continuous back-to-back stress runs), proving that active skirt cooling provides ample thermal headroom.
2. **Speed Overhead:** The machine operated without step loss up to **600 mm/s** ($900\text{ RPM}$ pure / $1,273\text{ RPM}$ diagonal). Running at `max_velocity: 400 mm/s` in production maintains a massive **50% safety cushion** below the kinematic limit.
3. **Acceleration & Input Shaper Tuning:** While the motors and belts reliably endured **10,000 mm/s²**, production accelerations are limited by print quality and input shaper smoothing:
   - Tool 0 resonance frequencies: MZV @ 62.4 Hz (X), 88.6 Hz (Y) $\implies$ recommended $a_{\max} \approx 5{,}500 - 6{,}500\text{ mm/s²}$.
   - Tool 1 resonance frequencies: MZV @ 60.1 Hz (X), 72.3 Hz (Y) $\implies$ recommended $a_{\max} \approx 5{,}000 - 6{,}000\text{ mm/s²}$.
   - **`max_accel: 5000 mm/s²`** is the ideal production baseline for crisp, ringing-free prints across both toolheads.

---

## 5. Normal Print Quality: Why High Z Speeds Do Not Affect Printing

A common concern is that increasing `max_z_velocity` will degrade print quality during regular layers. However, Z is **100% acceleration-limited** during printing:
* **Layer Change:** $\Delta Z = 0.20\text{ mm}$.
* **Slicer Z-Hop:** $\Delta Z = 0.20\text{ to } 0.40\text{ mm}$.
* Over a $0.20\text{ mm}$ move at $a = 500\text{ mm/s²}$, the peak speed reached before braking is:
  $$v_{\text{peak}} = \sqrt{a \cdot d} = \sqrt{500 \cdot 0.20} = \mathbf{10.0\text{ mm/s}}$$

The Z axis physically never exceeds 10 mm/s during normal printing moves. The 50 mm/s speed is only ever utilized during the long vertical climbs to the frame docks and homing/QGL hops.

---

## 6. Quad Gantry Leveling (QGL) Qualification & Cross-Tool Agreement

### Verification at 50 mm/s
Following the switch to 50 mm/s, full homing and QGL was verified:
* **Target Tolerance:** `0.0075 mm` (7.5 microns).
* **Achieved Tolerance:** **`0.0050 mm` (5.0 microns)** across all corners on pass 4.
* **Leveling Adjustments:**
  * `stepper_z  = -0.000200 mm` (-0.2 µm)
  * `stepper_z1 = -0.004500 mm` (-4.5 µm)
  * `stepper_z2 = +0.000200 mm` (+0.2 µm)
  * `stepper_z3 = +0.004500 mm` (+4.5 µm)

### Cross-Tool QGL Comparison (T0 OptoTap vs T1 OptoTap)
Comparing QGL runs between T0 and T1 revealed that **both probes converged to the exact same gantry tilt plane down to 0.2 microns**. This confirms that the carriage shuttle and docking mechanisms maintain perfect rigidity without any toolhead pitch or yaw skew between tools.

---

## 7. Multi-Tool Dual-Color Validation Print Benchmark

### Test Object & Conditions
* **Model:** `micro+bitonal+whistle+color` (dual-material whistle).
* **Toolheads:** Toolhead 0 (T0 - Rapido HF 0.4mm, Purple) & Toolhead 1 (T1 - V6 0.4mm, Green).
* **Kinematics:** CoreXY $v_{\max} = 400\text{ mm/s}$, $a = 5{,}000\text{ mm/s²}$; Z-axis $v_{\max} = 50.0\text{ mm/s}$, $a = 500\text{ mm/s²}$.
* **Toolchange Transit Speed:** `params_fast_speed: 15000 mm/min` ($250\text{ mm/s}$), `params_path_speed: 900 mm/min` ($15\text{ mm/s}$).
* **Probing:** OptoTap on T0/T1, `horizontal_move_z: 4.0 mm` for QGL and `BED_MESH_CALIBRATE ADAPTIVE=1`.

### Results & Verification
1. **Toolchange Reliability:** 100% successful pick-and-place cycles across dozens of tool swaps throughout the print. Zero lost steps, no false crash detection triggers, and zero magnet misalignments.
2. **Thermal Stability:** Active tool heated to print temperature while docked tool dropped to idle without temperature overshoot or buffer stall.
3. **Dimensional & Layer Alignment:** Excellent XY interfacial layer registration with zero visible seam shift between colors, confirming calibrated Nudge offsets (`gcode_x_offset: 0.450`, `gcode_y_offset: -0.250`, `gcode_z_offset: 0.3025`).
4. **Git Revert Point:** Tagged baseline `multitool-validated`.

---

## 8. OptoTap QGL & Bed Mesh Isolated Speed Optimization (`T0` Rapido HF StealthBurner)

### Hardware & Thermal Test Profile
Because StealthChanger magnetic shuttle settling depends directly on toolhead mass, center-of-gravity overhang, and umbilical stiffness, all measurements in this section are explicitly baselined to the heavier **Rapido HF StealthBurner** master reference assembly (distinct from the lighter `T1` V6 StealthBurner and planned DragonBurner/EBB36 expansion tools):
* **Toolhead Under Test:** **`T0` — Purple StealthBurner** (`Phaetus Rapido HF 0.4 mm` hotend, `Clockwork 2` extruder, `BTT EBB SB2209 RP2040` CAN board, `OptoTap` optical sensor on `^purple:gpio22`, `z_offset: -0.605`).
* **Build Plate & Thermal Equilibrium:** **Smooth PEI** spring-steel sheet held at **`60.0°C`**, `T0` nozzle held at **`150.0°C`** throughout all runs to eliminate bed thermal-expansion drift and match active Voron Tap `PRINT_START` conditions.
* **Kinematics Baseline:** CoreXY `400 mm/s` (`5,000 mm/s²`), Z-axis `50.0 mm/s` (`500 mm/s²`).
* **Stock Probe Baseline (`T0.cfg` / `printer.cfg`):** `speed: 10.0 mm/s`, `lift_speed: 10.0 mm/s` *(implicit default)*, `samples: 3` (`median`), `sample_retract_dist: 3.0 mm`, `samples_tolerance: 0.008 mm`, `horizontal_move_z: 4.0 mm` (QGL & Bed Mesh).
* **Methodology:** Every parameter was tested **strictly in isolation** via Klipper runtime G-code overrides (`LIFT_SPEED=`, `PROBE_SPEED=`, `SAMPLE_RETRACT_DIST=`, `SAMPLES=`, `SAMPLES_TOLERANCE=`, `HORIZONTAL_MOVE_Z=`) before evaluating combined profiles. Every QGL run started from an identical `+0.50 mm` `stepper_z2` corner droop injected via `FORCE_MOVE` at `(175, 175, Z=10)`. Every 5×5 bed mesh (`40,40` to `310,310`) was center-normalized against Phase 0 stock to quantify surface curvature fidelity (`mean` and `max` shape deviation in $\mu\text{m}$).

---

### 8.1 Root-Cause Diagnosis: Pre-Optimization Real-World Log Audit
Prior to controlled testing, an audit of `klippy.log` from a live Gridfinity print revealed why `QUAD_GANTRY_LEVEL` felt sluggish despite `400 mm/s` XY travels:
* **Live Print QGL Duration:** **149.1 seconds (2m 29s)** across **5 passes** and **111 probe taps**, hitting `"Probe samples exceed tolerance. Retrying..."` **19 times**.
* **Mechanical Mechanism (Tap #1 Shuttle Seating):** After a `250 mm` corner-to-corner travel at `400 mm/s`, **Tap #1 consistently triggered `10.0–25.0 µm` (`0.010–0.025 mm`) offset** from Taps #2–#4 as the heavy **Rapido HF StealthBurner** settled into its StealthChanger magnetic seats/linear rail. Once seated, consecutive taps at the same XY coordinate repeated within **`0.0000–0.0025 mm` (0–1 microstep)**.
* **Why `samples_tolerance: 0.008` Backfired:** Although `samples_result: median` (`samples: 3`) mathematically discards a single outlier (Tap #1), Klipper's `SampleAveragingHelper` checks `max(z) - min(z) > samples_tolerance` *before* computing the median. Setting `samples_tolerance: 0.008 mm` (3.2 microsteps) caused normal `10–15 µm` Tap #1 seating on the Rapido StealthBurner to wipe all 3 samples and force 6–9 taps per point.
* **Missing `lift_speed`:** Omitting `lift_speed` in `[tool_probe T0]` defaulted inter-sample retracts, `horizontal_move_z` lifts, *and* QGL's 4-motor Z adjustments (`quad_gantry_level.py`) to `10.0 mm/s`.

---

### 8.2 Stationary 20-Sample Repeatability (`PROBE_ACCURACY` at `175, 175`)

| Isolated Variable Tested | `probe_speed` | `lift_speed` | `retract` | Elapsed Time | Total Range (20 taps) | Std Dev ($\sigma$) | Engineering Observation |
|:---|:---|:---|:---|:---|:---|:---|:---|
| **Phase 0 / Stock Baseline** | `10.0 mm/s` | `10.0 mm/s` | `3.0 mm` | `22.50 s` | `0.0225 mm` | `5.61 µm` | Tap #1 outlier after `Z=10` approach inflates $\sigma$. |
| **Test 1A: `LIFT_SPEED=35`** | `10.0 mm/s` | **`35.0 mm/s`** | `3.0 mm` | `19.37 s` | `0.0150 mm` | `3.47 µm` | Crisp upward retract; zero accuracy degradation. |
| **Test 1B: `LIFT_SPEED=45`** | `10.0 mm/s` | **`45.0 mm/s`** | `3.0 mm` | `19.39 s` | `0.0100 mm` | **`1.82 µm`** | 17 of last 18 taps hit identical `-0.0225 mm` microstep. |
| **Test 2A: `PROBE_SPEED=8`** | **`8.0 mm/s`** | `10.0 mm/s` | `3.0 mm` | `23.95 s` | `0.0125 mm` | `2.41 µm` | $>2\times$ tighter $\sigma$ vs `10 mm/s` plunge. |
| **Test 2B: `PROBE_SPEED=6`** | **`6.0 mm/s`** | `10.0 mm/s` | `3.0 mm` | `28.01 s` | `0.0100 mm` | `2.66 µm` | Gentle smooth-PEI contact; minimal overshoot. |
| **Test 2C: `PROBE_SPEED=5`** | **`5.0 mm/s`** | `10.0 mm/s` | `3.0 mm` | `29.20 s` | **`0.0075 mm`** | **`1.87 µm`** | Sub-microstep $\sigma$; all 20 taps within `7.5 µm`. |
| **Test 3A: `RETRACT=2.0`** | `10.0 mm/s` | `10.0 mm/s` | **`2.0 mm`** | `18.74 s` | `0.0150 mm` | `3.58 µm` | Clean OptoTap magnet reseating (`>1.19 mm` above deadband). |
| **Test 3B: `RETRACT=1.5`** | `10.0 mm/s` | `10.0 mm/s` | **`1.5 mm`** | `16.85 s` | `0.0150 mm` | `3.49 µm` | Clean reseating, though tighter margin above `0.81 mm` deadband. |

---

### 8.3 Full-Motion `QUAD_GANTRY_LEVEL` Isolation Matrix

All runs started from an identical controlled `+0.50 mm` `stepper_z2` twist (except Phase 0 initial cold-sag recovery from `0.960 mm` sag):

| Test ID & Isolated Change | QGL Passes | Tolerance Retries | Final Gantry Range | Elapsed Time | Takeaway (`T0` Rapido HF StealthBurner) |
|:---|:---|:---|:---|:---|:---|
| **Live Print Log (Pre-Audit)** | 5 passes | **19 retries** | `0.0075 mm` | **149.10 s** | Severe `0.008 mm` retry storm after `400 mm/s` corner whips. |
| **Phase 0 (Unleveled `0.96 mm` sag)** | 3 passes | 4 retries | `0.0050 mm` | `62.77 s` | Proves `0.96 mm` idle sag—`horizontal_move_z: 2` on Pass 1 is unsafe. |
| **Stock Controlled (`+0.50 mm`)** | 2 passes | 2 retries | `0.0075 mm` | `39.84 s` | Controlled baseline (`speed=10`, `lift=10`, `retract=3`, `tol=0.008`). |
| **Test 1: `LIFT_SPEED=40` Only** | 3 passes | 4 retries | `0.0075 mm` | `54.11 s` | Fast Z lift + `0.008 mm` tol triggers extra Tap #1 retries (`10 µm` > `8 µm`). |
| **Test 2A: `PROBE_SPEED=8` Only** | 3 passes | 3 retries | `0.0050 mm` | `63.54 s` | Still bottlenecked by `lift_speed=10` and `0.008 mm` tol. |
| **Test 2B: `PROBE_SPEED=6` Only** | 2 passes | 2 retries | `0.0075 mm` | `47.01 s` | Converges in 2 passes despite slow `10 mm/s` Z lifts. |
| **Test 2C: `PROBE_SPEED=5` Only** | 2 passes | 2 retries | `0.0100 mm` | `50.61 s` | Converges in 2 passes; slow Z lifts dominate time. |
| **Test 4A: Two-Pass (`Z=8` 1-tap $\to$ `Z=2` 3-tap)** | 1 coarse + 1 fine | 1 retry | `0.0075 mm` | **`31.29 s`** | **Fastest stock-speed QGL.** Coarse pass nails tilt; fine pass finishes in 1 lap. |
| **Test 4B: Two-Pass (`Z=8` 1-tap $\to$ `Z=2` 2-tap)** | 1 coarse + 2 fine | 2 retries | `0.0025 mm` | `41.65 s` | **10.36s slower than 3-sample median!** `SAMPLES=2` lacks outlier rejection. |
| **Test 6A: `SAMPLES_TOLERANCE=0.015` Only** | 3 passes | **0 retries** | **`0.0000 mm`** | `52.88 s` | Eliminates all retries; converges to `0.0 µm` across all 4 corners. |
| **Test 6C: `SAMPLES_TOLERANCE=0.020` Only** | 2 passes | **0 retries** | `0.0075 mm` | **`35.35 s`** | Upstream KTC-Easy tolerance eliminates retries completely. |

---

### 8.4 Full-Motion 5×5 `BED_MESH_CALIBRATE` Isolation Matrix (`40,40` to `310,310`)

| Test ID & Isolated Change | Tolerance Retries | Elapsed Time | Mean Surface $\Delta$ vs Stock | Max Point $\Delta$ vs Stock | Engineering Verdict |
|:---|:---|:---|:---|:---|:---|
| **Phase 0: Stock Baseline** | 2 retries | `103.49 s` | `0.00 µm` *(ref)* | `0.00 µm` *(ref)* | Reference 75+ tap 5×5 smooth-PEI mesh at `60°C`. |
| **Test 1: `LIFT_SPEED=40` Only** | **13 retries** | `116.36 s` | `11.00 µm` | `22.50 µm` | **Slower in isolation!** Crisp Z-lift pushes Tap #1 to `10 µm`, tripping `0.008` tol 13×. |
| **Test 3: `RETRACT=2.0` Only** | **18 retries** | `138.48 s` | **`4.80 µm`** | `12.50 µm` | Surface matches stock within `<2` microsteps, but `0.008` tol causes 18 retries. |
| **Test 5A: `HORIZONTAL_MOVE_Z=2` Only** | 8 retries | `114.36 s` | **`5.10 µm`** | `15.00 µm` | Safe post-QGL clearance, but masked until `0.008` tol retries are fixed. |
| **Test 5B: `SAMPLES=2` Only** | 5 retries | `81.13 s` | `6.70 µm` | `17.50 µm` | Cuts 22s, still suffers 5 retries under `0.008` tol. |
| **Test 5C: `SAMPLES=1` Only** | **0 retries** | **`42.38 s`** | **`5.70 µm`** | `17.50 µm` | Constant Tap #1 seating offset cancels out in bed curvature (`5.7 µm` mean $\Delta$). |
| **Test 6B: `SAMPLES_TOLERANCE=0.015` Only** | **0 retries** | `97.35 s` | **`5.00 µm`** | **`10.00 µm`** | **Zero retries** and `5.0 µm` (2-microstep) agreement across entire 350 mm bed. |
| **Test 6D: `PROBE_SPEED=6` Only** | 3 retries | `126.36 s` | `7.80 µm` | `25.00 µm` | Needs `lift_speed: 35` so slower downward plunge doesn't add travel delay. |

---

### 8.5 Combined Production Candidates (`T0` Rapido HF StealthBurner)

Once isolated testing proved **why** `lift_speed` and `horizontal_move_z` were previously bottlenecked by `samples_tolerance: 0.008 mm`, the winning parameters were benchmarked together across QGL (`+0.50 mm` twist), `G28 Z`, full 5×5 bed mesh, and a real-world **KAMP 3×3 Adaptive Mesh** (`60×60 mm` Gridfinity footprint):

| Combined Profile | QGL Strategy | Probe Params (`speed` / `lift` / `retract` / `samples` / `tol`) | QGL Time (`+0.50 mm`) | Full 5×5 Mesh Time (Mean $\Delta$) | KAMP 3×3 Adaptive Mesh | Total Pre-Print (`QGL + G28 Z + KAMP 3×3`) |
|:---|:---|:---|:---|:---|:---|:---|
| **Pre-Audit Print Log** | Single-Pass (`Z=4`) | `10 / 10 / 3.0 / 3 / 0.008` | `149.10 s` *(19 retries)* | `103.49 s` *(ref)* | `~42 s` *(est)* | **`~198 s` (3m 18s)** |
| **Two-Pass Hybrid (`PS=8`, 3-Trial Mean)** | **`Z=8` (1-tap) $\to$ `Z=2` (3-tap)** | **`8 / 35 / 2.0 / 3 / 0.020`** | **`38.90 s ± 0.39 s`** *(0 retries ×3, `5.0 µm` ×3)* | **`73.40 s`** *(0 retries, **`5.1 µm`**)* | **`25.88 s`** *(0 retries)* | **`71.48 s` (1m 11.5s)** |
| **Upstream Single-Pass (`PS=8`)** | **Stock `G32` (`Z=4`)** *(zero macro edits)* | **`8 / 35 / 2.0 / 3 / 0.020`** | `45.37 s` *(0 retries, `7.5 µm`)* | **`73.40 s`** *(0 retries, **`5.1 µm`**)* | **`25.88 s`** *(0 retries)* | `77.95 s` (1m 18s) |
| **Upstream Single-Pass (`PS=6`)** | **Stock `G32` (`Z=4`)** *(zero macro edits)* | **`6 / 35 / 2.0 / 3 / 0.020`** | `50.54 s` *(1 retry, `0.0 µm`)* | `~80 s` | `~27 s` | `~84 s` (1m 24s) |
| **Cand 1: Two-Pass (`PS=6`)** | `Z=8` (1-tap) $\to$ `Z=2` (3-tap) | **`6 / 35 / 2.0 / 3 / 0.015`** | `43.90 s` *(`2.5 µm` final)* | `84.52 s` *(`8.5 µm`)* | `27.87 s` *(0 retries)* | `78.55 s` (1m 19s) |
| **Cand 2: Two-Pass (`PS=5` Upstream)** | `Z=8` (1-tap) $\to$ `Z=2` (3-tap) | **`5 / 35 / 2.0 / 3 / 0.020`** | `45.74 s` *(`5.0 µm` final)* | `85.69 s` *(0 retries, `9.7 µm`)* | `~28.5 s` *(0 retries)* | `~81 s` (1m 21s) |
| **Cand 3: Hybrid Fast Mesh (`SAMPLES=2`)** | 3-sample QGL + 2-sample Mesh (`Z=2`) | `6 / 35 / 2.0` (`S=2` on Mesh) | `43.90 s` | **`56.16 s`** *(0 retries, `9.8 µm`)* | **`19.50 s`** *(0 retries)* | `70.18 s` (1m 10s) |

---

### 8.6 Real-World `M84` Motor-Off Sag vs. Overnight Droop (`G28` + QGL Head-to-Head)

To verify whether artificial `+0.50 mm` `FORCE_MOVE` twists biased the comparison toward Two-Pass QGL, a realistic motor-cutoff benchmark was executed with the heavy **Rapido HF StealthBurner** overhanging front-left `(60, 30, Z=30)` followed by `M84` (motors disabled), full `G28` re-homing, and QGL:

| Real-World Starting Condition | Strategy (`8 / 35 / 2.0 / 3 / 0.020`) | Measured Initial Sag | Total QGL Passes | Retries | Final Spread | QGL Time | Winner & Mechanical Explanation |
|:---|:---|:---|:---|:---|:---|:---|:---|
| **Back-to-Back Print** *(motors still enabled)* | **Upstream Single-Pass** | `<= 0.0075 mm` | **1 pass** | 0 | `2.5–7.5 µm` | **`~15.5 s`** | **Single-Pass by ~22s.** Converges in 1 lap and stops immediately. |
| **Normal `M84` Cutoff** *(15s motors off @ `60,30`)* | **Upstream Single-Pass (`Z=4`)** | **`0.0175 mm`** (`17.5 µm`) | **2 passes** | 0 | `7.5 µm` | **`30.04 s`** | **Single-Pass by `7.89s`.** 5:1 geared Z drives hold within `17.5 µm` on `M84`! |
| **Normal `M84` Cutoff** *(15s motors off @ `60,30`)* | Two-Pass (`Z=8` 1-tap $\to$ `Z=2` 3-tap) | `0.0275 mm` *(after coarse)* | 1 coarse + 2 fine | 0 | `5.0 µm` | `37.93 s` | **Slower!** Unfiltered `SAMPLES=1` coarse pass injects `15–25 µm` Tap #1 noise into a nearly flat gantry, forcing 2 extra fine passes. |
| **Normal `M84` Cutoff** *(15s motors off @ `60,30`)* | Two-Pass (`PS=6`) | `0.0200 mm` *(after coarse)* | 1 coarse + 2 fine | 2 | `5.0 µm` | `43.40 s` | Same `SAMPLES=1` coarse-pass noise penalty. |
| **Severe Overnight Droop** *(`1.2 mm` front droop + `M84`)* | **Upstream Single-Pass (`Z=4`)** | `0.6875 mm` | 3 passes | 0 | `5.0 µm` | `44.97 s` | Clears `1.2 mm` droop with `>3.3 mm` margin; `0` retries. |
| **Severe Overnight Droop** *(`1.2 mm` front droop + `M84`)* | Two-Pass (`Z=8` 1-tap $\to$ `Z=2` 3-tap) | `0.7000 mm` | 1 coarse + 2 fine | 0 | `5.0 µm` | **`38.11 s`** | Faster *only* when initial gantry droop exceeds `~0.40 mm`. |

### Summary Conclusions for `T0` (Rapido HF StealthBurner + OptoTap)
1. **Decisive Real-World Winner — Option A (`PS=8` Upstream Single-Pass):** Testing real `M84` motor cutoffs proved that the 80:16 (5:1) geared Z drives only sag **`17.5 µm`** under the Rapido HF StealthBurner's weight during normal `M84` idle. Under real print starts, **pure upstream single-pass `G32`** (`speed: 8.0`, `lift_speed: 35.0`, `sample_retract_dist: 2.0`, `samples: 3`, `samples_result: median`, `samples_tolerance: 0.020`, plus `[bed_mesh] horizontal_move_z: 2`) finishes QGL in **`~15.5s` on back-to-back prints** and **`30.04s` after `M84`**—cutting typical pre-print `QGL + G28 Z + KAMP 3×3` down to **`48–62 seconds`** (down from `~198s`, a **69–75% reduction**) with **zero macro changes**.
2. **Why `SAMPLES=1` Coarse Pass Hurt Normal Starts:** Because Tap #1 on the Rapido HF StealthBurner carries `10–15 µm` of shuttle-seating offset, a 1-sample coarse pass actually *tilts* an already-flat (`<= 17.5 µm`) gantry out to `27.5 µm`, forcing 2 extra fine passes (`37.93s` vs `30.04s`).
3. **Keep `samples: 3` (`median`) for Homing & QGL:** Three-sample median cleanly discards Tap #1 on every corner and converged QGL **10.4 seconds faster** than `SAMPLES=2`.

