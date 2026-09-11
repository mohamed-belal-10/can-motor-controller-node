# CAN Motor Controller Node — PCB Layout Guidelines

**Companion document to `CAN_Motor_Controller_Node_Spec.md` — layout phase only.**
**Status:** Schematic capture complete (5/5 subsystem sheets, ERC clean). This document covers placement, routing, plane strategy, and DRC targets for the layout phase that follows.
**Scope:** This file is deliberately separate from the main spec. The spec owns *what* is on the board and *why*; this file owns *where it goes and how it's wired physically*. Keep them separate — don't let layout notes drift back into the spec or vice versa.

---

## 1. How to use this document

Read once before placement, then again before routing. The two passes catch different classes of mistake:
- **Before placement:** §3 (floorplan) and §4 (placement rules) — cheap to fix now, expensive after traces exist.
- **Before routing:** §2 (trace sizing, including via strategy in §2.4), §5 (plane strategy, including L1/L4 fill in §5.2), §6 (CAN specifics), §7 (thermal) — these shape how you draw each net.
- **Before ordering:** §8 (DRC/manufacturing) and §9 (pre-submission checklist).

---

## 2. Trace sizing

### 2.1 Power nets

| Net | Worst-case current | Min width (1oz Cu, 10°C rise, IPC-2221 external layer) | Recommended |
|---|---|---|---|
| VM / motor traces (12V → DRV8874 → motor) | 2.0–2.2A (I_TRIP-capped) | ~0.75–0.8mm (30–32mil) | **1.0–1.25mm (40–50mil), or a poured copper zone** |
| 12V input, pre-reverse-protection-diode | 2.1–2.2A | same as above | Pour from screw terminal through diode footprint to the VM split point |
| 5V intermediate rail (buck output → LDO input) | ~100–150mA | trivial by ampacity | 0.5mm (20mil) — sized for low impedance, not current capacity |
| 3.3V rail | ~100–150mA | trivial by ampacity | 0.5–0.6mm, or a poured zone. This rail is the ADC's indirect reference via VDDA — impedance matters more than ampacity here |
| I²C, GPIO, digital control | mA-level | n/a | 0.2mm (8mil) standard |
| CAN-H / CAN-L pair | mA-level | n/a | 0.2mm (8mil), matched pair — see §6 |

**Why VM/motor traces are oversized relative to bare ampacity:** at 2A, IPC-2221 only demands ~30mil. The extra margin isn't about heating — it's about keeping resistance (and therefore voltage sag during I_TRIP chopping events) low, since that sag is exactly the noise source the VM bulk cap is sized to absorb. A resistive trace partially undoes that capacitor's job.

**If board space forces narrower traces:** JLCPCB offers 2oz copper for a modest upcharge — halves the required width for the same current/temperature rise. Use this rather than shaving the VM/motor width below 30mil.

**Preferred approach for VM/motor current:** don't hand-route these as traces at all where avoidable. Pour a dedicated copper zone from the input connector, through the reverse-protection diode, to the DRV8874 VM pin and bulk cap. Narrow to a routed trace only where it must squeeze past another feature.

### 2.2 Why 5V doesn't get the same pour treatment as VM

The 5V intermediate rail (buck output → LDO input) is a trace, not a pour, and deliberately so — the factors that justified pouring VM don't apply here:

| Factor | VM/12V | 5V intermediate |
|---|---|---|
| Current | 2.0–2.2A, with I_TRIP transients | ~100–150mA steady, no chopping events |
| Physical run | Spans the board: input connector → diode → splits to buck input and DRV8874 VM | Short, local hop between the buck output cap and LDO input cap, which sit adjacent in the floorplan specifically to keep this run short |
| Noise sensitivity | Sag here couples directly into current-sense noise | Already accepted as noisy by design — that's the entire reason the LDO exists downstream. A lower-impedance trace doesn't protect anything the LDO isn't already cleaning up. |

