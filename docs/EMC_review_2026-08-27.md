# EMC Pre-Compliance Review — motor_controller

- **Date:** 2026-08-27
- **Target standard:** FCC Part 15 Class B (market: US)
- **Overall EMC risk score:** 49 / 100 *(below 50 = significant risk)*
- **Findings:** 29 total — 7 error, 14 warning, 8 info
- **Source report:** `analysis/2026-08-27_2300/emc.json`
- **SPICE:** not run (no ngspice found) — PDN/filter checks are analytical only

Board is **4-layer**:

| Layer | Contents |
|-------|----------|
| L1 / F.Cu | signals + fragmented local GND/power pours |
| L2 / In1.Cu | **solid GROUND plane** |
| L3 / In2.Cu | **split power plane: +12V + +3V3 (no ground)** |
| L4 / B.Cu | large GND pour + ~20 signal segments |

Reference-plane rule of thumb: a trace's return current flows in the plane on the
*adjacent layer*, not in a pour on its own layer.
- F.Cu traces reference **L2 (solid GND)** — good.
- B.Cu traces reference **L3 (split power)** — bad.

---

## MUST FIX before fabrication

### M1 — Move B.Cu signal segments to F.Cu  *(rules GP-001, RP-001)*
`/SDA` (8 of ~15 segments on B.Cu), `/SCL` (2), `/PA14` (2), `/PB2` (8) are routed
on B.Cu, so their return path is the L3 +3V3/+12V split, not ground.
Reference-plane coverage measured: `/PA14` 56%, `/SCL` 70%, `/SDA` 74%, `/PB2` 87%.
- **Action:** route SCL, SDA, PA14, PB2 entirely on **F.Cu** (references L2 solid GND).
- If any net must stay on B.Cu: flood tight GND pour both sides + stitch to L2 with
  vias ≤ 5 mm spacing, and never cross the +3V3↔+12V boundary on L3.

### M2 — U1 (LM2596S-5) has no decoupling capacitor  *(rule DC-002)*
No capacitor within 10 mm of the buck regulator — the dominant noise source on the board.
- **Action:** add input bulk (~10 µF) + 100 nF X7R directly at U1 VIN, and minimise the
  hot loop: input cap → VIN → SW → inductor → diode/GND → back to input cap.

### M3 — Solid reference plane under `/OUT_1` and `/OUT_2`  *(rule GP-001)*
Both nets are 100% on F.Cu but only 71–76% covered — real voids in the **L2 GND plane**
beneath them (via antipads / plane pulled back near the motor-output / mounting area).
- **Action:** inspect L2 in that region; fill the GND plane so it is continuous under
  the full length of both motor-output nets.

### M4 — Ground stitching vias at every layer transition  *(rule RP-001)*
`/PB2` (4 transitions), `/PA14` (2), `/SDA` (2), `/SCL` (2) change layers with **no GND
via within 1 mm** of the signal via. (Largely resolved by M1; applies to any remaining
transition.)
- **Action:** place a GND stitching via immediately next to each signal via that changes layer.

### M5 — Fix U3 unconnected VSS / ground domains  *(rule GP-005)*
U3 pad 23 (VSS) appears on 4 separate nets — `GND` plus 3 `unconnected-(U3-VSS-Pad23)*`.
Reported as "4 ground domains".
- **Action:** tie all U3 VSS/EP pads to GND with their own vias into the L2 plane.

---

## SHOULD FIX  (strongly recommended, not a hard blocker)

### S1 — CAN bus reference gaps  *(rule GP-001, warning)*
`/CANH` 94% and `/CANL` 84% coverage over ~55 mm. CAN is an external-cable interface.
- **Action:** tighten the F.Cu routing so both run over continuous L2 GND for their full length.

### S2 — Crystal routing  *(rules CK-001, CK-003)*
`/OSC_IN`, `/OSC_OUT` 100% on outer layer; `/OSC_IN` passes within 9.6 mm of connector J4.
- **Action:** keep the crystal loop tiny, add a local GND guard/pour stitched to L2 under Y1,
  and route OSC_IN away from J4 (or add a guard trace between them).

