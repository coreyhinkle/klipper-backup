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

## 4. CoreXY Horizontal Kinematics Qualification (400 mm/s @ 5000 mm/s²)

### Driver & Motor Configuration:
* **Steppers:** LDO-42STH48-2004MAH (CoreXY A/B motors) powered by TMC2209 drivers on 24V bus.
* **Driver Current:** Bumped from stock `0.80 A` $\to$ **`1.00 A`** (`run_current: 1.0` in [`printer.cfg`](../printer_data/config/printer.cfg#L76)) to guarantee torque margin at high accelerations.
* **Rapid Transit Speed:** Bumped `params_fast_speed` from `10000 mm/min` ($166\text{ mm/s}$) $\to$ **`15000 mm/min`** ($250\text{ mm/s}$) in [`toolchanger-2.4.cfg`](../printer_data/config/tapchanger/toolchanger-2.4.cfg#L5).

### Empirical Rapid Torture Move Test:
An automated CoreXY stress routine was executed across the full 350 mm build volume:
* **Move Profile:** 8 consecutive corner-to-corner diagonal rapid moves ($X, Y: [30, 30] \leftrightarrow [320, 320]$) at `400 mm/s` with `5000 mm/s²` acceleration.
* **Execution Time:** **9.47 seconds** total.
* **Verification:** Immediate `G28 X` and `G28 Y` sensorless homing confirmed zero stall-guards or step loss across both axes.

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
