# Klipper MCU Firmware Upgrade Guide (Katapult / CANBus)

This guide documents the standardized, repeatable procedure for upgrading Klipper firmware across the microcontrollers on the Voron 2.4 StealthChanger.

---

## 1. System Architecture & Bus Topology

The system uses a **USB-to-CAN bridge architecture**. The mainboard acts as the gateway between the host computer and the toolhead MCUs.

```
                  ┌─────────────────────────────────┐
                  │    Host Computer (Pi / CB1)     │
                  └────────────────┬────────────────┘
                                   │ USB (gs_usb driver)
                                   ▼
                  ┌─────────────────────────────────┐
                  │   BTT Manta M8P V2.0 Mainboard  │  <-- USB-to-CAN Bridge
                  └────────────────┬────────────────┘
                                   │ CAN Bus (1 Mbps, txqueuelen 128)
         ┌─────────────────────────┴─────────────────────────┐
         ▼                                                   ▼
┌─────────────────────────────────┐         ┌─────────────────────────────────┐
│ Toolhead 0 (BTT EBB SB2209)     │         │ Toolhead 1 (BTT EBB SB2209)     │
│ MCU: RP2040                     │         │ MCU: RP2040                     │
│ CAN UUID: 4445b54fe351          │         │ CAN UUID: a7f6c7cc913a          │
└─────────────────────────────────┘         └─────────────────────────────────┘
```

### Microcontroller Profiles

| Board | Role | MCU Chip | Klipper Mode | Bootloader / Katapult Mode | Config Template |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **BTT Manta M8P V2.0** | Mainboard (`mcu`) | STM32H723 (25MHz crystal) | CAN UUID: `3d2cb7337ea8`<br>Offset: 128 KiB (`0x8020000`)<br>USB-to-CAN Bridge (`PD0/PD1`) | USB Serial (`0x1d50:0x6177`)<br>`/dev/serial/by-id/usb-katapult_stm32h723xx_1F001C000751313433343333-if00` | `~/klipper/config.manta` |
| **BTT EBB SB2209** | Toolhead 0 (`mcu purple`) | RP2040 | CAN UUID: `4445b54fe351`<br>Offset: 16 KiB (`0x10004000`)<br>CAN: `gpio4/gpio5`, Init: `gpio26` | CAN Katapult UUID: `4445b54fe351` | `~/klipper/config.ebb` |
| **BTT EBB SB2209** | Toolhead 1 (`mcu EBBCan`) | RP2040 | CAN UUID: `a7f6c7cc913a`<br>Offset: 16 KiB (`0x10004000`)<br>CAN: `gpio4/gpio5`, Init: `gpio26` | CAN Katapult UUID: `a7f6c7cc913a` | `~/klipper/config.ebb` |

> [!IMPORTANT]
> **Why Order of Operations Matters:**
> The Manta M8P is the gateway providing the `can0` interface. When the Manta enters Katapult mode, it ceases bridging CAN packets and enumerates as a pure USB serial device. This causes `can0` to drop. Therefore, **both toolheads must be flashed first over CAN while the Manta is running**, and the Manta is flashed **last**.

---

## 2. Key References & Community Guides

