# Printer Software & MCU Firmware Version Manifest

> [!NOTE]
> **Automated Version Tracking (`klipper-backup` Hook):**
> This manifest ([`docs/VERSION_MANIFEST.md`](VERSION_MANIFEST.md) and [`docs/versions.json`](versions.json)) is automatically updated by `~/klipper-backup/utils/update_version_manifest.py` whenever `klipper-backup` runs.
> Every Git commit in this repository is permanently paired with the exact Host OS kernel, Klipper/KTC-Easy commit SHAs, and flashed MCU firmware versions active at that point in time.

---

## 1. Host & CAN Bus Baseline

| Component | Active Specification |
|:---|:---|
| **Host Hardware** | `Raspberry Pi Compute Module 5 Rev 1.0` |
| **Operating System** | `Debian GNU/Linux 12 (bookworm)` |
| **Linux Kernel** | `6.12.109+rpt-rpi-2712` |
| **CAN Interface (`can0`)** | Bitrate `1000000` (1 Mbps), `txqueuelen 128` |

---

## 2. Host Software Stack Versions

| Component | Version / Tag | Commit SHA | Commit Date | Branch |
|:---|:---|:---|:---|:---|
| **klipper** | `v0.13.0-800-g9cb99754` | [`9cb99754`](https://github.com/Klipper3d/klipper/commit/9cb99754) | 2026-10-13 | `master` |
| **klipper-toolchanger-easy** | `e881fe4` | [`e881fe40`](https://github.com/jwellman80/klipper-toolchanger-easy/commit/e881fe40) | 2026-07-30 | `main` |
| **klipper-toolchanger** | `9e69d6a` | [`9e69d6ac`](https://github.com/viesturz/klipper-toolchanger/commit/9e69d6ac) | 2026-10-19 | `main` |
| **moonraker** | `v0.11.0-4-gb559b9a` | [`b559b9a8`](https://github.com/Arksine/moonraker/commit/b559b9a8) | 2026-10-21 | `master` |
| **mainsail** | `v2.19.0` | — | — | — |
| **mainsail-config** | `v1.2.1-1-gff3869a` | [`ff3869a6`](https://github.com/mainsail-crew/mainsail-config/commit/ff3869a6) | 2024-11-24 | `master` |
| **KlipperScreen** | `v0.4.7-205-g34610ce` | [`34610cea`](https://github.com/KlipperScreen/KlipperScreen/commit/34610cea) | 2026-10-25 | `master` |
| **katapult** | `v0.0.1-111-g32584cb` | [`32584cbb`](https://github.com/Arksine/katapult/commit/32584cbb) | 2026-03-05 | `master` |
| **crowsnest** | `v4.1.17-1-g9cc3d4a` | [`9cc3d4af`](https://github.com/mainsail-crew/crowsnest/commit/9cc3d4af) | 2025-10-07 | `master` |
| **Klipper-Adaptive-Meshing-Purging** | `v1.1.2-14-g997838c` | [`997838cb`](https://github.com/kyleisah/Klipper-Adaptive-Meshing-Purging/commit/997838cb) | 2026-09-13 | `main` |
| **sonar** | `v0.2.0-2-g74494cc` | [`74494cc3`](https://github.com/mainsail-crew/sonar/commit/74494cc3) | 2026-09-22 | `main` |
| **moonraker-timelapse** | `v0.0.1-143-gc7fff11` | [`c7fff11e`](https://github.com/mainsail-crew/moonraker-timelapse/commit/c7fff11e) | 2023-12-16 | `main` |
| **klipper-backup** | `v1.3.6-7-g32f42e8` | [`32f42e8e`](https://github.com/Staubgeborener/klipper-backup/commit/32f42e8e) | 2026-10-13 | `main` |

---

## 3. Flashed Microcontroller (MCU) Firmware Matrix

| MCU Identifier | Role / Toolhead | Hardware Board | Chip | CAN UUID | Flashed Firmware (`mcu_version`) | `RECEIVE_WINDOW` |
|:---|:---|:---|:---|:---|:---|:---|
| **`[mcu]`** | Mainboard / USB-to-CAN Bridge | BTT Manta M8P V2.0 | `stm32h723xx` | `3d2cb7337ea8` | **`v0.13.0-800-g9cb99754`** | `192` |
| **`[mcu purple]`** | Toolhead 0 (T0 - Master Reference, StealthBurner Rapido HF) | BTT EBB SB2209 CAN v1.0 (RP2040) | `rp2040` | `4445b54fe351` | **`v0.13.0-800-g9cb99754`** | `192` |
| **`[mcu green]`** | Toolhead 1 (T1 - Secondary, StealthBurner V6) | BTT EBB SB2209 CAN v1.0 (RP2040) | `rp2040` | `a7f6c7cc913a` | **`v0.13.0-800-g9cb99754`** | `192` |
| **`[mcu cyan]`** | Toolhead 2 (T2 - Dragon Burner) | BTT EBB36 CAN v1.2 (STM32G0B1) | `stm32g0b1xx` | `108d45e46fd2` | **`v0.13.0-800-g9cb99754`** | `192` |
| **`[mcu yellow]`** | Toolhead 3 (T3 - Dragon Burner, Standby/Unmounted) | BTT EBB36 CAN v1.2 (STM32G0B1) | `stm32g0b1xx` | `ce0655e3d001` | **`v0.13.0-800-g9cb99754`** | `192` |

---

## 4. Historical Hardware & Software Version Milestones

| Era / Date | Host Hardware & Kernel | Klipper Version | Toolchanger Stack | Key Architectural Changes |
|:---|:---|:---|:---|:---|
| **Jan – Apr 2025**<br>*(Initial Build)* | **BTT CB1** (Allwinner H616, 1 GB RAM, Linux 5.16 sunxi) | Early `v0.12.0` / `v0.13.0` | `viesturz/klipper-toolchanger` *(early 2025)* | Initial Voron 2.4 + StealthChanger bring-up (`T0 purple` + `T1 EBBCan`). Suffered CB1 USB/CAN scheduling latency and early `usb_canbus.c` / `trsync` TTC bugs. |
| **Feb 15–16, 2026**<br>*(`9432ad1` / `0fd67df`)* | **BTT CB1** (`/home/biqu`) | Pre-upgrade `v0.13.0` | `viesturz/klipper-toolchanger` | Switched `T0`/`T1` extruders from StealthChop to SpreadCycle (`stealthchop_threshold: 0`), disabled `kTAMV`, and debugged `trsync` / `Timer too close` errors. |
| **Mar 22, 2026**<br>*(`10d23ca`)* | **Raspberry Pi CM5 Rev 1.0**<br>(Debian 12, `6.12.75+rpt-rpi-2712`) | `v0.13.0-583-g594ec7e1`<br>*(2026-03-21)* | `viesturz/klipper-toolchanger` (`94756df`) | Replaced BTT CB1 with Raspberry Pi CM5 (`/home/zyper`). Picked up multithreaded C `steppersync`, decoupled `serialqueue` locks, and `trsync` `reqclock` homing fixes. |
| **Oct 18–19, 2026**<br>*(`c63775d` / `1ae8296`)* | **Raspberry Pi CM5 Rev 1.0**<br>(`6.12.75+rpt-rpi-2712`, `txqueuelen 128`) | `v0.13.0-800-g9cb99754`<br>*(2026-10-13)* | `viesturz/klipper-toolchanger` + PR #174 | Flashed Manta M8P (`STM32H723`) and `T0`/`T1` (`RP2040`) via Katapult to `v0.13.0-800-g9cb99754` (`192B` CAN `transmit_buf` + `usb_canbus.c` wakeup fix). Locked `txqueuelen 128`. |
| **Oct 23, 2026**<br>*(`f04f72c`)* | **Raspberry Pi CM5 Rev 1.0**<br>(`6.12.75+rpt-rpi-2712`) | `v0.13.0-800-g9cb99754` | **`jwellman80/klipper-toolchanger-easy`** (`e881fe40`) | Migrated from legacy `tapchanger/` configs to upstream **KTC-Easy** (`toolchanger/readonly-configs/`, `toolchanger/tools/`). |
| **Nov 7–10, 2026**<br>*(`9d677c3`)* | **Raspberry Pi CM5 Rev 1.0**<br>(`6.12.109+rpt-rpi-2712`) | `v0.13.0-800-g9cb99754` | `jwellman80/klipper-toolchanger-easy` (`e881fe40`) | Flashed `T2` (`cyan`) and `T3` (`yellow`) BTT EBB36 CAN v1.2 (`STM32G0B1`) Dragon Burner toolboards to `v0.13.0-800-g9cb99754`; renamed `T1` MCU from `EBBCan` to `green`. |