**0.5mm (20mil)** trace is sufficient (already reflected in §2.1's sizing table), routed as a short, direct hop on whichever outer layer the two caps land on. If a layer change is unavoidable, 1–2 vias are enough — this is a single crossing, not a current-stitching job like §2.4. No third L3 zone is warranted for this net; adding one would mean another clearance boundary to route around for a net that never needed the impedance reduction a pour buys.

### 2.3 Signal nets

- Standard digital (GPIO, I²C, control lines): 0.2mm (8mil) — comfortably within JLCPCB standard tier.
- CAN differential pair: 0.2mm (8mil) traces, routed as a matched pair (see §6). Controlled impedance to a specific target (e.g. 120Ω differential) is not critical at 500kbps — matched routing and noise isolation matter more than a precise impedance number here.

### 2.4 Via strategy for VM/12V power nets

**Why vias at all, here:** VM/12V current is carried primarily as an **L3 copper pour**, not an outer-layer trace (see §5 for the plane split rationale). Every VM-carrying component pad on L1/L4 needs to reach that L3 pour through a cluster of vias — never a single via — dropping straight down from the pad into the pour. The pour itself does the job of "connecting everything together"; the vias are just the entry points into it.

**Two via classes cover the whole board:**

| Class | Drill / pad | Approx. current capacity per via* | Use for |
|---|---|---|---|
| **Standard power via** | 0.3mm drill / 0.6mm pad | ~0.8–1A (10°C rise, 1oz Cu) | Most VM/12V connections |
| **Thermal/high-density via** | 0.3mm drill / 0.6mm pad (same size, arranged as a grid) | same per-via, aggregated | DRV8874 thermal pad specifically |

*Via current-capacity figures vary by calculator and are conservative by design — treat them as a floor, not a tight budget. Don't go below 0.3mm drill for anything carrying real current: resistance and reliability both suffer below that, without meaningfully saving board area.

**Per-component via count:**

| Node | Worst-case current | Via count | Why this many |
|---|---|---|---|
| Input connector → VM net | ~2.2A | **4× 0.3mm** | Full board current passes through here first — most current-critical single point on the board |
| Reverse-protection diode, anode pad | ~2.2A | **3× 0.3mm** | Small SMA/DO-214 pad area — tightest current-density squeeze point on the board; watch this in the footprint |
| Reverse-protection diode, cathode pad | ~2.2A | **3× 0.3mm** | Same bottleneck, other side |
| DRV8874 VM pin | up to 2.0A (I_TRIP) | **4× 0.3mm** | High *dI/dt*, not just DC average — this is where current chops on/off every switching cycle, so low inductance matters as much as ampacity |
| VM bulk cap, positive terminal | transient/ripple, up to ~2A | **3–4× 0.3mm** | Absorbs energy during I_TRIP events — needs a low-impedance path into the plane |
| VM bulk cap, negative/ground terminal | same magnitude, return path | **3–4× 0.3mm → L2 ground** (not the VM plane) | Part of the star-point ground convergence (§5) — same current magnitude as the positive side, so same via count, but stitches to ground rather than VM |
| LM2596 input (Vin) pin | ~100–150mA (buck's own draw) | **2× 0.3mm** | Ampacity alone needs 1, but a single via is a single point of failure and an unnecessarily high-inductance connection — never leave any power node on exactly one via |
| DRV8874 thermal pad (PGND) | thermal + power-ground return | **6–9× 0.3mm**, grid across the pad | Doing double duty: heat dissipation *and* PGND stitching — check the datasheet, the exposed pad ties internally to power ground, so this grid is also the switching-current ground return, not just thermal relief |

**Two things happening at once, and they don't always agree:**
- **Ampacity** scales with current — why the input connector, diode, and DRV8874 VM pin get 3–4 vias while the LM2596 input gets only 2.
- **Inductance / dI-dt** matters independently of average current — why the DRV8874 VM pin gets as many vias as the diode despite lower average current. Its current is chopping at 20kHz; a low-inductance connection matters for switching noise even when the RMS number looks manageable.

**Zone fill / thermal relief:** when the L3 VM pour is drawn as a copper zone, check the pad-connection style for high-current pads. Default thermal-relief spokes (four thin connecting legs) are meant for hand-soldering ease and add resistance exactly where it's unwanted on this net. For the DRV8874 VM pin and the VM bulk cap especially, set the pad connection to **solid fill** rather than thermal relief — reflow assembly doesn't need the thermal isolation the way hand soldering does.

**KiCad practicalities:**
- Set up a dedicated net class (e.g. `VM_HIGH_CURRENT`) with the 0.3mm/0.6mm via and 40–50mil trace width as class defaults, rather than manually overriding each net.
- The DRV8874 thermal pad via grid is usually a manual placement in the footprint editor, not something the net class or autorouter handles sensibly on its own — place that one deliberately.
- After the L3 zones are filled, visually re-check that the VM/3.3V pour boundary doesn't cross under the CAN pair, crystal, or I²C bus (§4.1, §5) — easy to lose track of once looking at a filled zone instead of the planned boundary line.
- Crystal (OSC_IN/OSC_OUT) and load-cap traces: shortest possible, standard signal width, no other signal routed parallel or underneath.

---

## 3. Floorplan — physical zones

Map the five schematic sheets to physical board regions. The goal is to let the electrical "quiet" and "noisy" boundaries coincide with the geographic ones, so a single ground-plane split (if used on L3) can follow a sensible line.

```
┌─────────────────────────────────────────────┐
│  [PWR IN]      [BUCK STAGE]   [MCU CORE]     │
│  screw term →  LM2596,        STM32,         │
│  reverse-prot  inductor,      crystal,       │
│  diode → VM    diode →        NRST, BOOT0,   │
│  bulk cap      LDO input      SWD header     │
│                                                │
│  ───────────── quiet / noisy boundary ──────  │
│                                                │
│  [MOTOR DRIVE]                [CAN]           │
│  DRV8874, VREF divider        SN65HVD230,     │
│  + filter cap, motor          NUP2105L,       │
│  output connector,            CAN connectors, │
│  second VM bulk cap           termination     │
│                                jumper         │
│  ─────────────────────────────────────────    │
│                [SENSING]                       │
│                I²C pull-ups,                   │
│                JST-SH → AS5600 sub-PCB         │
└─────────────────────────────────────────────┘
```

**Rationale:**
- Power-in and buck stage anchor one edge — where the 12V cable physically enters anyway.
- MCU core sits as the quiet island, geographically buffered from both switching zones.
- Motor drive occupies its own corner as the single noisiest block on the board (highest dI/dt, highest current).
- CAN sits on the *opposite* side from motor drive — it has its own external cable and shouldn't share an edge with the H-bridge.
- Sensing/I²C is pushed as far from motor drive as the floorplan allows — no error correction on that bus, so it gets the most protective placement despite being electrically the lowest-power signal on the board.

**Note:** placement within a zone still needs a second check beyond the zone assignment itself — see §4.4 for the zone-adjacent vs. loop-adjacent distinction, using the AMS1117's placement relative to the buck stage as a worked example.

---

## 4. Placement rules

### 4.0 Proximity to the DRV8874 — proximity isn't a single rule

"Away from the DRV8874" is not a blanket distance requirement from the part as a physical object — several things are *forced* to sit right next to it and are fine there. The real driver is **signal character**, not raw distance:

- **Functionally close, and correctly so:** VCC decoupling cap, VREF filter cap, VM bulk cap, the thermal pad via grid, IN1/IN2 (from STM32), nFAULT and its pull-up, and the motor output traces themselves. These either exist specifically to catch noise at the source, or are digital/logic-level connections with enough noise margin (crossing ~1.65V to flip state on a 3.3V rail) that nearby switching noise doesn't threaten them. Pushing these away would only add inductance and trace length for no protective benefit — don't try to apply §4.1's distance logic to this group.
- **Genuinely needs distance:** anything analog, timing-critical, or without error correction — the crystal, the I²C bus, the CAN pair, and the *routing* of the IPROPI trace once it leaves the DRV8874 pin (the pin itself is unavoidably on the part; where the trace goes afterward is the part that matters). These have little to no noise margin, and that's what §4.1 is actually protecting.

Read §4.1 as "keep the low-margin signals away from the switching node and motor-current loop specifically," not "keep everything away from the DRV8874 except caps."

### 4.1 Keep apart — non-negotiable

| Pair | Why | Guidance |
|---|---|---|
| Crystal ↔ buck stage / DRV8874 | Crystal is the most sensitive net on the board; switching noise coupling here corrupts the CAN bit-timing reference | ≥10mm physical separation as a target on this board size. No switching trace routed near or under the crystal or its load caps. |
| I²C pair ↔ motor / switching traces | No error correction on I²C; coupled noise = silent bad reads | Don't route SDA/SCL parallel to or alongside motor output traces or the buck switching node. Cross perpendicular only, never parallel-adjacent. |
| IPROPI trace (DRV8874 pin → PA0) ↔ motor power traces | Small current riding through a noisy neighborhood the whole way to the ADC | Route on an inner layer where possible, referenced to solid L2 ground for the full run. Don't let it parallel the motor traces for any distance, even though R_IPROPI itself sits at the STM32 end. Full via-routing procedure and layer choice: §5.4. |
| CAN-H/CAN-L ↔ buck switching node / motor traces | Crosstalk into a differential pair from an adjacent single-ended noisy trace degrades signal integrity even at CAN's relatively low speed | No parallel runs near switching nets. |

### 4.2 Keep close — deliberately

- Every decoupling cap: <2mm from its pin where layout allows, dedicated via to ground plane at the cap's ground pad (not shared with another net's return).
- VREF filter cap (100nF): directly at the DRV8874 VREF pin. A long trace between the divider and the cap partially defeats the cap's purpose (catching switching noise before the trip comparator sees it).
- DRV8874 VCC decoupling cap: at the VCC pin, but *not* sharing a via/ground stub with the VM bulk cap's ground return — different current personalities (logic supply vs. motor power return) even though both land on the same plane.
- VM bulk cap: directly at the VM pin — trace/pour length between them reintroduces the inductance the cap exists to defeat.
- Thermal pad via grid: on the part itself, by necessity (§7.1).
- IN1/IN2 (DRV8874 control inputs, from STM32) and nFAULT + its pull-up: land wherever convenient near the DRV8874. These are digital, logic-level signals with real noise margin — not a noise risk in the way an analog or timing signal is, so there's no benefit to routing them the long way around to gain distance.
- Test points: reachable from one side of the board with a probe. Don't bury behind the motor connector or under the CAN header.

### 4.3 The tightest loop on the board

The LM2596 switching node — SW pin → inductor → catch diode — should be treated as its own small island. Inductor and diode placed immediately adjacent to the SW pin, loop area minimized above all else in this zone. This is the single highest-priority tight-loop for EMC on the entire board; everything else in §4 is secondary to getting this one right.

### 4.4 Zone-adjacent vs. loop-adjacent — two different questions about the same placement

These sound similar but answer different concerns, and a component can pass one while failing the other. Worth checking both separately for anything placed near the buck stage or the motor drive zone.

**Zone-adjacent** means two components sit in the same region of the floorplan (§3) — a coarse-grained placement decision based on functional/net relationship. The AMS1117 is zone-adjacent to the LM2596 because it's the next stage downstream in the power chain; putting it in the same neighborhood keeps the 5V trace between them short (§2.2). This is correct and intentional.

**Loop-adjacent** is a tighter, independent concern: is a component sitting physically close to — or worse, overlapping — a *specific* high-dI/dt current loop, regardless of which zone that loop lives in. The SW-inductor-diode loop (§4.3) radiates a real local magnetic field at 150kHz with fast switching edges; anything crowded directly against it can have noise coupled in through that field alone, independent of trace routing or ground-plane quality.

**Why both have to be checked separately:** being in the right zone doesn't protect a component from being too close to a specific noisy loop inside that zone. The AMS1117 is a working example — it should be zone-adjacent to the buck stage (short 5V trace, §2.2), but *not* loop-adjacent to the SW-inductor-diode loop specifically, since its whole job is delivering a clean rail to VDDA (the ADC reference IPROPI is measured against). Noise picked up here through proximity to the switching loop would leak into the current-sense reading through a different path than the one the two-stage power architecture was built to avoid (spec §4). Concretely: place the LDO and its caps on the *output* side of the switching loop, oriented so the loop isn't facing directly at them — a few mm and orientation is enough, not crystal-level separation (§4.1's ≥10mm is a different, larger concern).

---

## 5. Ground / power plane strategy

Stackup (already locked): **L1 signal, L2 solid ground, L3 split power (VM/3.3V), L4 signal.**

- **L3 will need a split** between the 3.3V zone and the VM/12V zone — same layer, different nets. Route that split to run under the geographic boundary between the MCU/sensing zone and the power/motor zone (§3), so the electrical and physical boundaries agree.
- **Do not let the L3 split pass underneath:** the CAN pair, the crystal, or the I²C bus. Any signal on L1/L4 that needs a clean return path will find a discontinuity if the plane directly beneath it is split — see §5.3 for why this matters more than it might seem.
- **Star-point convergence:** VM bulk cap ground, motor output connector ground, and DRV8874 ground should all converge at one point near the power input. That convergence point is also where the "quiet" ground (MCU, CAN, sensing) ties in — not scattered vias dropped wherever a component happens to land.
- **L1 is directly adjacent to L2.** A sensitive signal routed on L1 gets L2's solid, unsplit ground as its immediate reference — the ideal case.
- **L4 is directly adjacent to L3, not L2.** L3 sits physically between L2 and L4 in the stackup, so a signal on L4 references the *split* power plane, not the solid ground plane, unless the local L4 ground flood (§5.2) is doing the work instead. This is an easy mix-up to make when thinking "L1 or L4 both work the same" — they don't. Treat L1 as the preferred layer for anything genuinely sensitive; treat L4 as workable only where the local ground flood and stitching vias (§5.2) are actually present and unbroken along the signal's path.

### 5.1 L2 is solid — L3's split does not apply here

**L2 is one single, continuous, unsplit ground pour across the entire layer** — no slots, no boundaries, one GND net covering the whole plane. This is different from L3, which is deliberately split into separate VM and 3.3V zones. Don't confuse the two: the star-point convergence discussed above is about *where on the one continuous L2 plane* each net's return current enters (via placement discipline), not about cutting the ground copper itself.

**L3 zone-to-zone clearance:** the VM zone and 3.3V zone on L3 are different nets, so KiCad enforces a clearance gap between them and DRC will flag insufficient spacing. At this voltage difference (~9V), the electrical breakdown minimum is trivial (well under 0.1mm per IPC-2221) — the real driver is manufacturing margin, not electrical necessity. Use **0.5mm (20mil)** as the zone-to-zone clearance: comfortably above JLCPCB's manufacturing floor (0.15–0.2mm standard tier), with enough slack for via placement near the boundary and some tolerance if the boundary line has to shift slightly once routing is underway. Set this explicitly in both places KiCad reads it from — the zone properties clearance field for each zone, and the net class clearance for `VM_HIGH_CURRENT` — so there's no ambiguity between the two when reading the DRC report.

Where the gap physically runs matters more than its exact width: route it along the geographic boundary between the MCU/sensing zone and the power/motor zone (§3), and after the zones are filled — not just planned — visually confirm it doesn't pass underneath the CAN pair, the crystal, or the I²C bus.

**Other L2-specific items:**
- **Board-edge clearance:** pull the L2 pour back slightly from the physical board outline (~0.25–0.3mm) rather than running copper to the edge — standard fab practice, avoids copper exposure/burring at the cut edge.
- **Mounting holes:** tie the 4× M3 mounting holes to ground by default (gives a chassis-ground path if the board ever mounts to a metal frame), unless there's a specific reason to isolate one.
- **Thermal relief on L2:** same logic as §2.4 — fine for hand-solderable low-current pads, but the DRV8874 thermal pad and VM bulk cap ground connections should be solid-filled given the current involved.

### 5.2 Ground fill on L1 and L4

Neither L1 nor L4 needs a current-carrying power pour — VM lives on L3, and 5V/3.3V distribution stays as traces near their local zones (see §2.1, §2.4). But both outer layers should still get a **flood-filled GND pour** over unused copper area, for reasons separate from current-carrying:

- **Shielding**, especially over/around the motor-drive and buck-switching zones — a grounded copper layer above the noisiest components helps contain radiated emissions rather than letting them couple into whatever's routed nearby on the opposite outer layer.
- **Extra return-path lowering** for any L1/L4 signal referencing ground — effectively parallels L2 locally.
- **Thermal spreading** — modest but free, particularly useful near the LM2596 and around the DRV8874 footprint.

**Stitch the fill to L2 with vias every 5–10mm** where component placement allows, across both L1 and L4. This keeps the outer-layer ground fill genuinely tied to the solid L2 reference rather than floating as a series of disconnected islands, and it reinforces the return path for anything routed nearby.

This fill is safe to run right up next to — even over — the crystal, CAN pair, or I²C traces. The earlier "keep apart" rules (§4.1) are about noisy *signal or switching* copper running near those nets; a grounded fill near sensitive traces is protective, not harmful, and doesn't violate those rules.

**Component-local copper that isn't a full pour but isn't a minimum-width trace either:**
- LM2596 tab — thermal pour under/around the TO-263 tab (§7.2).
- DRV8874 exposed pad — copper on L1 by definition; a small surrounding apron before the via grid takes over is normal.
- Reverse-protection diode landing pads and the input connector pad — these are the tightest current-density squeeze points (§2.4). A small widened copper polygon immediately around those pads, before the via cluster hands current down to L3, reduces localized heating right at the bottleneck.

**Check for isolated copper islands after fill.** Dense routing can pinch off a patch of L1/L4 ground pour from its stitching vias, leaving an unreferenced floating conductor — worse than no pour at all. Enable KiCad's "isolated copper" DRC check and clear any hits before finalizing.

### 5.3 Why crossing an L3 zone boundary hurts a signal — and why the detour path is the real cost

**Never route a sensitive signal so its path crosses the L3 VM/3.3V zone boundary.** This is worth understanding rather than just following, because the mechanism shows up elsewhere in this document too (§4.1, §6).

Every signal trace forms a loop: current goes out through the trace and *must* return through something. With a solid reference plane directly beneath a trace, return current naturally flows in the plane, hugging the signal's path almost exactly — tightest coupling, lowest impedance, physics doing the work for you without any deliberate routing on your part.

**A gap in that plane breaks this.** If the trace crosses over the L3 zone boundary, the return current directly beneath it hits a dead end at the gap and has to detour — around the split, through a distant via, or through weak coupling to some other plane. That detour is the actual problem, and it's worse than it sounds for two compounding reasons:

1. **Loop area balloons.** Instead of a tight loop (trace + the copper immediately beneath it), the return path now traces a long way around the gap. Loop area is directly proportional to radiated emissions *and* to susceptibility to picked-up noise — a long detour path turns a well-behaved signal into a small antenna, in both directions at once.
2. **Impedance discontinuity right at the crossing.** The signal sees a localized bump in characteristic impedance exactly where its reference disappears — reflections and ringing concentrated at one specific point on the board, right where you can least afford it on a low-margin signal.

This is exactly why L2 (never split, §5.1) is the preferred reference whenever a layer choice exists, and why the CAN pair, crystal, and I²C bus are all called out separately (§4.1) as unable to tolerate crossing the L3 gap even though each has a different underlying reason for being sensitive.

### 5.4 Worked example: routing IPROPI off L1

The IPROPI net (DRV8874 pin → PA0, feeding R_IPROPI) is a good template for handling any signal that has to escape a noisy zone via a layer change:

1. **Via at the DRV8874 pin, immediately.** Minimal exposed trace on L1 before dropping down — the less distance this signal spends on the noisy layer, the less it picks up.
2. **Route the L4 leg through the L4 ground flood (§5.2)**, not over bare L3 power copper. The local flood plus its stitching vias down to L2 gives this leg a workable, if imperfect, ground reference.
3. **Explicitly verify this leg doesn't cross the L3 zone boundary** underneath it — per §5.3, that crossing would reintroduce exactly the long-detour, high-loop-area problem the layer change was meant to avoid. This is a visual check against the filled zones, not something to assume from the floorplan sketch alone.
4. **Via back up to L1** near the STM32/R_IPROPI side — two vias total for the whole path, no more. Each transition is itself a small discontinuity; this signal needs a clean drop-and-return, not extra hops.
5. **Keep the final stretch — R_IPROPI to PA0 — as short as possible on L1.** This is the one segment carrying a voltage rather than a current, and it's the genuinely sensitive part of the whole path.

---

## 6. CAN pair — routing specifics

At 500kbps, precise controlled-impedance calculation is not critical the way it would be for a gigabit link. Do the following regardless — all free, all worth it:

- Route CAN-H and CAN-L as a matched pair: same layer, tightly coupled (spacing close to trace width), length-matched to within 1–2mm.
- No stubs off the pair between transceiver and connector.
- Continuous ground reference (L2) under the pair for its entire length — see §5 on avoiding the L3 split crossing underneath.
- Maintain physical order: NUP2105L between the connector and the transceiver, not the other way around — don't let routing convenience put the transceiver closer to the connector than the TVS array.

---

## 7. Thermal

### 7.1 DRV8874 (HTSSOP-16, exposed thermal pad)

- Stitch the exposed pad with a **grid of vias** — not one or two. Target 4–9 vias at 0.3mm drill spread across the pad area, landing directly on the L2 ground pour, which doubles as the heatsink.
- Confirm JLCPCB's via-in-pad handling (tented/filled vs. standard) for this pad before finalizing. Unfilled vias in a thermal pad can wick solder paste during reflow — a real assembly defect risk at this pad size, not a theoretical one.

### 7.2 LM2596 (TO-263)

- Generous copper pour on the tab side, even though thermal stress is lower here than at the DRV8874 given the current levels involved. Cheap insurance, no design cost.

---

## 8. Manufacturing / DRC targets (JLCPCB)

Design to these so DRC doesn't surprise you at the end. Confirm current values against JLCPCB's live capability page for your exact chosen stackup before final submission — these shift slightly by tier and revision.

| Parameter | Advanced tier | Design target (comfortable standard-tier margin) |
|---|---|---|
| Min trace/space | 0.127mm/0.127mm (5/5mil) | 0.15mm (6mil) |
| Min via drill | 0.3mm | 0.3mm, larger where thermal/current demands it |
| Annular ring | per drill size | Standard JLCPCB minimums |
| Silkscreen on pads | avoid | Keep silkscreen clear of all pads |

---

## 9. Pre-routing checklist

Run this after placement, before you start drawing traces. Cheaper to fix here than after routing exists.

- [ ] Crystal ≥10mm from LM2596 and DRV8874, no switching trace routed near/under it
- [ ] I²C pair placement doesn't parallel any motor or switching trace
- [ ] R_IPROPI still at the STM32 end (per spec), and the IPROPI trace's planned route doesn't parallel motor traces
- [ ] IPROPI (and any other L1/L4 layer-swapped signal) verified not to cross the L3 zone boundary anywhere along its path (§5.3, §5.4)
- [ ] CAN pair placement doesn't parallel buck switching node or motor traces
- [ ] LM2596 SW–inductor–diode loop is as tight as physically possible
- [ ] AMS1117 (and any other zone-adjacent component near a switching loop) checked for loop-adjacency separately from zone placement (§4.4)
- [ ] VREF filter cap placement is directly at the DRV8874 VREF pin, not downstream of the divider
- [ ] DRV8874 VCC decoupling cap ground via is separate from the VM bulk cap ground return
- [ ] Star-point ground convergence identified as a single physical location near power input
- [ ] L3 plane split path sketched, confirmed not to cross under CAN pair, crystal, or I²C bus
- [ ] Test points reachable without obstruction from the motor connector or CAN header
- [ ] DRV8874 thermal pad via grid planned (4–9 vias, 0.3mm), JLCPCB via-in-pad handling confirmed
- [ ] VM/motor current path sized as a pour where possible, ≥40mil where routed as a trace
- [ ] Mounting holes (4× M3, corners) clear of copper/components per JLCPCB keep-out
- [ ] L3 VM/3.3V zone clearance set to 0.5mm in both zone properties and net class (§5.1)
- [ ] L1 and L4 flood-filled with GND, stitched to L2 every 5–10mm, checked for isolated copper islands (§5.2)

---

## 10. Cross-reference

This document assumes the following are already fixed by `CAN_Motor_Controller_Node_Spec.md` and does not re-derive them:
- Component values and part numbers (§3 of the spec)
- Decoupling cap counts and values (STM32, DRV8874, SN65HVD230, AS5600)
- 4-layer stackup decision and layer assignment
- Ground-return-path philosophy (§4 of the spec) — this document operationalizes it into concrete plane/routing rules
- Pin assignments (§5.2 of the spec)

If any of those change, revisit §3–§6 of this document for knock-on effects, particularly floorplan (§3) and plane split routing (§5).

---

*Last updated: alongside schematic capture completion — layout phase kickoff.*