* **[Esoterical's CANBus Guide (Main)](https://canbus.esoterical.online/)** – The definitive community reference for Klipper CAN bus setups.
* **[Esoterical Updating Firmware Guide](https://canbus.esoterical.online/Updating_Firmware.html)** – Detailed walkthrough of Katapult / `flashtool.py` firmware updates.
* **[Esoterical Troubleshooting Guide](https://canbus.esoterical.online/Troubleshooting.html)** – Common issues, missing UUIDs, and CAN recovery procedures.
* **[Esoterical Katapult Bootloader Guide](https://canbus.esoterical.online/CanBoot_Flashing.html)** – How Katapult works and manual entry instructions.
* **[Esoterical GitHub Repository](https://github.com/Esoterical/voron_canbus)** – Upstream source and documentation repository.
* **[Katapult (formerly CanBoot) Official Repository](https://github.com/Arksine/katapult)** – Source code and documentation for `flashtool.py`.
* **[Klipper Official CANBus Documentation](https://www.klipper3d.org/CANBUS.html)** – Official Klipper docs on CAN bus configuration.
* **[BTT EBB SB2209 GitHub](https://github.com/bigtreetech/EBB)** – Pinouts, schematics, and hardware specifications for toolhead boards.
* **[BTT Manta M8P V2.0 GitHub](https://github.com/bigtreetech/Manta-M8P)** – Pinouts, jumpers, and manual for Manta M8P V2.0.

---

## 3. Pre-Upgrade Preparation & Backups

Before beginning the firmware compilation or flashing process:

1. **Verify Existing Configuration Templates:**
   Ensure the saved templates exist in `~/klipper`:
   ```bash
   ls -la ~/klipper/config.manta ~/klipper/config.ebb
   ```
2. **Back Up Existing Binaries:**
   ```bash
   cp ~/klipper/config.manta ~/klipper/config.manta.bak
   cp ~/klipper/config.ebb ~/klipper/config.ebb.bak
   [ -f ~/klipper/out/klipper.bin ] && cp ~/klipper/out/klipper.bin ~/klipper/out/klipper_prev.bin
   ```
3. **Verify CAN Bus State:**
   Confirm `can0` is active and operating at `txqueuelen 128`:
   ```bash
   ip -details link show can0
   ```
4. **Stop Klipper Host Service:**
   `flashtool.py` requires exclusive access to the CAN / serial devices. Stop Klipper to prevent communication collisions:
   ```bash
   sudo systemctl stop klipper
   ```

---

## 4. Phase-by-Phase Execution Plan

### Phase 1: Update Host Klipper Software
```bash
cd ~/klipper
git fetch origin
git pull
```
> [!NOTE]
> `klipper-toolchanger` symlinks extra modules into `klippy/extras/`. These symlinks are untracked by git and will persist cleanly across fast-forward git pulls.

---

### Phase 2: Compile & Flash Toolhead 0 (EBB SB2209 RP2040)

1. **Load RP2040 Config & Build:**
   ```bash
   cd ~/klipper
   make clean
   cp config.ebb .config
   make olddefconfig
   make -j4
   ```
2. **Request Katapult Bootloader on Toolhead 0:**
   ```bash
   python3 ~/katapult/scripts/flashtool.py -i can0 -r -u 4445b54fe351
   ```
3. **Verify Katapult Node Enumeration:**
   ```bash
   python3 ~/katapult/scripts/flashtool.py -i can0 -q
   ```
   *Expected output: Detected Katapult node with UUID `4445b54fe351`.*
4. **Flash Klipper Firmware:**
   ```bash
   python3 ~/katapult/scripts/flashtool.py -i can0 -u 4445b54fe351 -f ~/klipper/out/klipper.bin
   ```
5. **Verify Return to Normal Mode:**
   The toolhead automatically restarts and resumes running Klipper application code.

---

### Phase 3: Flash Toolhead 1 (EBB SB2209 RP2040)

*Because Toolhead 1 shares identical RP2040 hardware and pinouts, no re-compilation is necessary.*

1. **Request Katapult Bootloader on Toolhead 1:**
   ```bash
   python3 ~/katapult/scripts/flashtool.py -i can0 -r -u a7f6c7cc913a
   ```
2. **Verify Katapult Node Enumeration:**
   ```bash
   python3 ~/katapult/scripts/flashtool.py -i can0 -q
   ```
   *Expected output: Detected Katapult node with UUID `a7f6c7cc913a`.*
3. **Flash Klipper Firmware:**
   ```bash
   python3 ~/katapult/scripts/flashtool.py -i can0 -u a7f6c7cc913a -f ~/klipper/out/klipper.bin
   ```
4. **Verify Return to Normal Mode:**
   Toolhead 1 reboots back into Klipper application code.

---

### Phase 4: Compile & Flash Mainboard (BTT Manta M8P V2.0)

1. **Load Manta STM32H723 Config & Build:**
   ```bash
   cd ~/klipper
   make clean
   cp config.manta .config
   make olddefconfig
   make -j4
   ```
2. **Request Katapult Bootloader on Manta via CAN:**
   ```bash
   python3 ~/katapult/scripts/flashtool.py -i can0 -r -u 3d2cb7337ea8
   ```
3. **Verify USB Serial Enumeration:**
   Once Manta enters Katapult, `can0` temporarily goes down and the board appears as a USB serial device:
   ```bash
   ls -la /dev/serial/by-id/usb-katapult*
   ```
   *Expected path: `/dev/serial/by-id/usb-katapult_stm32h723xx_1F001C000751313433343333-if00`.*
4. **Flash Klipper Firmware via Serial:**
   ```bash
   python3 ~/katapult/scripts/flashtool.py -f ~/klipper/out/klipper.bin -d /dev/serial/by-id/usb-katapult_stm32h723xx_1F001C000751313433343333-if00
   ```
5. **Wait for Manta Reboot & Interface Restoration:**
   The Manta reboots, re-establishes the USB-to-CAN bridge, and the kernel brings `can0` back up.

---

### Phase 5: Verification & System Start

1. **Verify `can0` Network Interface:**
   ```bash
   ip link show can0
   ```
2. **Restart Klipper Service:**
   ```bash
   sudo systemctl start klipper
   ```
3. **Verify All MCUs in Moonraker / Mainsail:**
   Check printer state and logs:
   ```bash
   curl -s http://localhost:7125/printer/info | jq .
   ```
   Confirm all three MCUs (`mcu`, `mcu purple`, `mcu EBBCan`) are reported as connected, synchronized, and running matching software versions.

---

## 5. Troubleshooting & Contingency Protocols

### Problem A: Node Does Not Enter Katapult via Software Command (`-r`)
* **Symptom:** `flashtool.py -i can0 -r -u <uuid>` times out or node does not appear in `-q`.
* **Remedy (Double-Click Reset):**
  * EBB SB2209 Toolheads: Press the physical `RST` button twice rapidly on the toolhead board. The status LED will start blinking continuously indicating Katapult mode.
  * Manta M8P: Press the physical `Reset` button twice rapidly on the mainboard.

### Problem B: `can0` Interface Missing After Mainboard Flash
* **Symptom:** `ip link show can0` returns `Device "can0" does not exist`.
* **Root Cause:** Firmware was compiled without USB-to-CAN bridge mode, or device failed to enumerate.
* **Remedy:**
  1. Check `dmesg | tail -n 20` for USB enumeration errors.
  2. Put Manta into Katapult via double-click reset.
  3. Re-verify `config.manta` has `CONFIG_USBCANBUS=y` and `CONFIG_STM32_CANBUS_PD0_PD1=y`.
  4. Recompile and reflash via USB serial.

### Problem C: `txqueuelen` Resets
* **Symptom:** `ip link show can0` shows `qlen 10` instead of `qlen 128`.
* **Remedy:** Re-apply production setting:
  ```bash
  sudo ip link set can0 txqueuelen 128
  ```