### S3 — I/O connector filtering + ESD  *(rules IO-001, IO-002)*
J2 (bridge) and J5 (external) have no ferrite / CM choke / TVS within 25 mm.
- **Action:** if either carries an off-board cable, add a series ferrite (e.g. BLM18AG601SN1D,
  600 Ω @ 100 MHz) and TVS/ESD protection at the connector. If both are fully inside the
  enclosure with no external cable, this drops to LOW priority.

### S4 — L3 power-plane strategy  *(root cause of M1)*
L3 being a +12V/+3V3 split is why any B.Cu routing is problematic.
- **Action (optional redesign):** pour L3 mostly as GND with power only where needed, or
  carry +12V/+3V3 as wide pours on the outer layers. Makes B.Cu routing safe and improves PDN.

### S5 — Decoupling placement tidy-up  *(rules DC-001, DC-003)*
- U2 (AMS1117-3.3): nearest cap 6.5 mm — move within ~3 mm.
- C16: 3.8 mm from its via (~2.6 nH) — put a via directly on the cap pad.

---

## NICE TO HAVE  (low priority / good practice)

### N1 — Fragmented F.Cu ground pours
13 separate GND zones on F.Cu. Not harmful once L2 is solid, but stitch them to L2 with
vias every 5–10 mm so they don't float.

### N2 — Second ground via at TVS D4  *(rule ES-002)*
D4 has 1 GND via within 3 mm. A second via halves the ground inductance (~0.5 nH → ~0.25 nH).

### N3 — `/PA13` reference gap  *(GP-001, warning)*
86% coverage over 10 mm — minor; clean up if convenient while doing M1.

---

## INFORMATIONAL  (no action — use for lab prep)

- **EE-001 board cavity resonances:** (1,0) 767 MHz, (0,1) 1125 MHz, (1,1) 1362 MHz,
  (2,0) 1534 MHz, (2,1) 1903 MHz — ensure decoupling SRF coverage near these.
- **EE-002 / SW-001 switching envelope:** U1 at ~150 kHz, harmonics reach the 30–88 MHz
  band from ~harmonic 200; envelope knee at 32 MHz.
- **Pre-compliance test plan:**
  - Priority band **30–88 MHz** (MEDIUM): LM2596 harmonics + 8 MHz crystal 4th/5th (32/40 MHz).
  - Near-field probe points: L1 switching inductor (104.3, 53.9), Y1 crystal (123.6, 83.2),
    J2 (146.6, 105.2), J5 (146.6, 90.6).
  - Highest-risk interfaces: J2, J5 (unfiltered).
- **Coverage caveat:** radiated FCC Class B coverage from this tool is *minimal* — it catches
  design mistakes, it cannot predict dB levels or enclosure/cable effects. Only an accredited
  lab measurement confirms compliance.

---

## Priority summary

| ID | Item | Priority | Rule |
|----|------|----------|------|
| M1 | I2C/PA14/PB2 off B.Cu onto F.Cu | **MUST** | GP-001, RP-001 |
| M2 | Decoupling at U1 buck | **MUST** | DC-002 |
| M3 | Fill L2 GND void under OUT_1/OUT_2 | **MUST** | GP-001 |
| M4 | GND stitching via at layer transitions | **MUST** | RP-001 |
| M5 | U3 VSS to GND / ground domains | **MUST** | GP-005 |
| S1 | CAN bus over continuous GND | Should | GP-001 |
| S2 | Crystal guard + keep from J4 | Should | CK-001, CK-003 |
| S3 | Ferrite + TVS on J2 / J5 | Should (if external cable) | IO-001/002 |
| S4 | Rework L3 to GND-dominant plane | Should (redesign) | — |
| S5 | U2 / C16 decoupling placement | Should | DC-001, DC-003 |
| N1 | Stitch F.Cu GND pours to L2 | Nice to have | — |
| N2 | 2nd GND via at D4 | Nice to have | ES-002 |
| N3 | /PA13 reference gap | Nice to have | GP-001 |
