# Klipper-Backup 💾 
Klipper backup script for manual or automated GitHub backups 

This backup is provided by [Klipper-Backup](https://github.com/Staubgeborener/klipper-backup).

## Hardware Notes
- **Printer:** Voron 2.4 (350mm, Formbot base kit) with StealthChanger / TapChanger
- **Host:** Raspberry Pi Compute Module 5 Rev 1.0 (Static IP: `192.168.0.46/22`)
- **Mainboard:** BTT Manta M8P V2.0 (USB-to-CAN bridge, `1000000` bitrate, `txqueuelen 128`)
- **Toolheads (StealthBurner + CW2 on both):**
  - **T0 (`purple`, `sb-2.cfg`):** BTT EBB SB2209 RP2040, Phaetus Rapido HF (0.4mm nozzle; UHF upgrade kit available on hand), cheap BMG/CW2 extruder (`rotation_distance: 4.58`), OptoTap
  - **T1 (`EBBCan`, `ebb-sb-rp2040.cfg`):** BTT EBB SB2209 RP2040 (from Formbot kit), Formbot Voron/V6 clone hotend & 0.4mm nozzle, cheap BMG/CW2 extruder (`rotation_distance: 22.9`, `gear_ratio: 50:10`), OptoTap
- **CAN Bus Termination:** Toolhead 0 (T0) has the 120-ohm resistor jumper installed. Toolhead 1 (T1) does not. The Manta M8P mainboard also has the 120-ohm resistor jumper installed, resulting in the required 60-ohm total bus resistance.

## Documentation & Debugging Context
See [`GEMINI.md`](./GEMINI.md) for full architecture notes, operational conventions, and the root-cause analysis for the **second toolchange crash**.
