# Hardware Specifications & Electronics Inventory

This document details the physical hardware, electronics, toolhead components, and CAN bus network architecture for this **Voron 2.4 (350mm)** multi-tool printer.

---

## 1. Core Machine & Control System
- **Frame & Kinematics:** Voron 2.4 (350mm build volume, Formbot base kit) with flying gantry and CoreXY motion.
- **Host System:** **Raspberry Pi Compute Module 5 (CM5) Rev 1.0** (Debian Bookworm `6.12.75+rpt-rpi-2712`, upgraded from earlier BTT CB1).
  - **Networking:** Dedicated static IP **`192.168.0.46/22`** (gateway `192.168.0.1`) configured via NetworkManager (`Chez CoreCats` profile) to remain outside the router's standard DHCP pool (`192.168.1.1+`).
- **Main Control Board:** **BigTreeTech (BTT) Manta M8P V2.0**
  - **CAN Bridge:** Acts as the primary USB-to-CAN bus bridge (`canbus_uuid: 3d2cb7337ea8`).
  - **Motor Drivers:** TMC2209 stepper drivers on all axes (Z steppers run in SpreadCycle `stealthchop_threshold: 0`).
  - **Driver Cooling:** Active skirt cooling fans.

---

## 2. Power Architecture & 24V DC Budget (6-Tool Fleet)

### Active Hardware Baseline
- **Installed 24V DC Power Supply:** **Mean Well LRS-200-24** (200W, 24V @ 8.8A, passively cooled, from original Formbot kit).
- **Target Upgrade Power Supply:** **Mean Well LRS-350-24** (350W, 24V @ 14.6A, actively cooled, direct drop-in replacement on Voron DIN rail mounts).
- **AC Mains Heated Bed:** Keenovo-style silicone bed heater powered directly via 110V/220V AC mains through an Omron G3NA-210B-DC5 SSR (`PA1`, clamped to `max_power: 0.6` in [`printer.cfg`](../printer_data/config/printer.cfg)). The heated bed draws **0 Watts** from the 24V DC power supply.

### 24V DC Load Breakdown (6-Tool Fleet)

| Subsystem | Typical Draw | Peak / Transient Draw | Notes |
|:---|:---|:---|:---|
| **CoreXY Motion (A/B Steppers)** | ~25W | ~45W | TMC2209 @ `1.0A` RMS under rapid CoreXY direction changes. |
| **Z-Axis Flying Gantry (4x Z Steppers)** | ~5W | ~35W | SpreadCycle; low continuous draw during layer hops, transient peak during 50 mm/s rapid transit & QGL. |
| **Base Electronics & Skirt Cooling** | ~10W | ~12W | Manta M8P V2.0 + Raspberry Pi CM5 host. |
| **Chamber Filtration & Auxiliary Fans** | ~10W | ~15W | Nevermore Micro V5 (`PF7`) + chamber exhaust (`PF8`). |
| **6x Toolhead CAN Boards Logic** | ~12W | ~15W | 6x EBB boards (SB2209 / EBB36) @ ~2W each idle (MCUs + CAN transceivers). |
| **Toolhead NeoPixel Lighting** | ~3W | ~6W | Parked/active tool LEDs. |
| **Active Tool Extruder Stepper** | ~5W | ~8W | Only the currently active toolhead extruder motor is driven. |
| **Active Tool Hotend (T0 Rapido HF)** | ~45W (PID) | **115W (100% PWM)** | Phaetus Rapido ceramic heater core pulls ~4.8A at peak inrush. |
| **Active Tool Hotend (T1-T5 Dragon/V6)** | ~25W (PID) | ~50W (100% PWM) | Standard 40W–50W heater cartridges. |
| **Parked Standby Heat (5 parked tools @ 150°C)** | ~50W–75W | ~75W | ~10W–15W per parked hotend to maintain 150°C in an enclosed chamber. |
| **Parked Toolhead Heatsink Fans** | ~12.5W | ~12.5W | 5x 3010/4010 fans running continuously when hotends are >50°C. |

### Operational Modes & Power Supply Demands

1. **Strict Single-Tool Mode (Cold Standby / Inactive Tools Off at 0°C):**
   - **Continuous Running Load:** $\approx 115\text{--}130\text{ W}$ (within LRS-200 limits).
   - **Peak Transient Load:** Up to **$225\text{--}245\text{ W}$** when T0's Rapido HF ramps up at 100% PWM while CoreXY and Z axes are moving fast.
   - **Status:** Operates at 110%–120% of the LRS-200's continuous rating during heating spikes, risking voltage dips.

2. **Standard Multi-Material Toolchanger Mode (150°C Standby + Overlap Preheat):**
   - Real-world multi-tool printing requires standby temperatures (~150°C) on parked tools to avoid 45-second heating dwells per toolchange.
   - Furthermore, slicers and macros use **preheat overlap** (ramping the incoming tool 15–20s before tool swap while the outgoing tool is still finishing its extrusion).
   - **Continuous Running Load:** $\approx 185\text{--}210\text{ W}$.
   - **Peak Transient Load (Overlap Swap with T0):** Up to **$300\text{--}320\text{ W}$**.
   - **Status:** **Exceeds LRS-200 capacity.** Mandates upgrade to **Mean Well LRS-350-24**.

