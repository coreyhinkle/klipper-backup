# Klipper-Backup 💾 
Klipper backup script for manual or automated GitHub backups 

This backup is provided by [Klipper-Backup](https://github.com/Staubgeborener/klipper-backup).

## Hardware Notes
- **Host:** Raspberry Pi Compute Module 5 Rev 1.0
- **Mainboard:** BTT Manta M8P V2.0 (USB-to-CAN bridge, `1000000` bitrate, `txqueuelen 128`)
- **Toolheads:**
  - **T0 (`purple`):** BTT EBB RP2040 (`sb-2.cfg`)
  - **T1 (`EBBCan`):** BTT EBB RP2040 (`ebb-sb-rp2040.cfg`)
- **CAN Bus Termination:** Toolhead 0 (T0) has the 120-ohm resistor jumper installed. Toolhead 1 (T1) does not. The Manta M8P mainboard also has the 120-ohm resistor jumper installed, resulting in the required 60-ohm total bus resistance.

## Documentation & Debugging Context
See [`GEMINI.md`](./GEMINI.md) for full architecture notes, operational conventions, and the root-cause analysis for the **second toolchange crash**.
