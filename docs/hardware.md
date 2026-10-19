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

## 2. Toolhead Configurations

Both toolheads utilize the StealthBurner toolhead assembly with Clockwork 2 (CW2) extruders mounted to the StealthChanger carriage shuttle.

### Toolhead 0 (T0 - Reference Tool, Purple StealthBurner)
- **Toolhead MCU:** BTT EBB SB2209 RP2040 (`[mcu purple]`, `canbus_uuid: 4445b54fe351`).
- **Hotend:** Phaetus Rapido HF (0.4mm nozzle). User also possesses an uninstalled UHF upgrade kit on hand.
  - Temperature Sensor: PT1000/PT100 via onboard MAX31865 RTD amplifier (`purple:gpio9`).
- **Extruder:** Clockwork 2 (CW2) with Amazon BMG-style dual-gear kit (`B09WQYHBZY` / Formbot clone gears).
  - Rotation Distance: `22.9` with `gear_ratio: 50:10`.
  - Motor: TMC2209 at `run_current: 0.650A` (SpreadCycle).
- **Z Probe:** OptoTap (`^purple:gpio22`, configured in [`sb-2.cfg`](../printer_data/config/sb-2.cfg)).
  - Configured Z Offset: `-0.800 mm` (aligned with T1 `-0.80 mm`).
- **Accelerometers:** Onboard ADXL345 (`purple:gpio1`).
- **Lighting:** StealthBurner RGB LEDs (`sb_leds_t0`).
- **Operational Status:** Serves as the global coordinate reference (`gcode_x_offset: 0`, `gcode_y_offset: 0`, `gcode_z_offset: 0`).

### Toolhead 1 (T1 - Green StealthBurner)
- **Toolhead MCU:** BTT EBB SB2209 RP2040 (`[mcu EBBCan]`, `canbus_uuid: a7f6c7cc913a`, from original Formbot kit).
- **Hotend:** Voron / V6 clone hotend (0.4mm nozzle, from Formbot kit).
  - Temperature Sensor: PT1000/PT100 via onboard MAX31865 RTD amplifier (`EBBCan:gpio9`).
- **Extruder:** Clockwork 2 (CW2) with Formbot kit / Amazon (`B09WQYHBZY`) BMG dual-gear kit.
  - Rotation Distance: `22.9` with `gear_ratio: 50:10`.
  - Motor: TMC2209 at `run_current: 0.650A` (SpreadCycle).
- **Z Probe:** OptoTap (`^EBBCan:gpio22`, configured in [`ebb-sb-rp2040.cfg`](../printer_data/config/ebb-sb-rp2040.cfg)).
  - Configured Z Offset: `-0.80 mm`.
- **Accelerometers:** Onboard ADXL345 (`EBBCan:gpio1`).
- **Lighting:** StealthBurner RGB LEDs (`sb_leds_t1`).

---

## 3. Umbilical Cable Suspension & Wire Tension
- **Overhead Harness Setup:** Toolhead CAN and power cables run overhead to keep the gantry clear.
- **Tension Wire Note:** 
  - T0's umbilical suspension wire is taut.
  - T1's overhead cable currently has slight slack compared to T0.
  - **Mechanical Effect:** Because OptoTap measures carriage deflection against magnetic preloads, umbilical slack at extreme reach (`X=50, Y=275`, rear-left corner) exerts slight variable vertical drag, resulting in minor ~2.5 µm (1 microstep) tap variances during initial probing passes before settling.
  - **Hardware Plan:** Install dedicated metal tension wire to ensure both toolhead umbilicals maintain identical, taut suspension across the entire 350mm build envelope.

---

## 4. CAN Bus Network Architecture
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

## 5. Chamber & Auxiliary Accessories
- **Enclosure Doors & Clearances:** **DraftChange Front Door Extenders** installed. Custom outward-offset hinges and perimeter spacers push the front acrylic doors outward past the frame boundary.
  - **Dock Clearance:** Provides the necessary clearance for the front-mounted StealthChanger docks ($Y \approx 1\text{--}2\text{ mm}$, $Z \approx 298\text{ mm}$) and carriage shuttle, preventing contact during toolchanges.
  - **Front Accessory Mounting:** The forward clearance also enables mounting front-bed accessories (such as a front-edge nozzle brush or purge wiper) without door interference, utilizing the forward-shifted toolhead reach.
- **Air Filtration:** Nevermore Micro V5 carbon filter (`[fan_generic Nevermore]` on pin `PF7`).
- **Display:** BigTreeTech Mini12864 RGB LCD with Neopixel backlights.
- **Cooling:** Skirt fans for electronics bay cooling and rear chamber exhaust fan.