### Failure Modes of an Overloaded LRS-200-24

- **Fire Hazard:** **None.** Genuine Mean Well power supplies are equipped with Over-Current Protection (OCP / Hiccup Mode) and Over-Temperature Protection (OTP). They safely cut or pulse output power rather than overheating or catching fire.
- **Electrical Hardware Damage:** **None.** Stepper drivers and MCUs feature Under-Voltage Lockout (UVLO) to prevent electrical damage.
- **Mechanical Damage Risk (Gantry & Docking):** **HIGH.**
  - If the 24V supply enters hiccup mode mid-move, the 4 Z steppers lose holding torque, causing the heavy flying gantry to drop or rack.
  - If power cuts while the carriage shuttle is entering or exiting a tool dock ($Y \le 35\text{ mm}$), unpowered momentum can cause the carriage to bind or collide with dock arms, **snapping 3D-printed dock brackets, breaking carriage latch tabs, or bending dock pins.**
- **Operational Failures:**
  - **CAN Bus Dropout:** Voltage sag below ~20V starves toolhead buck converters, dropping CAN transceivers and triggering fatal Klipper aborts: `Lost communication with MCU 'EBBCan'` or `Timer too close`.
  - **ADC Out of Range:** Voltage dips induce noise on thermistor/PT1000 analog reference lines, tripping safety halts.
  - **Thermal Shutdown in Electronics Bay:** The fanless LRS-200 runs near 100% capacity in a 45°C enclosed electronics bay, tripping OTP after 1–2 hours.

### Interim Operating Rules (While Running LRS-200-24)
Before upgrading to the LRS-350-24, adhere strictly to these operational safeguards:
1. **Cold Standby Only:** Set parked tool standby temperatures to `0°C` in slicer profiles and toolchange macros. Never hold parked tools at temperature.
2. **Sequential Preheating:** Heat the bed first, then heat one hotend at a time during `PRINT_START`. Never heat multiple tools in parallel.
3. **Soft-Cap T0 Rapido HF:** Add `max_power: 0.80` under `[extruder]` in [`toolchanger/tools/T0.cfg`](../printer_data/config/toolchanger/tools/T0.cfg) to clamp maximum ceramic heater draw from 115W down to ~92W.
4. **Fusing:** Ensure the 24V trunk to the distribution board (FYSETC Hexa Distro Fusion V1.3 in the Fanny Pack) has individual 5A or 7.5A fuses per toolhead branch.

---

## 3. Toolhead Configurations

Both toolheads utilize the StealthBurner toolhead assembly with Clockwork 2 (CW2) extruders mounted to the StealthChanger carriage shuttle.

### Toolhead 0 (T0 - Reference Tool, Purple StealthBurner)
- **Toolhead MCU:** BTT EBB SB2209 RP2040 (`[mcu purple]`, `canbus_uuid: 4445b54fe351`).
- **Hotend:** Phaetus Rapido HF (0.4mm nozzle). User also possesses an uninstalled UHF upgrade kit on hand.
  - Temperature Sensor: PT1000/PT100 via onboard MAX31865 RTD amplifier (`purple:gpio9`).
- **Extruder:** Clockwork 2 (CW2) with Amazon BMG-style dual-gear kit (`B09WQYHBZY` / Formbot clone gears).
  - Rotation Distance: `22.9` with `gear_ratio: 50:10`.
  - Motor: TMC2209 at `run_current: 0.650A` (SpreadCycle).
- **Z Probe:** OptoTap (`^purple:gpio22`, configured in [`sb-2.cfg`](../printer_data/config/sb-2.cfg)).
  - Configured Z Offset: `-0.450 mm`.
- **Accelerometers:** Onboard ADXL345 (`purple:gpio1`).
- **Lighting:** StealthBurner RGB LEDs (`sb_leds_t0`).
- **Operational Status:** Serves as the global coordinate reference (`gcode_x_offset: 0`, `gcode_y_offset: 0`, `gcode_z_offset: 0`).

### Toolhead 1 (T1 - Green StealthBurner)
- **Toolhead MCU:** BTT EBB SB2209 RP2040 (`[mcu green]`, `canbus_uuid: a7f6c7cc913a`, from original Formbot kit).
- **Hotend:** Voron / V6 clone hotend (0.4mm nozzle, from Formbot kit).
  - Temperature Sensor: PT1000/PT100 via onboard MAX31865 RTD amplifier (`EBBCan:gpio9`).
- **Extruder:** Clockwork 2 (CW2) with Formbot kit / Amazon (`B09WQYHBZY`) BMG dual-gear kit.
  - Rotation Distance: `22.9` with `gear_ratio: 50:10`.
  - Motor: TMC2209 at `run_current: 0.650A` (SpreadCycle).
