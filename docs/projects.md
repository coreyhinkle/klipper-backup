# Master Project & Print Prioritization Queue

This document tracks all active, staged, and planned hardware and printing projects across the **Voron 2.4 (350mm) StealthChanger**, **Prusa MK4S MMU3**, and workshop—organized by priority and **split by printer bed fit (`MK4S` vs `Voron 2.4`)** so you know **what is already printed, what still needs printing before vacation, what to build on vacation, and what waits until you are home**.

> [!IMPORTANT]
> **Critical Dependency — Voron ABS Printing Is Blocked by the Tophat:**
> The Voron 2.4's top frame is currently open for the front StealthChanger docks and umbilicals, so **the Voron cannot safely print ABS/ASA (including the 7 large StealthMax parts) until the Extrusion Tophat is printed, assembled, and enclosed.** All ABS/ASA prints until then must run on the **enclosed Prusa MK4S MMU3 (`250 × 210 mm`)**.

---

## 1. At-a-Glance Priority Queue

| Priority | Project | Current Status | Remaining Prints (Printer & Material) | Blockers / Dependencies |
|:---:|:---|:---|:---|:---|
| **P0** | **StealthChanger Extrusion Tophat (350mm)** | 🔴 **CRITICAL BLOCKER** *(STLs staged; extrusions cut; hardware owned)* | **MK4S (ABS/ASA, ~210g / ~8h 53m):** `18×` [`corner_panel_clip_4mm_x8.stl`](../models/tophat/corner_panel_clip_4mm_x8.stl), `18×` [`midspan_panel_clip_4mm_x7.stl`](../models/tophat/midspan_panel_clip_4mm_x7.stl), `1×` [`LDO Tophat Hinge (1mm foam).stl`](../models/tophat/LDO%20Tophat%20Hinge%20%281mm%20foam%29.stl) *(rotate 90° on MK4S bed)* | **Unlocks Voron ABS printing.** Still need: **check ASAP at home if cut extrusions are drilled/tapped**, buy/cut or order `4×` `483×223×3mm` panels |
| **P0** | **Dock Spring-Steel Silicone Stoppers** | 🟡 **6× Bodies DONE** *(MK4S #1165)* | **MK4S (ABS):** `6×` [`Spring_Steel_Clamp.stl`](../models/modular_dock_blockers/Spring_Steel_Clamp.stl) (**Accent**) + `3–6×` [`Nozzle_Pad_Mold_A/B`](../models/modular_dock_blockers/Nozzle_Pad_Molds_Plate.stl) (**White**) | Print clamps & extra mold sets on MK4S now; cast RTV on vacation (**uses M3 inserts + M3×8 BHCS**, not M2) |
| **P1** | **StealthMax v2 (`DOPM`, 300mm)** | 🟡 **8 Small Parts DONE** *(MK4S #1159–1164)*; 📦 **HW Ordered** | **MK4S (ABS):** 7 small parts (incl. missing `M_2.8`)<br>**Voron 2.4 (ABS/ASA):** **7 large parts** (`>250 mm`) | **7 large parts wait for Tophat enclosure on Voron.** Download `M_2.8_BowdenTubeAccessLatch.stl`; fan/HEPA/sponge cord ordered, sensors likely owned |
| **P1** | **`T2` (`cyan`), `T3` (`yellow`) & `T4` Toolheads** | 🟡 **Plastics Printed** *(Unassembled; `T2` flashed/PID/Tap done; `T3`/`T4` hardware believed owned)* | **Done** *(verify if any spare/accent parts needed)* | **Vacation:** Pack & build/solder **`T3` + `T4`** (verify CAN boards/hotends/OptoTaps/gears before packing). **Home:** Finish `T2` brace/dock & calibrate `T2`–`T4` |
| **P1** | **Nudge Probe Re-Measure & Copper Screws** | 🟡 **Ready When Home** | None *(copper `M3×6` screws owned)* | Swap copper screws (optional) & run `SET_NUDGE_POSITION` with `T0` |
| **P1** | **Fanny Pack + Hexa Distro + Umbilicals** | 🟡 **Built w/ CEB v1.0** *(Not mounted)* | Check at home if `[a]_ptfe_ramp` / collet parts need printing (**ABS/ASA**) | Waiting on **FYSETC Hexa Distro Fusion V1.3** (AliExpress) + `0.3×3mm` spring steel spines |
| **P2** | **24V PSU (`LRS-350-24`) & CM5 Cooling** | 🟡 **CM5 Passive OK** / 💡 **PSU Queued** | None *(drop-in DIN PSU)* | PSU timing TBD (keep parked tools at `0°C` cold standby until upgraded); CM5 passive + `skirt1` OK |
| **P3** | **`T5`+ (Additional Fleet Expansion)** | ⚪ **Planned** | Dragon Burner v8 (`Cowl_NoProbe.stl`), Sherpa Mini R2, `DragonBurner.stl` backplates, Crossbar Docks | Ingest official STLs via `stl-link` if expanding beyond `T4` |
| **P3** | **Front-Bed Nozzle Brush & USB Camera** | ⚪ **Waiting on Camera** *(Brushes owned)* | Front-edge bed/frame brush mount + ESP32-S3 USB camera mount (**ABS/ASA**) | Waiting on **3-In-1 USB Camera Module** (AliExpress) |
| **Ongoing** | **Gridfinity & Workshop Utility** | 🟢 **Active Anytime** | Bins, battery/tool trays, spool labels (**PLA / PETG**) | Non-blocking; run between ABS/ASA batches |

---

## 2. Print Plan by Printer & Sequence

### Phase 1: Prusa MK4S (`250 × 210 mm`) — Print Now / Pre-Departure (ABS)

#### 1A. Extrusion Tophat Hardware Prints *(Highest Priority — Unlocks Voron ABS Enclosure)*
- **Already Owned:** Cut 2020 extrusions (`4× 250 mm`, `8× 470 mm`), M3 roll-in T-nuts, M3×8/M3×12 SHCS, M5×16 BHCS, `625-2RS` bearings, M5 1 mm shims, `1×8 mm` foam sealing tape, and existing Voron top roof panel.
- **To Print on MK4S (~209.5 g / ~8h 53m ABS/ASA, STLs staged in [`models/tophat/`](../models/tophat/)):**
  - [ ] **`1×` [`LDO Tophat Hinge (1mm foam).stl`](../models/tophat/LDO%20Tophat%20Hinge%20%281mm%20foam%29.stl)** (`90.04 × 211.12 × 42.00 mm`, matches your `1×8 mm` foam tape — **must be rotated 90°** on MK4S `250 × 210 mm` bed; `~4h 35m`, `125.3 g`).
  - [ ] **`18×` [`corner_panel_clip_4mm_x8.stl`](../models/tophat/corner_panel_clip_4mm_x8.stl)** (`~2h 39m`, `50.9 g`).
  - [ ] **`18×` [`midspan_panel_clip_4mm_x7.stl`](../models/tophat/midspan_panel_clip_4mm_x7.stl)** (`~1h 39m`, `33.4 g`).
- **Non-Print Tophat Blockers to Close Off the Voron:**
  - [ ] **CHECK ASAP WHEN HOME:** Verify whether the cut `4× 250 mm` + `8× 470 mm` 2020 extrusions have blind-joint access holes / M5 taps (believed cut, **not drilled**).
  - [ ] **Buy/cut locally or order:** **`4×` side perimeter panels (`483 × 223 × 3 mm` acrylic/polycarbonate, ~$45)** *(none on hand currently)*.

#### 1B. Dock Spring-Steel Silicone Stoppers *(Small MK4S Parts for Vacation Casting)*
- **White ABS (Slot 1):**
  - [ ] [`Nozzle_Pad_Mold_A.stl`](../models/modular_dock_blockers/Nozzle_Pad_Mold_A.stl) & [`Nozzle_Pad_Mold_B.stl`](../models/modular_dock_blockers/Nozzle_Pad_Mold_B.stl) — `NOZZLE~1.BGC` (`1×` set) is staged on USB; **recommend slicing `3×–6×` mold sets on one plate** because Permatex Ultra Red RTV requires **24 hours per cure** (`1` mold = `6` days for `6` wipers).
- **Space Grey ABS (Slot 4):**
  - [x] **`6×` [`Spring_Steel_Body.stl`](../models/modular_dock_blockers/Spring_Steel_Body.stl)** — **DONE** (MK4S #1165).
- **Accent ABS (Per-Tool Accent Spools):**
  - [ ] **`6×` [`Spring_Steel_Clamp.stl`](../models/modular_dock_blockers/Spring_Steel_Clamp.stl)** *(planned in each toolhead's accent color)*.

#### 1C. StealthMax v2 (`DOPM`) — Small MK4S Parts
- **Space Grey ABS (Slot 4) — *Already Completed (#1159–#1164):***
  - [x] `Spacer.stl`, `M_2.3_HEPACover.stl`, `1.4c_CableEntryCap_Closed.stl`, `1×` `1.3a_BowdenCap_Closed.stl`, `0b.1_DirectOnPanelMountScrew_DOPM.stl`, `0b.2_ExhaustPipe_DOPM.stl`, `M_0b.0_ChamberExhaustConnection_DOPM.stl`, `3.2_LidFrontHinge.stl`, `M_1b.1_RemovableAirDuct_DOPM.stl`.
- **Space Grey ABS (Slot 4) — *Remaining Small Parts on MK4S:***
  - [ ] **`1×`** [`1.3a_BowdenCap_Closed.stl`](../models/stealthmax/01_Chambers_and_Mount/1.3a_BowdenCap_Closed.stl) *(2nd cap — upstream BOM requires `2×`)*.
  - [ ] **`1×`** `M_2.8_BowdenTubeAccessLatch.stl` *(missing locally; pull from upstream Nevermore StealthMax DOPM repo via `stl-link`)*.
  - *(Note: Delete stale `SPRING~1.BGC` and `STE~633C.BGC` from MK4S USB so they aren't run by mistake.)*
- **Accent ABS — *Remaining Small Parts on MK4S:***
  - [ ] **`3×`** [`3.3_LidLatch.stl`](../models/stealthmax/03_Lid_and_Hardware/3.3_LidLatch.stl) *(print vertically, 6–7 perimeters, thick bridges OFF, gap fill OFF, outer-first walls)*.
  - [ ] **`1×`** [`1.2_AirExhaustSlider.stl`](../models/stealthmax/01_Chambers_and_Mount/1.2_AirExhaustSlider.stl)
  - [ ] **`1×`** [`04_Isiks_PCB_Mount/Latch.stl`](../models/stealthmax/04_Isiks_PCB_Mount/Latch.stl) *(replaces `M_2.6a_PicoAccessLatch.stl` — do NOT print `M_2.6a`)*.
  - [ ] **`1×`** [`M_2.7_IntakeSensorStackAccessLatch.stl`](../models/stealthmax/03_Lid_and_Hardware/M_2.7_IntakeSensorStackAccessLatch.stl)
  - [ ] **`1×`** [`M_2.5a_LogoCap.stl`](../models/stealthmax/03_Lid_and_Hardware/M_2.5a_LogoCap.stl)
  - [ ] *(Only if switching from `M_2.2a` to `M_2.2b` no-handles lid:)* **`2×`** [`2.9_CarbonBasketLidHandle_Spare.stl`](../models/stealthmax/02_Baskets_and_Internals/2.9_CarbonBasketLidHandle_Spare.stl)

---

### Phase 2: Voron 2.4 (`350 × 350 mm`) — Print AFTER Tophat Encloses the Printer
All 7 oversized StealthMax ABS/ASA parts exceed the MK4S `250 × 210 mm` bed and **must** print on the enclosed Voron 2.4 (supports disabled, 100% scale):
- [ ] **`1×`** [`M_1b.0_FlowChamber_DOPM.stl`](../models/stealthmax/01_Chambers_and_Mount/M_1b.0_FlowChamber_DOPM.stl) (`286.4 × 286.4 mm`) — *keep 40 mm crosshairs intact as rear-panel drill guides.*
- [ ] **`1×`** [`M_2.0_MainChamber.stl`](../models/stealthmax/01_Chambers_and_Mount/M_2.0_MainChamber.stl) (`290.3 × 298.6 mm`)
- [ ] **`1×`** [`M_2.1_CarbonBasket.stl`](../models/stealthmax/02_Baskets_and_Internals/M_2.1_CarbonBasket.stl) (`279.3 × 279.3 mm`) — *upstream qty is **1×**, not 2×.*
- [ ] **`1×`** [`M_2.2a_CarbonBasketLid.stl`](../models/stealthmax/02_Baskets_and_Internals/M_2.2a_CarbonBasketLid.stl) (`267.9 × 271.4 mm`) — *or upstream-recommended `M_2.2b_CarbonBasketLid_noHandles.stl` + `2× 2.9` handles.*
- [ ] **`1×`** [`M_3.0_LidFrame.stl`](../models/stealthmax/03_Lid_and_Hardware/M_3.0_LidFrame.stl) (`300.3 × 300.3 mm`, 4–5 walls for gasket stiffness)
- [ ] **`1×`** [`M_3.1_LidPanelHolder.stl`](../models/stealthmax/03_Lid_and_Hardware/M_3.1_LidPanelHolder.stl) (`264.8 × 264.8 mm`)
- [ ] **`1×`** [`M_GhettoPanel.stl`](../models/stealthmax/03_Lid_and_Hardware/M_GhettoPanel.stl) (`250.0 × 250.0 mm` circular — exceeds MK4S 210 mm Y)

---

## 3. Critical BOM & Vacation Bench Audit

> [!IMPORTANT]
> Check these verified hardware details before packing for the trip:

1. **StealthMax v2 Hardware Status:**
   - **Owned / Likely Owned:** `Stealthmax PCB v2.0 Rev C`, activated charcoal, `8×3 mm` magnets, M3 inserts/screws, [`SM2.cfg`](../printer_data/config/SM2.cfg) stub, and `SGP40` + `BME280` sensors *(confirm sensors are in your parts bin before packing)*.
   - **Ordered (En Route):**
     - `1×` **12V** 4-pin PWM blower (**Delta `BFB0712HF`** 7530 — *note: 12V, powered by Isik's Tech PCB `24V → 12V` buck*)
     - `2×` **Xiaomi Mijia G1 HEPA** filters
     - `1 m` of **3 mm sponge cord** (`Shore A ≤ 0`)
   - *Panel Thickness Check:* `0b.1_DirectOnPanelMountScrew_DOPM.stl` was printed in the stock **3 mm** spec; verify rear panel + foam tape thickness when home in case `4/5/6 mm` UserMod nuts are needed.

2. **Dock Spring-Steel Wiper Fastener & Strip Width Corrections:**
   - **Fasteners are `M3`, NOT `M2`:** Both `DraftShift/ModularDock/BOM.md` and `Spring_Steel_Clamp.stl` (`3.40 mm` holes) specify **`2× M3 brass heat-set inserts` + `2× M3×8 BHCS`** per dock (`12` total for `6` docks — already in stock).
   - **Strip Width:** `Spring_Steel_Clamp.stl` has a **`7.00 mm`** wide slot (`BOM.md` assumes `7.2 × 34 mm` tape-measure blade). Ordered OFFO 304 stainless ties are **`4.6 mm × 0.25 mm`**—they work if centered carefully under the M3 clamp, or pack a `7 mm` tape-measure blade as backup.
   - **Already Owned:** `NeoPixel_Soldering_Jig.stl` does **not** need printing.

---

## 4. Home Integration & Remaining Machine Projects

### Step A: Finish `T2` (`cyan`) & Nudge (First Task Back at the Voron)
> [!CAUTION]
> **Safety Lockout:** `[tool T2]` is currently uncommented in [`T2.cfg`](../printer_data/config/toolchanger/tools/T2.cfg). **Do NOT run `T0`, `T1`, or `UNSELECT_TOOL` while `T2` is on the shuttle** until `T2`'s dock is mounted and `params_park_x/y/z` are calibrated.

- [ ] Assemble and install printed `T2` rear support brace and secure wires/umbilical.
- [ ] Assemble and mount printed `T2` DraftShift Modular Dock (`Top_Crossbar.stl` & `Bottom_Crossbar.stl` for DraftChange door extender clearance).
- [ ] Verify `[extruder2]` motor direction (`dir_pin`), gear ratio (`50:10` vs `50:8`), and 100 mm `rotation_distance` calibration.
- [ ] Run Input Shaper using `[adxl345 T2]` and update `params_input_shaper_*` in [`T2.cfg`](../printer_data/config/toolchanger/tools/T2.cfg).
- [ ] Calibrate `T2` `params_park_x/y/z` (`TOOL_ALIGN_START` / `TOOL_ALIGN_TEST`).
- [ ] *(Optional)* Install solid copper `M3×6` contact screws on Nudge probe, then re-run `SET_NUDGE_POSITION` with `T0` in [`toolchanger-config.cfg`](../printer_data/config/toolchanger/toolchanger-config.cfg).
- [ ] Calibrate `[tool_probe T2]` `z_offset` manually in [`T2.cfg`](../printer_data/config/toolchanger/tools/T2.cfg) *(never run `SAVE_CONFIG`)*, then run Nudge calibration for `T2` `gcode_x/y/z_offset`.

### Step B: Fanny Pack, Hexa Distro, & Umbilicals
- [ ] Check at home whether `[a]_ptfe_ramp.stl` / ECAS04 collet pieces still need printing alongside the umbilical plate ([`models/fannypack_ceb/`](../models/fannypack_ceb/fannypack_and_umbilical_plate_suite.zip)).
- [ ] Once **FYSETC Hexa Distro Fusion V1.3** arrives, swap/mount Fanny Pack (currently built with **BTT CEB v1.0**, unmounted) onto rear frame with fused 24V trunk + USB-C to CM5.
- [ ] Build ~740 mm universal umbilicals with `0.3 × 3 mm` flat spring-steel spines across `T0`–`T2` to eliminate `T1` OptoTap drag.
- [ ] Bolt cured spring-steel silicone nozzle stoppers onto `T0`–`T2` docks.

### Step C: Power, Cooling & Future Expansion
- **CM5 Cooling:** Passive heatsinks + `skirt1` (`PF6` `[temperature_fan]`) keep idle SoC at `52.35°C` (`58.4°C` max seen) with `skirt1` off at idle; optionally bump `target_temp` from `55°C` to `62°C` in [`printer.cfg`](../printer_data/config/printer.cfg#L234-L246) instead of buying an active CM5 cooler.
- **24V PSU (`LRS-350-24`):** Queued upgrade (timing TBD). While on stock `LRS-200-24`, keep parked tools at `0°C` cold standby, sequential preheat only, and `max_power: 0.80` on `T0`.
- **Toolheads `T3` (`yellow`) & `T4` (Vacation Build):** User expects to bring and build/solder **both `T3` and `T4`** on vacation and believes all required hardware is on hand (`T3` EBB36 already flashed in [`T3.cfg`](../printer_data/config/toolchanger/tools/T3.cfg); verify `T4` EBB36, hotends, OptoTaps, thermistors, 3010 fans, and Sherpa gear kits before packing). Once home: flash `T4` if needed, mount `T3`/`T4` crossbar docks, and run extruder/IS/Nudge calibrations. *(Strictly use Chirpy Dragon Burner v8 `Cowl_NoProbe.stl`, Sherpa Mini R2, DraftShift `DragonBurner.stl` backplate, and Crossbar Docks—never CMYK repo assets.)*
- **Front-Bed Nozzle Brush & USB Camera:** Mount front-edge brush (10-pack owned) + mount for ordered **3-In-1 USB Camera Module** once delivered.
- **Ongoing:** Gridfinity storage prints (PLA/PETG), filament spool photo/weigh-in, Prusa MK4S `0.6 mm` nozzle (wishlist), and incoming Voron 0.2 kit.