- **Z Probe:** OptoTap (`^EBBCan:gpio22`, configured in [`ebb-sb-rp2040.cfg`](../printer_data/config/ebb-sb-rp2040.cfg)).
  - Configured Z Offset: `-0.80 mm`.
- **Accelerometers:** Onboard ADXL345 (`EBBCan:gpio1`).
- **Lighting:** StealthBurner RGB LEDs (`sb_leds_t1`).

### Toolhead Electronics Spares & Inventory
- **Spare StealthBurner CAN Toolboard:** **BigTreeTech EBB SB2209/SB2240 CAN V1.0** on hand in workshop inventory. Direct replacement for T0/T1 StealthBurner CW2 setups.

---

## 4. Umbilical Cable Suspension & Wire Tension
- **Overhead Harness Setup:** Toolhead CAN and power cables run overhead to keep the gantry clear.
- **Current Baseline:** 
  - T0's umbilical suspension wire is taut.
  - T1's overhead cable currently has slight slack compared to T0.
  - **Mechanical Effect:** Because OptoTap measures carriage deflection against magnetic preloads, umbilical slack at extreme reach (`X=50, Y=275`, rear-left corner) exerts slight variable vertical drag, resulting in minor ~2.5 µm (1 microstep) tap variances during initial probing passes before settling.
- **Umbilical Management & Expansion Architecture:**
  - **Mechanical Anchor (Top Rear):** **DraftShift / N3MI Voron 350 6-Tool Umbilical Plate** and **Fanny Pack** enclosure printed and mostly installed on the top-rear frame over the exhaust port opening. Houses integrated ECAS04 push-fit collets for PTFE guide tubes, integrated wire duct, and relief anchor points for 0.3x3mm flat spring steel spines.
  - **Spine System:** 0.3mm x 3mm flat coil spring steel spine along each umbilical to form a self-supporting vertical arc into the top hat, eliminating downward weight/drag on the OptoTap carriage.
  - **Fleet Interchangeability:** Universal umbilical spans sized for 350mm reach (~740mm). Terminated with Molex Micro-Fit 3.0 for active StealthBurners (CW2 + EBB SB2209), and modularly compatible with future DragonBurners (Sherpa Mini + EBB36) without altering the top plate or CAN bus backbone.
  - **Electrical Hub Separation:** Distribution board is enclosed inside the rear-mounted **Fanny Pack** enclosure, mechanically isolated from umbilical tugging loads with integrated wire ducting down the rear spine.

---

## 5. CAN Bus Network Architecture
- **Bitrate:** `1000000` (1 Mbps).
- **Network Interface Management:** Managed via `systemd-networkd` (`/etc/systemd/network/25-can.network`).
- **Queue Length (`txqueuelen`):** **`128`** via `/etc/udev/rules.d/10-can.rules`.
  > [!IMPORTANT]
  > Do **NOT** change `txqueuelen` to `1024` or create a custom `can0.service`. Official Klipper and Esoterical CAN bus documentation explicitly specifies `txqueuelen 128` to avoid kernel buffer bloat that causes `"Timer too close"` fatal halts during dense multi-axis moves.
- **Topology & Termination Jumpers:**
  - Star / Hub topology using a centralized distribution board.
  - To achieve the required **60-ohm** total bus resistance:
    - **Toolhead 0 (T0):** 120-ohm resistor jumper is **ON**.
    - **Toolhead 1 (T1):** 120-ohm resistor jumper is **OFF**.
    - **Manta M8P Mainboard:** 120-ohm resistor jumper is **ON**.
  - Total parallel resistance: $120\Omega \parallel 120\Omega = 60\Omega$.

---

## 6. Chamber & Auxiliary Accessories
- **Enclosure Doors & Clearances:** **DraftChange Front Door Extenders (Door Buffer)** installed. Custom outward-offset hinges and perimeter spacers push the front acrylic doors outward past the frame boundary.
  - **Dock Clearance:** Provides the necessary clearance for the front-mounted StealthChanger docks ($Y \approx 1\text{--}2\text{ mm}$, $Z \approx 298\text{ mm}$) and carriage shuttle, preventing contact during toolchanges.
  - **Modular Dock Crossbar Mounting:** Because the door buffer extends the front enclosure forward, all Modular Docks mounted to the front 2020 crossbar strictly require **`Top_Crossbar.stl`** and **`Bottom_Crossbar.stl`** (from `DraftShift/ModularDock/STLs/Frame/2020/`). The stock `Top.stl` and `Bottom.stl` lack the extended drop/clamp geometry and must NOT be used.
  - **Front Accessory Mounting:** The forward clearance also enables mounting front-bed accessories (such as a front-edge nozzle brush or purge wiper) without door interference, utilizing the forward-shifted toolhead reach.
- **Air Filtration:** Nevermore Micro V5 carbon filter (`[fan_generic Nevermore]` on pin `PF7`).
- **Display:** BigTreeTech Mini12864 RGB LCD with Neopixel backlights.
- **Cooling:** Skirt fans for electronics bay cooling and rear chamber exhaust fan.

