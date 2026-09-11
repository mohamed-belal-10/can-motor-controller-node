# CAN-Connected Closed-Loop DC Motor Controller Node

**A portfolio project for Mohamed Belal ("Belo") — Mechatronics Engineering, GUC**
**Status:** **Every electrical design decision and component value closed**, across four audit passes. **Schematic capture in progress in KiCad — all five subsystem sheets drawn (Power Stage, MCU Core, Motor Drive, CAN Bus, Sensing).** **One item flagged but deliberately unresolved:** DRV8874 power-up sequencing (VCC vs. VM order) — check during schematic capture. **Three items remain deliberately deferred:** DRV8874 sourcing, magnet sourcing (both external, your action), and the AS5600 bracket finalization (§9, paused until the sub-PCB layout exists). **One item open before ERC/layout:** cross-check that the main board's AS5600 link connector and the sub-PCB's `J1` connector share the same pin order (§3.4) — both were placed independently.
**Prerequisite:** ESP-Drone board ordered and fabricated first

---

## 1. Project Description

This is a single PCB — one "smart joint" of the kind used in robot arms, rovers, or any multi-actuator robotic system. It bolts onto a DC motor, closes a cascaded position/speed/current control loop locally, and takes setpoints over a CAN bus instead of running dedicated wires back to a central controller. Two or more of these boards can share one twisted-pair CAN bus, each addressed independently.

**What it does, end to end:**
1. A command arrives over CAN — *"go to position X"* or *"hold speed Y."*
2. The onboard STM32 runs three nested PID loops (current → speed → position) to get the motor there safely and smoothly.
3. The board reports back over CAN: current position, speed, motor current, fault flags.

**Why this project, specifically:**
- It's the first board with a **real power stage** — current in the amps range, not the milliamps of ESP-Drone's digital signals. This is the actual PCB skill gap in your record: Kelvin sensing, thermal copper, return-path separation under switching noise.
- It introduces **CAN bus**, currently a zero on your skill matrix and the exact protocol your DLR-thesis framing is built around.
- It introduces **cascaded multi-loop control**, the conceptual bridge between the single-loop PID you already proved on maglev and the state-space work further down your priority list.
- It's a legitimate, honest answer to "tell me about your robotics work" — an actuator control node, not an inflated claim.

**Reference designs this is built from:** CVRA's `motor-control-board` (KiCad, CC-BY, near-identical spec) and Cormack's STM32L433 motor driver (single-board layout reasoning, undervoltage-lockout-as-safety-feature trick). Read both `.kicad_pcb` files directly before drawing your own schematic, the way you did with ESP-Drone.

---

## 2. System Architecture

```
                    ┌──────────────┐      ┌──────────────┐
   12V IN  ────────▶│ LM2596S-5V   │─5V──▶│ AMS1117-3.3  │────▶ 3.3V rail
   (motor            │ buck (switch)│      │ LDO (linear) │      │
    supply)          └──────────────┘      └──────────────┘      │
        │                                                          │
        │ (direct, unregulated)                                    │
        ▼                                                          ▼
   ┌──────────┐    PWM x2      ┌──────────┐   IPROPI    ┌─────────────┐
   │ DRV8874  │◀───────────────│ STM32F103│◀────────────│ (internal to │
   │ H-bridge │                │  C8T6    │             │  DRV8874)    │
   └────┬─────┘                └────┬─────┘             └─────────────┘
        │ motor terminals            │  I²C
        ▼                            ▼
   ┌──────────┐                ┌──────────┐        ┌──────────────┐
   │ DC MOTOR │────coupled────▶│  AS5600  │        │ SN65HVD230   │◀──▶ CAN H/L
   │          │  (shaft end)   │ (angle)  │        │ (transceiver)│    to bus
   └──────────┘                └──────────┘        └──────────────┘
```

---

## 3. Final Component List

### 3.1 Core compute & communication

| Component | Role | Key specs | Package | Local availability |
|---|---|---|---|---|
| **STM32F103C8T6** | Main MCU. Runs all three PID loops, reads the encoder over I²C, drives PWM to the H-bridge, runs the CAN peripheral (bxCAN). | 72 MHz, bxCAN built in, 5V-tolerant GPIO on most pins | LQFP-48 | ✅ Local (Future Electronics Egypt / UGE) |
| **8MHz crystal (HSE) + load caps** | External oscillator for the STM32 — feeds the internal PLL to reach the full 72MHz core clock. **Required for reliable CAN bit timing** at 500kbps; the internal RC oscillator (HSI) has looser tolerance and drift that isn't recommended for CAN. This is genuinely new territory for this board — previous boards (maglev) used the Blue Pill module, which has this circuit built in; this board uses the bare LQFP-48 chip, so the oscillator circuit has to be designed here for the first time. **Sourced: 3225 SMD quartz 4-pin crystal, 8MHz, `C_L=20pF` (UGE Electronics Egypt).** Load caps: **C1=C2=20pF**, matched directly to the crystal's own specified load capacitance. No external feedback resistor needed — internal to the STM32. Connects to the dedicated `OSC_IN`/`OSC_OUT` pins, not the general GPIO pool. | 8MHz, C_L=20pF, ±10ppm tolerance | 3225-SMD (crystal) + 0402/0603 (caps) | ✅ Local (UGE Electronics Egypt) |
| **Reset circuit (`NRST`)** | Manual/clean reset for the STM32. **No external pull-up needed** — the chip has one built in (~40kΩ, per datasheet). Add a **100nF capacitor** `NRST`→GND to filter noise and stretch the reset pulse on power-up, plus a **momentary push-button** `NRST`→GND for manual reset during bring-up/debugging without power-cycling the board. | 100nF cap + momentary pushbutton | Passive + button | ✅ Local, trivially common parts |
| **`BOOT0` strapping** | Selects boot source: low=main flash, high=system bootloader. **Tied to GND via a fixed 10kΩ pull-down** — no jumper. Since programming is exclusively via ST-Link/SWD (same as ESP-Drone and maglev), the system bootloader is never needed, so a fixed tie-down is simpler and more reliable than a selectable jumper. | 10 kΩ | Passive, 0603/0805 | ✅ Local |
| **STM32 decoupling capacitors** | Standard MCU support that was missing from the BOM entirely. **Precise breakdown for this LQFP-48 package: 3 digital `VDD` pins → 3× 100nF ceramic (one per pin, placed as close to each pin as layout allows), plus 1× 4.7µF bulk cap on the `VDD` rail. Separately, `VDDA`/`VSSA` (the analog supply feeding the ADC) gets its own 100nF, ideally isolated from digital `VDD` by a small series ferrite bead.** `VBAT` ties directly to `VDD` (no RTC backup battery in this design). This isn't a generic nice-to-have on this board specifically: `VDDA` is the literal reference voltage the ADC measures `IPROPI` against, so noise leaking onto it from the digital rail *is* current-measurement error, not just abstract EMC hygiene — it connects directly to the entire reason this board has a two-stage `LM2596→AMS1117` power architecture in the first place. All of these caps benefit from via-ing straight down to the solid ground plane (L2 of the 4-layer stackup already decided) right under each pad. | 3× 100nF (digital VDD) + 1× 4.7µF bulk + 1× 100nF (VDDA) + ferrite bead | Passive | ✅ Local |
| **Power-on LED + resistor** | Genuinely missing until this audit: no visual "is this board alive" indicator existed anywhere. Placed on the 3.3V rail — the first, cheapest signal you get the moment power is applied, before any probe or debugger is connected. Standard practice, especially valuable across the multi-stage bring-up sequence in §7. | Standard LED + series resistor sized for ~2-5mA at 3.3V (e.g. 1kΩ for a typical red LED) | Passive + LED | ✅ Local |
| **SN65HVD230** | CAN transceiver. Converts the MCU's digital CAN TX/RX signals into the differential CAN-H/CAN-L signaling that travels on the bus, and back. Standard **100nF local decoupling cap** at the VCC pin (not previously listed). **`RS` (pin 8) tied directly to GND** — selects high-speed mode (datasheet: strong pull-down to GND = high-speed, no slew-rate limiting). Standby/sleep mode is never used on this board, so a fixed tie-down is correct rather than routing RS to a GPIO. **`Vref` (pin 5) left floating (N/C)** — only meaningful with split termination (tying to the split's center point); this board uses standard single-resistor termination, and the datasheet explicitly permits leaving Vref floating when unused. | **3.3V supply only** — do not power from 5V | SOIC-8 | ✅ Local (UGE Electronics Egypt) |
| **120 Ω termination resistor** | Placed across CAN-H/CAN-L at each physical end of the bus cable, jumper-selectable. Prevents signal reflections that corrupt CAN frames. **Decided: 2-pin, 2.54mm pin header + removable jumper shunt cap**, in series with the resistor. Cap installed = termination enabled; cap removed = disabled. Chosen over a solder jumper specifically because the board's dual JST-XH IN/OUT connectors are designed for future daisy-chaining — if a third node is ever added mid-chain, this board's termination may need to be toggled off without breaking out a soldering iron. Matches how CVRA's reference board (this project's basis) handles it. | Standard 0805/1206 resistor + 2-pin 2.54mm header + shunt cap | Passive | ✅ Local (generic passive, any store) |

### 3.2 Power stage & motor drive

| Component | Role | Key specs | Package | Local availability |
|---|---|---|---|---|
| **DRV8874** | H-bridge motor driver. Takes PWM logic signals from the MCU and switches full motor current (volts/amps) to the motor. **Has a built-in current-sense pin (IPROPI)** — outputs a current proportional to actual motor current, set by one external resistor (`R_IPROPI`). No separate shunt resistor or current-sense amplifier IC needed. **Two layout requirements not previously noted:** standard **100nF local decoupling** at the `VCC` (logic supply) pin; and the HTSSOP-16 package's **exposed thermal pad needs via-stitching to a copper pour** for heat dissipation, genuinely relevant here given the current-limiting events this board is designed to handle regularly (`I_TRIP` chopping during stalls). | Up to ~6.5 A continuous, 4.5–37 V motor supply, integrated current sense, integrated current regulation/chopping | HTSSOP-16 (PowerPAD) | ❌ **Not stocked locally.** You've confirmed you'll source it internationally regardless (Digikey/Mouser/RS/TME, or a Pololu carrier board as an import fallback). Lead time and shipping cost apply — order early. |
| **R_IPROPI** | Sets the current-sense gain (converts DRV8874's IPROPI output current into a readable voltage for the ADC). **Calculated: 2.2 kΩ** — sized so the STM32's 3.3V ADC reads full-scale at ~3.3A (headroom above the JGA25-370's 1.2A stall current), giving stall current a clean ~36% of ADC range. `A_IPROPI` = 455 µA/A per DRV8874 datasheet. Output routes to **`PA0` (ADC1_IN0)** — see §5.2 pin tracker. | 2.2 kΩ, precision resistor recommended (1% tolerance) | Passive, 0603/0805 | ✅ Local |
| **VREF divider** | Sets the DRV8874's hardware current-chopping trip point (`I_TRIP`), independent of firmware — a physical safety backstop. **Calculated: R1 = 3.3 kΩ (3.3V→node), R2 = 5.1 kΩ (node→GND)**, giving `VREF = 2.004V` → `I_TRIP ≈ 2.0A`. Add a **100nF filter cap** from the VREF node to ground — the divider alone will pick up H-bridge switching noise, and since VREF sets a hard trip threshold, that noise can cause false trips. | R1 = 3.3kΩ, R2 = 5.1kΩ, C = 100nF | Passive | ✅ Local |
| **LM2596S-5V** | Buck converter, bare IC, fixed 5V output. Steps 12V motor supply down to a regulated 5V intermediate rail. **This is your one real switching-regulator layout exercise** — inductor selection, switching-node layout, feedback network — with no output-voltage calculation needed since it's a fixed-voltage part. | 3A continuous, 150 kHz switching, TO-263 | TO-263 (SMD) | ✅ Local (UGE Electronics Egypt) |
| **AMS1117-3.3** | Linear regulator (LDO), fixed 3.3V output. Takes the 5V rail down to the clean 3.3V logic rail powering the STM32, SN65HVD230, and AS5600. Chosen specifically to sit *after* the switcher: an LDO rejects the switching noise the LM2596 leaves on its output, giving the MCU's ADC a cleaner reference — which matters because the ADC is also reading the DRV8874's IPROPI current-sense signal. Noise on the supply becomes noise in the current measurement. **Output cap: 22µF electrolytic, 25V rating** — confirmed sourced (Flux Electronix Egypt, 0.75 EGP). Electrolytic works as well as tantalum here: its higher ESR naturally sits within the AMS1117's required 0.3–22Ω stability window, the opposite problem ceramic has. **Note: through-hole part** — since the board's assembly plan is JLCPCB SMT, this one component will likely need hand-soldering after assembly rather than going through the automated run, unless swapped for an SMD electrolytic. **Input cap: 10µF tantalum/electrolytic.** | 1A output, dropout ~1.1V, fixed 3.3V | SOT-223 | ✅ Local / near-universal — stocked everywhere, including as a standalone part at UGE |
| **Inductor** (buck stage) | Energy-storage element for the switching regulator. | **68 µH — confirmed part: `CD54-680M` (DMBJ / Dongguan Zhenbaojia Electronics), 6.1×5.5×4.85mm, in stock at 11.50 EGP. Datasheet-confirmed: DCR max 0.46Ω, IDC max 0.61A** (defined as the current causing either 25% inductance drop or 40°C temperature rise, whichever is smaller — a thermally-soft limit, not a hard cliff edge). Checked against this rail's realistic load: STM32 (~35–50mA active) + SN65HVD230 (~10–70mA) + AS5600 (a few mA) totals roughly 100–150mA steady-state, giving **~4–6× headroom** under the 0.61A rating — comfortable margin including power-up inrush. DCR loss at this current is trivial (~10mW). **No further verification needed — this is a fully specified, sourced, in-stock part.** | SMD power inductor | ✅ Local, confirmed in stock, datasheet-verified |
| **Schottky catch diode** (buck stage) | Provides the current path when the LM2596's internal switch is off. | 3A, low forward-voltage Schottky (e.g. SS34-class) | SMA/DO-214 | ✅ Local (generic, widely stocked) |
| **Reverse-polarity protection diode** | Series diode in the +12V input line, before it splits to DRV8874 `VM` and the `LM2596S-5V` input. Blocks all current if the supply is ever connected backward — a real bench mistake this protects against, cheaply. **Reuses the same 3A SS34-class Schottky already specified for the buck stage's catch diode** — one fewer distinct BOM part rather than sourcing a second diode type. Forward drop (~0.4V) is inconsequential here: the motor tolerates 6–18V and the buck stage has ample input headroom either way. Power dissipation at `I_TRIP` (2A): ~0.8W momentarily during a stall — well within the diode's 3A rating with real margin. A P-channel MOSFET "ideal diode" would lose less voltage/power, but adds gate-biasing complexity not worth it at these current levels for a first board. | Same part as buck catch diode — 3A SS34-class Schottky | SMA/DO-214 | ✅ Local (same sourcing as buck diode) |
| **`nFAULT` pull-up resistor** | Required for the DRV8874's open-drain `nFAULT` output to read as a valid logic high when no fault is present. | 10 kΩ | Passive, 0603/0805 | ✅ Local |
| **VM bulk capacitor + local decoupling** | Absorbs energy kicked back from the motor winding's inductance during switching/current-limit events, preventing voltage sag/spikes on the 12V rail that would otherwise show up as noise in the current-sense reading. **Sized via energy estimate: `E=½LI²` at I_TRIP=2.0A with an assumed ~2mH motor winding inductance (not datasheet-confirmed — this motor doesn't publish it) gives ~264µF for a 10% rail-voltage-rise tolerance.** Rounded to a standard value: **220–330µF electrolytic, 25V rated**, placed close to the DRV8874's VM pin, plus a **0.1–1µF ceramic** alongside it for the fast edges the electrolytic can't respond to in time. | 220–330µF/25V electrolytic + 0.1–1µF ceramic | Mixed | ✅ Local (basic, common values — same store tier as the AMS1117 cap) |
| **Bulk/decoupling capacitors** | Input/output filtering for the buck stage, input/output caps for the LDO (per AMS1117 datasheet — it needs a minimum output cap for stability), plus local decoupling at every IC's supply pin. | Electrolytic (bulk) + ceramic (decoupling), values per datasheet | Mixed | ✅ Local |

### 3.3 Sensing

| Component | Role | Key specs | Package | Local availability |
|---|---|---|---|---|
| **AS5600** | Magnetic rotary position sensor. Reads a small diametric magnet mounted on the motor shaft end and reports absolute shaft angle over I²C. Chosen over a mechanical 600 PPR encoder because it's **absolute** (always knows true position immediately, even after a reset or glitch) rather than incremental (loses position on any missed pulse or power interruption). **Implementation decided: bare SOP-8 IC on its own small dedicated sub-PCB (not a generic breakout module).** Most off-the-shelf breakout modules include their own onboard I²C pull-ups — combined with the main board's already-calculated 4.7kΩ pull-ups, the parallel result (~2.35kΩ) would quietly undo that calculation. Bare IC + own sub-PCB keeps the main board's pull-ups as the only ones on the bus, and matches how CVRA/Cormack's reference boards do it. Connects to the main board via a **JST-SH 1.0mm 4-position connector** (see §3.4 for the part) rather than a vague loose cable. Consider panelizing this small sub-PCB alongside the main board in the same JLCPCB order. **Power pin wiring — real datasheet requirement, previously missed: for 3.3V single-supply operation (this board's case), `VDD5V` and `VDD3V3` pins must be tied together and both fed from the 3.3V rail directly.** Getting this wrong (leaving `VDD3V3` floating, or not tying the two pins) is a real, easy schematic mistake for this specific chip. **Decoupling — corrected during schematic capture: `C1 = 100nF` + `C2 = 10µF`, both in parallel across the tied `VDD5V`/`VDD3V3` node to `GND`.** This matches the datasheet's own 3.3V-operation diagram (Fig. 13) exactly, not the separate 5V-mode "1µF LDO cap" table (Fig. 39) that doesn't apply once the internal LDO is bypassed by tying the two supply pins together. The 10µF value is specifically required for OTP burn procedures at 3.3V — kept in the design even though OTP burning isn't planned, since it's a negligible cost to leave in and guarantees the option stays open. **Bare IC confirmed local at UGE Electronics Egypt — but listed at 7,700 EGP, dramatically higher than typical pricing for this chip (a few dollars via LCSC).** Worth confirming actual unit quantity/price directly with UGE before ordering — this may be a bulk-reel listing or pricing error rather than a genuine per-unit cost. Your established LCSC + `easyeda2kicad` pipeline is very likely the cheaper route for this specific part even with international shipping. | 12-bit (4096 counts/rev), I²C interface, 3.3–5V supply | SOP-8 (bare IC, on its own sub-PCB) | ⚠️ Confirmed local but price unverified — compare against LCSC before committing |
| **Diametric magnet** | Mounted flush on the motor shaft end, read by the AS5600 through the PCB. **Mounting method decided: glued flat onto the shaft stub's end face**, centered on the rotation axis — not fitted around the shaft diameter via a collar. This sidesteps a real gap: the JGA25-370's rear shaft stub dimensions aren't published anywhere (checked the manufacturer manual and multiple listings — only the front output shaft is dimensioned). Gluing flat only requires the shaft end to be reasonably flat and accessible, true for essentially any small DC motor, and doesn't depend on knowing the shaft diameter at all. | 6mm diameter × 2.5mm–3mm thick, diametrically magnetized, N35–N42 grade | — | ❌ **Not confirmed local.** A cheap local candidate (Electra Store-style CNC/3D-printer supplier, 6×3mm, 8 EGP) was found, but the listing only says "neodymium disc magnet" — doesn't state diametric vs. axial magnetization, and the product photo shows the magnets stacking flat-face-to-flat-face, the classic behavior of **axial** magnets (poles on the flat faces). Diametric magnets (poles split across the round edge) don't stack that way. Axial magnets will not work with the AS5600 — they don't produce the rotating field the sensor needs. **Must confirm "diametrically magnetized" explicitly with the seller before buying**, or default to a specialty supplier / AliExpress listing that states it outright. |
| **I²C pull-up resistors** | Required on SDA/SCL lines for any I²C bus to function — open-drain signaling needs external pull-ups. **Calculated: 4.7kΩ, confirmed against the I²C-bus spec formula, not just assumed.** With `V_DD=3.3V`, `I_OL(max)=3mA`: `Rp(min) = (3.3−0.4)/0.003 ≈ 966Ω`. With an estimated bus capacitance of ~50pF (short on-board trace, two devices) at Standard-mode 100kHz (plenty fast — the AS5600 is only read once per ~10ms position-loop tick): `Rp(max) = 1000ns/(0.8473×50pF) ≈ 23.6kΩ`. **4.7kΩ sits comfortably mid-range** between 966Ω and 23.6kΩ — well-margined on both rise time and static current, not just a default that happens to work. Both resistors pull to **3.3V** (matching AS5600/STM32 logic level, not 5V). Placed only on the main board — deliberately **not** duplicated on the AS5600 sub-PCB, to avoid a second parallel pull-up quietly shifting the calculated value. | 4.7 kΩ ×2 | Passive | ✅ Local |
| **`JP1` — DIR configuration jumper** | Sets the AS5600's counting direction (`DIR` pin: GND = clockwise-increasing, VDD = counterclockwise-increasing). **Decided during schematic capture: 3-pad SMD solder jumper**, not a fixed tie. Center pad → `DIR`, one outer pad → `+3V3`, other outer pad → `GND`. Defers the CW/CCW decision from schematic time to assembly time — bridge whichever side matches the physical mounting once the motor and bracket are in hand, rather than guessing polarity now and risking a respin. | 3-pad solder jumper | Passive | ✅ Local (generic) |
| **AS5600 `OUT`/`PGO` pins** | **`OUT` (Pin 3): No-Connect.** Angle is read digitally over I²C (`ANGLE` register); the analog/PWM output isn't used. **`PGO` (Pin 5): No-Connect**, relying on its internal pull-up. Programming is done via I²C (datasheet Option A), so the OUT-pin programming path (Option B, which requires `PGO` tied to GND) doesn't apply. | — | — | — |

### 3.4 Connectors

| Component | Role |
|---|---|
| **SWD header (4-pin)** | Programming/debugging the STM32 via ST-Link. |
| **Power input connector** | Brings in 12V (or your chosen motor supply voltage). **Decided: 2-pin, 5.08mm pitch screw terminal block** (UGE Electronics Egypt or Micro Ohm's pluggable M+F version, ~6 EGP). Rated 10–16A depending on exact manufacturer — real-world worst-case current here is ~2.1–2.2A (motor's I_TRIP-limited draw plus buck stage overhead), giving **~5–7× headroom**, not a tight fit. Same part reused for the motor output connector below — one fewer distinct BOM entry. |
| **CAN bus connector** | **Decided: JST-XH, 2.5mm pitch, 3-position (CAN-H, CAN-L, GND). Two identical connectors per board ("IN" and "OUT"), both wired to the same three nets internally**, enabling clean node-to-node daisy-chaining rather than star wiring or splicing. Only the two physical end-of-bus boards need their `120Ω` termination jumper enabled. GND included (not just the two CAN signal lines) since each board has its own independent power supply — there's no other shared ground reference between nodes otherwise. Standard, locking, polarized, widely available. **ESD protection: `NUP2105L` dual-channel TVS diode array, placed at the connector before the `SN65HVD230`.** Industry-standard part built specifically for CAN bus protection — IEC 61000-4-2 ESD rated to 30kV, automotive-grade (AEC-Q101), SOT-23 package, low enough capacitance (30pF) to not distort CAN signal integrity. **Confirmed local: UGE Electronics Egypt.** Warranted here specifically because this connector is externally exposed — cables plugged/unplugged between boards. |
| **Motor output connector** | **Decided: same 2-pin, 5.08mm pitch screw terminal block as the power input connector** — reused part, not a separate one. Motor current is hard-capped at `I_TRIP` (2.0A) by the DRV8874's hardware chopping, never higher in sustained operation — well within the connector's 10–16A rating. Easy to service for a first board (loosen/tighten screws, or unplug entirely if using the pluggable variant). **ESD/transient protection: generic bidirectional TVS diode across the motor terminals**, rated above the 12V rail. Lower priority than the CAN connector — the DRV8874 already handles switching-induced transients internally and the VM bulk cap already absorbs winding energy — but this connector is still externally touchable, so a basic external-event backstop is worth the trivial part cost. |
| **AS5600 sub-PCB link connector** | Connects the small AS5600 sub-PCB (§3.3) to the main board — carries VCC/GND/SDA/SCL. **Decided: JST-SH, 1.0mm pitch, 4-position**, on both the main board and sub-PCB sides. Standard, small, low-profile connector for exactly this kind of low-pin-count sub-board link. Closes an earlier vagueness — previously just described as "a short 4-wire cable" with no actual connector specified. **Pinout, sub-PCB side (`J1`): Pin 1 = `+3V3`, Pin 2 = `I2C1_SCL`, Pin 3 = `I2C1_SDA`, Pin 4 = `GND`.** ⚠️ **Not yet cross-checked against the main board's matching connector** — JST-SH is keyed against reversal, but not against a pin-order mismatch between two independently-placed footprints. Verify both sides agree pin-for-pin before ordering. **Harness guidance:** keep this cable short and, if using discrete wires rather than a ribbon, pair SDA and SCL each with an adjacent GND (or lightly twist) — this cable runs near the DRV8874/motor switching noise this board's EMC audit (§10) was written around, and I²C has no error correction to fall back on if a transaction glitches. |
| **Main PCB mounting holes** | Genuinely missing until this audit — nothing addressed how this board physically attaches to anything (a chassis, fixture, or eventual robot mount). **Decided: 4× M3 mounting holes**, one near each corner, with standard clearance from components and copper per JLCPCB's keep-out rules. |
| **Test points** | Not previously included. Exposed pads/points for **3.3V, 5V, VM, and the `IPROPI` voltage** — cheap, easy to add during layout, and directly useful across the multi-stage bring-up sequence in §7 (probing rail health and the current-sense signal without hunting for a component leg). |

### 3.5 Motor

| Component | Role | Key specs | Local availability |
|---|---|---|---|
| **JGA25-370, 12V, 133 RPM, 20 kg·cm** | The actuator this board drives. Brushed DC gearmotor with an accessible rear shaft stub (before the gearbox) for AS5600 magnet mounting. | 12V nominal (6–18V range), 1:78 metal gearbox, 4mm D-shaft output, free-run current 50 mA, **stall current 1.2 A** | ✅ Local (Electra Store, Egypt — 360 EGP, currently on backorder) |



## 4. Design Notes Worth Remembering

**Why DRV8874 changes the BOM from earlier drafts.** Earlier planning assumed a DRV8871 substitute (locally available, but lacking integrated current sense), which required adding an external shunt resistor plus an INA240-family current-sense amplifier. Since you're proceeding with the real DRV8874, **that external current-sense stage is no longer needed** — the DRV8874's IPROPI pin does the job internally. This simplifies the board: one fewer IC, one fewer Kelvin-sensing layout problem. (You still get real power-stage layout experience from the buck converter and the motor-current traces themselves, so the core learning goal is intact.)

**Why there's no separate 5V rail anymore.** ~~The AS5600 (unlike the mechanical encoder originally considered) runs happily on 3.3V. Nothing else on this board needs 5V.~~ **Superseded — the 5V rail is back, but now as an intentional intermediate stage rather than an end rail.** Nothing on the board *needs* 5V directly (STM32 and SN65HVD230 are 3.3V-only; AS5600 is fine at 3.3V), but the power architecture is now two stages: `LM2596S-5V` (switching, noisy but efficient) feeding `AMS1117-3.3` (linear, clean but lossy). This trades a small amount of efficiency (the LDO burns the 5V→3.3V difference as heat, which is negligible at this current draw) for a materially quieter 3.3V rail — worth it specifically because that rail also feeds the ADC reading motor current. It also removes the feedback-resistor-divider calculation that the single-stage `LM2596S-ADJ` approach required, since both `LM2596S-5V` and `AMS1117-3.3` are fixed-voltage parts.

**Voltage safety check, already applied to this BOM:**
- STM32 and SN65HVD230 → 3.3V only, strictly.
- DRV8874 has two separate supply inputs — VM (motor voltage, e.g. 12V) and VCC (logic supply, 3.3–5V) — don't cross them.
- AS5600 → 3.3–5V, flexible.

**Encoder resolution, through the gearbox.** The AS5600 gives 4096 counts/rev at the motor's fast shaft. Through the JGA25-370's 1:78 gear ratio, that's `4096 × 78 = 319,488` counts per revolution of the actual output shaft — about 0.0011° per count. Resolution is not a limiting factor for this build; mechanical backlash in the gearbox will dominate positioning accuracy long before encoder resolution does.

**How cycle-by-cycle current limiting actually behaves.** When motor current crosses `I_TRIP` (2.0A) mid-PWM-cycle, the DRV8874 doesn't wait for the cycle to end — after a short blanking + deglitch delay (tens to hundreds of nanoseconds, rejects switching noise rather than reacting to it), it chops the output off for whatever remains of that period, then tries again at the next period boundary. Current decays through the motor's normal freewheeling path during that off-time — nothing is "held" at a fixed value, it's the same inductor decay physics as a normal PWM off-time, just triggered early.

The off-duration is **not constant** — it depends on duty cycle. At 20kHz (`T = 50µs`), if the trip happens right after blanking ends, the off-duration is close to a full period (~50µs) regardless of duty cycle. But the *shortest possible* off-duration shrinks as duty cycle rises — at 90% duty cycle the floor is only ~5µs, at 30% duty cycle it's ~35µs — because that floor is just the duty cycle's normal off-time (`(1−D) × T`). Since the PID loop changes `D` every cycle, this floor moves around continuously during operation. Practical effect: average current during a sustained stall isn't perfectly flat — it has some cycle-to-cycle wobble tied to duty cycle, visible as a slightly uneven (though still safely bounded) shape if you ever plot `IPROPI` during a stall event. This was weighed against fixed off-time (a constant off-duration set by an external timer, giving a flatter stall-current profile) and cycle-by-cycle was kept anyway, because fixed off-time introduces a second hardware timer that can briefly override firmware PWM commands during a stall — added complexity not worth it for a general-purpose gear motor that isn't delicate.

Two safety layers exist independently: your 2.0A `I_TRIP` (tuned to this motor's real operating range, described above) and the DRV8874's own separate, much higher hardware overcurrent protection (OCP) that protects the chip itself against events like a dead short — not user-configured, always active regardless of `I_TRIP`/`IMODE` settings.

**DRV8874 sourcing.** Since it's not stocked in Egypt, budget for international shipping lead time (Digikey/Mouser/RS/TME direct, or a pre-assembled Pololu carrier board as a fallback that trades layout learning for reliability). Order this part first — it's the long pole in getting the board built.

**Ground return path — layout guidance, not a component.** The 4-layer stackup gives a solid ground plane (L2), but a single plane doesn't automatically mean current returns behave well. The DRV8874's motor return current and the sensitive `IPROPI`/AS5600 signal paths sharing that same plane can couple noise into each other if component placement isn't deliberate. Practical guidance for layout: keep the motor power return path physically separated from the sensitive analog area, converging at a single point near the power entry — a star-point strategy even though it's technically one continuous plane, not physically split copper.

**DRV8874 power-up sequencing — flagged, not resolved.** Some H-bridge drivers have a preference for which supply (logic `VCC` vs. motor `VM`) powers up first. This hasn't been verified against the DRV8874 datasheet specifically — worth checking directly during schematic capture rather than assuming either order is fine.

---

## 5. Decisions Required Before Schematic Capture

These aren't sourcing gaps — they're design choices that determine how the schematic gets drawn. Working through them now avoids redrawing later.

| Decision | What it affects | Status |
|---|---|---|
| **DRV8874 `PMODE` (control interface)** | ✅ **Decided: PH/EN mode.** `IN1` → MCU GPIO (direction), `IN2` → MCU timer/PWM channel (speed, the current-loop output). Simplest option, one timer channel, leaves more MCU timer resources free for the encoder and anything added later. | Closed |
| **DRV8874 `IMODE` (current regulation behavior)** | ✅ **Decided: cycle-by-cycle.** When current hits `I_TRIP` (2.0A), the driver chops the output off for whatever remains of the current PWM period, then retries at the next period boundary — off-duration varies with duty cycle (see design notes below). Chosen over fixed off-time to avoid a second hardware timer that could override firmware PWM commands during a stall, keeping the current-loop behavior easier to reason about while debugging. | Closed |
| **`nFAULT` pin wiring** | ✅ **Decided: `PB2`, with a 10kΩ pull-up to 3.3V.** Open-drain, active-low fault output from the DRV8874 (overcurrent shutdown, thermal shutdown, undervoltage, charge-pump fault). `PB2` supports EXTI2 for immediate interrupt-driven fault handling rather than polling. **Tentative** until the full pin map is finalized in the STM32 pin assignment pass below — timer/encoder pin choices could still shift things. | Closed (tentative) |
| **CAN message protocol** | ✅ **Decided.** 500 kbps, standard 11-bit IDs. Command frames at `0x100+node_id` (host→node), telemetry at `0x200+node_id` (node→host). `node_id` set in firmware at flash time (no jumpers/DIP switches needed). Full payload layout below. | Closed |
| **PCB layer count** | ✅ **Decided: 4-layer.** Stackup: L1 signal/components, L2 solid ground plane, L3 power plane (3.3V + 12V), L4 signal. Chosen for EMC (dedicated ground plane rather than a pour fighting for space with signal routing, avoiding the fragmentation problem you diagnosed on ESP-Drone's 2-layer board) and to handle the compact, motor-adjacent form factor without crosstalk between the CAN pair, I²C bus, and current-sense signal. | Closed |
| **Magnet part number** | ✅ **Decided (spec confirmed, sourcing external).** 6mm diameter × 2.5mm thick, diametrically magnetized NdFeB, N35–N42 grade — confirmed against the AS5600 datasheet. **Not stocked locally**: checked UGE, Micro Ohm, and Future Electronics directly — none carry a standalone diametric magnet, and the Micro Ohm AS5600 module ships without one (currently also out of stock). Source via AliExpress (commonly listed for AS5600/AS5048 use) or a specialty magnet supplier. Low-stakes, cheap, doesn't block schematic capture or PCB ordering — only needed at mechanical bring-up. | Closed |
| **H-bridge PWM switching frequency** | ✅ **Decided: 20kHz.** Sits at the edge of the audible range (below ~20kHz, motor windings produce audible switching whine), matches the top end of the current loop's target bandwidth (10–20kHz) rather than falling short of it, and keeps DRV8874 switching losses low. Timer resolution checked and not a constraint: at the STM32's 72MHz timer clock, 20kHz gives 3600 discrete duty-cycle steps (~0.028% resolution). Consistent with the `T=50µs` assumption already used in the earlier current-limiting walkthrough. | Closed |
| **VM-side bulk capacitance / transient protection** | ✅ **Decided: 220–330µF/25V electrolytic + 0.1–1µF ceramic** at the DRV8874's VM pin. See §3.2 BOM for the sizing calculation. | Closed |

### 5.1 CAN Protocol Definition

**Bit rate:** 500 kbps — standard for short-distance robotics CAN, comfortable margin at this bus length/node count.

**Frame format:** Standard 11-bit ID (not extended). 2048 possible IDs is far more than needed for a handful of motor nodes; extended IDs add complexity with no benefit at this scale.

**ID allocation:**
```
Command IDs:    0x100 + node_id   (host → motor node)
Telemetry IDs:  0x200 + node_id   (motor node → host)
```
`node_id` is set in firmware at flash time — no jumpers or DIP switches, one less connector and one less thing to get wrong in layout. Node 1: commands on `0x101`, telemetry on `0x201`. Node 2: `0x102`/`0x202`, and so on. `0x000`–`0x0FF` reserved for future bus-wide messages (broadcast, emergency stop).

**Command frame** (host → node, ID `0x100+n`, 8 bytes):

| Bytes | Field | Type |
|---|---|---|
| 0–3 | Setpoint value | `float32` |
| 4 | Control mode (0=position, 1=speed, 2=current) | `uint8` |
| 5–7 | Reserved | — |

**Telemetry frame** (node → host, ID `0x200+n`, 8 bytes):

| Bytes | Field | Type |
|---|---|---|
| 0–3 | Current position | `float32` |
| 4–5 | Current motor current (mA) | `uint16` |
| 6 | Fault flags (bit 0=overcurrent, bit 1=stall, etc.) | `uint8` |
| 7 | Reserved | — |

Both directions fit in a single classic CAN frame — no multi-frame reassembly needed. Since the full cascaded PID loop (current→speed→position) runs locally on each node, the bus only ever carries setpoints and status, not raw control-loop data, so bandwidth is not a concern even with several nodes sharing the bus.

### 5.2 STM32 Pin Assignment (running tracker — built incrementally)

Populated as each decision above locks in a pin. Not final until every row is filled.

| Pin(s) | Function | Status | Note |
|---|---|---|---|
| `PA13`, `PA14` | SWD (program/debug) | Fixed | Never available for reassignment |
| `PB8` | CAN1_RX (bxCAN, remapped) — SN65HVD230 `R`/RXD pin drives this | Assigned | Full remap (`AFIO_MAPR CAN_REMAP=10`), off default `PA11` to keep it free of the USB/CAN SRAM-sharing constraint entirely |
| `PB9` | CAN1_TX (bxCAN, remapped) — drives SN65HVD230 `D`/TXD pin | Assigned | Full remap, off default `PA12`. Direction matches the CAN adapter board's PA12(TX)/PA11(RX) wiring (§8) — TX always drives the transceiver's D input, RX always reads the transceiver's R output |
| `PB6`, `PB7` | I²C1 (AS5600 encoder) | Reserved | Default I²C1 pins |
| `PB2` | `nFAULT` (DRV8874) | **Tentative** | EXTI2-capable, no conflicts yet — may shift once timer pins below are chosen |
| `PA5` | `IN1` — direction (DRV8874, PH/EN mode) | Assigned | Plain GPIO output, no special peripheral needed |
| `PA6` | `IN2` — PWM speed (DRV8874, PH/EN mode) | Assigned | `TIM3_CH1`, no remap needed. TIM4 deliberately avoided — its default pins overlap I²C1 (`PB6`/`PB7`) and the remapped CAN pins (`PB8`/`PB9`) |
| `PA0` | `IPROPI` → ADC (current sense) | Assigned | `ADC1_IN0`. **This pin was missing from the tracker entirely until now** — the current-sense signal needs an ADC-capable pin same as any other input, and it hadn't been assigned. Caught while closing out `IN1`/`IN2`. |

**Pin tracker complete.** Every signal identified so far has a home with no conflicts.

**Encoder timer input — resolved, not needed.** Hardware encoder-timer mode exists for reading raw A/B quadrature pulse trains, which is what the mechanical `YT06-OP` encoder from earlier comparisons would have produced. The AS5600 doesn't work that way — it's read over I²C as an absolute angle value on demand, with no pulse stream to lock a hardware timer onto. Position is read by issuing an I²C request each control-loop tick (~100Hz per the cascaded-loop plan), not by counting pulses. No pin or peripheral needed beyond the I²C1 bus already reserved above.

---

## 6. Open Items Before Schematic Capture

- [x] ~~Confirm target motor and its expected stall/running current~~ → **JGA25-370, 12V, 133 RPM: free-run 50 mA, stall 1.2 A**
- [x] ~~Size `R_IPROPI`~~ → **2.2 kΩ**, full-scale ADC at ~3.3A
- [x] ~~Confirm buck inductor value~~ → **68 µH, `CD54-680M`, fully specified — see item below**
- [x] ~~Add crystal oscillator (was missing entirely from the BOM)~~ → **8MHz, 3225-SMD, C_L=20pF, UGE Electronics Egypt — C1=C2=20pF load caps**
- [x] ~~Calculate the VREF resistor divider values~~ → **R1 = 3.3kΩ, R2 = 5.1kΩ**, gives VREF ≈ 2.0V, I_TRIP ≈ 2.0A. Add 100nF filter cap at the VREF node.
- [x] ~~Assign real STM32 pins for `IN1`/`IN2`~~ → **`IN1`=`PA5`, `IN2`=`PA6`(`TIM3_CH1`)**. Also caught and assigned `IPROPI`→**`PA0`(`ADC1_IN0`)**, which had no pin at all until now.
- [x] ~~Check AMS1117-3.3's minimum output capacitor~~ → **22µF tantalum output, 10µF tantalum/electrolytic input** — ceramic avoided due to ESR mismatch with the regulator's stability requirement (0.3–22Ω range; typical ceramic ESR is far below that)
- [x] ~~Size the VM-side bulk capacitor~~ → **220–330µF/25V electrolytic + 0.1–1µF ceramic**, sized from an energy estimate at I_TRIP
- [x] ~~Decide CAN connector standard~~ → **JST-XH 2.5mm, 3-position (CAN-H/CAN-L/GND), two per board for daisy-chain topology**
- [x] ~~Confirm the H-bridge PWM switching frequency~~ → **20kHz**, edge of audible range, matches current loop's target bandwidth ceiling
- [x] ~~Pick an exact 68µH inductor part number~~ → **`CD54-680M`, DCR 0.46Ω max, IDC 0.61A max — datasheet-confirmed, no outstanding verification needed**
- [x] ~~Design reset circuit~~ → **100nF cap + momentary pushbutton on `NRST`→GND**, no external pull-up needed (internal ~40kΩ built into the chip)
- [x] ~~Decide `BOOT0` pin strapping~~ → **Fixed 10kΩ pull-down to GND**, no jumper — SWD/ST-Link programming never needs the system bootloader
- [x] ~~Add reverse-polarity protection on the 12V input~~ → **Series Schottky diode, reusing the buck stage's catch diode part (3A SS34-class)** — placed in +12V line before the split to DRV8874 VM and the buck stage
- [x] ~~Add ESD/transient protection on CAN bus lines and motor connector~~ → **CAN: `NUP2105L` dual-channel TVS array, confirmed local at UGE, industry-standard CAN-specific part. Motor: generic bidirectional TVS across terminals, lower priority but included.**
- [x] ~~Finalize I²C pull-up value~~ → **4.7kΩ, calculated and confirmed against the I²C-bus spec formula (valid range 966Ω–23.6kΩ for this bus), not just assumed**
- [x] ~~Pick specific rated parts for the power input and motor output connectors~~ → **2-pin, 5.08mm screw terminal (10–16A rated), same part reused for both — confirmed local at UGE and Micro Ohm, ~5–7× headroom over actual worst-case current (~2.1–2.2A)**
- [x] ~~Decide the physical implementation of the CAN termination jumper~~ → **2-pin, 2.54mm pin header + removable shunt cap, in series with the 120Ω resistor** — matches CVRA's reference design, supports future daisy-chain reconfiguration without a soldering iron
- [x] ~~Wire AS5600 power pins correctly for 3.3V single-supply operation~~ → **`VDD5V` and `VDD3V3` tied together, both fed from 3.3V rail, plus a 1µF cap on the internal LDO pin — real datasheet requirement, previously missed**
- [x] ~~Specify STM32 decoupling capacitors~~ → **Precise count: 3× 100nF (digital VDD pins) + 4.7µF bulk + 100nF on VDDA with series ferrite bead — VDDA isolation directly protects the current-sense ADC reading, the whole reason for the two-stage power architecture**
- [x] ~~Note DRV8874 thermal pad handling~~ → **Via-stitching to copper pour required, HTSSOP-16 PowerPAD package, relevant given regular `I_TRIP` chopping events**
- [x] ~~Add local decoupling for SN65HVD230 and DRV8874 VCC~~ → **100nF each, standard practice, previously unlisted**
- [x] ~~Assign SN65HVD230 `RS` and `Vref` pins~~ → **`RS` tied directly to GND (high-speed mode, no standby/sleep needed); `Vref` left floating (N/C) since standard, not split, termination is used** — datasheet-verified, closed via direct cross-check against `SN65HVD230` datasheet during schematic capture
- [x] ~~Document CAN1_TX/RX direction on the remapped `PB8`/`PB9` pins~~ → **`PB8`=CAN1_RX (from transceiver `R`/RXD), `PB9`=CAN1_TX (to transceiver `D`/TXD)** — previously listed as a combined "CAN" pair with no direction; now matches the CAN adapter board's PA11/PA12 wiring convention exactly
- [x] ~~Add a power-on indicator~~ → **LED + series resistor on the 3.3V rail — genuinely missing until this audit, no "is it alive" signal existed anywhere**
- [x] ~~Add main PCB mounting holes~~ → **4× M3, corner-placed, standard JLCPCB clearance — nothing addressed how this board physically attaches to anything**
- [x] ~~Specify the AS5600 sub-PCB link connector~~ → **JST-SH, 1.0mm, 4-position — closes the earlier vague "short cable" description**
- [x] ~~Add bring-up test points~~ → **3.3V, 5V, VM, IPROPI — cheap, aids the multi-stage bring-up sequence in §7**
- [x] ~~Address ground return path routing~~ → **Layout guidance added (§4): keep motor return current physically separated from the sensitive analog area, star-point convergence near power entry, even on one continuous plane**
- [x] ~~Decide AS5600 `DIR` pin handling~~ → **3-pad solder jumper (`JP1`)**, center=`DIR`, sides=`+3V3`/`GND` — defers CW/CCW polarity to assembly time rather than guessing at schematic time
- [x] ~~Terminate AS5600 `OUT`/`PGO` pins~~ → **Both No-Connect** — `OUT` unused (I²C read, not analog/PWM), `PGO` floats on its internal pull-up (I²C programming via Option A, not the OUT-pin Option B path)
- [x] ~~Reconcile AS5600 decoupling values with the actual 3.3V-mode datasheet diagram~~ → **`C1=100nF` + `C2=10µF`** (matches datasheet Fig. 13's 3.3V-operation diagram; the earlier "1µF" note referenced the 5V-mode LDO table, which doesn't apply once `VDD5V`/`VDD3V3` are tied together)
- [x] ~~Confirm all five schematic subsystem sheets are drawn~~ → **Power Stage, MCU Core, Motor Drive, CAN Bus, and Sensing all complete in KiCad**
- [x] ~~Add motor output connector protection~~ → confirmed present in schematic (generic bidirectional TVS across terminals, per §3.4)
- [x] ~~Add main PCB mounting holes to schematic~~ → confirmed present (4× M3, corner-placed)
- [ ] **Flagged, unresolved:** DRV8874 power-up sequencing (VCC vs. VM order) — not verified against the datasheet, check during schematic capture
- [ ] **New, before ERC/layout:** cross-check main-board ↔ AS5600 sub-PCB connector pin order matches 1:1 (§3.4) — both footprints were placed independently and haven't been diffed against each other yet

- [~] ~~Design the AS5600 mounting bracket~~ → **Spec drafted (§9), but finalization deliberately deferred until the AS5600 sub-PCB layout is confirmed** — same logic as the CAN adapter firmware deferral
- [ ] **External, in progress:** source DRV8874 internationally
- [ ] **External, in progress:** source a *confirmed diametrically-magnetized* magnet — verify magnetization type before buying anything local

---

## 7. Build Sequence

1. Schematic capture (CAN node + power stage + DRV8874 + AS5600), reference CVRA/Cormack boards
2. ERC clean
3. Layout — Kelvin considerations mostly retired by DRV8874's integrated sense, but power-trace sizing and switching-node layout on the buck stage remain the core exercise
4. DRC clean, JLCPCB stackup check, order
5. Bring-up: power on, confirm 3.3V rail, confirm CAN enumerates, confirm AS5600 reads a sane angle at standstill, *then* close loops one at a time — current first, then speed, then position
6. Bring up the CAN adapter (§8) and use it from a laptop to send real command frames (`0x101`) and read telemetry (`0x201`) back from the motor board — this is what actually proves the CAN half of the board works, not just that it powers on

---

## 8. CAN Adapter (Supporting Tool — Separate Board)

Not part of the motor controller node itself — this is a second, small board whose only job is letting your laptop send/receive real CAN frames to test board #1. Not confirmed available locally as a ready-made purchase (checked UGE, Micro Ohm, Future Electronics — nothing found), so this is a build, using parts already on hand or already being sourced for the main board.

**Why a second chip, not the motor board's own USB:** on the STM32F103, the CAN peripheral and USB peripheral share the same SRAM block and can't run simultaneously. A USB-CAN bridge needs both active at once by definition, so this can't run on the motor board's own MCU, and can't be merged onto the same board as a second chip either without effectively building two boards fused together for no real benefit — better to keep it fully separate and reusable for future CAN nodes too.

### 8.1 Hardware

| Part | Role | Source |
|---|---|---|
| Blue Pill (STM32F103C8T6) | Runs the bridge firmware | Already owned, same as maglev board |
| `SN65HVD230` | CAN transceiver — talks to the bus | Same part as the motor board's BOM — source a second one |
| USB-to-UART module (CH340G or FT232) | Bridges PC USB ↔ Blue Pill UART | New, cheap, common — same sourcing tier as passives |
| ST-Link programmer | Flashes the firmware | Already owned |

### 8.2 Wiring

**Power:** no separate design needed — the Blue Pill's onboard 5V→3.3V regulator handles everything, powered directly from the USB connection to the laptop. The `SN65HVD230` taps 3.3V from the Blue Pill's own regulated output.

**Termination:** with exactly two devices on the bus (this adapter and the motor board), **both ends need their `120Ω` termination jumper enabled** — they're the only two physical points on the wire. If a third node is ever added later, this changes: only the two true physical ends of the bus stay terminated.

**CAN transceiver to STM32** (native default pins — safe here since USB is never used on this chip):
```
SN65HVD230 VCC  → STM32 3.3V
SN65HVD230 GND  → STM32 GND
SN65HVD230 TXD  → STM32 PA12 (CAN1_TX)
SN65HVD230 RXD  → STM32 PA11 (CAN1_RX)
```

**CAN transceiver to the bus** — same 3-pin JST-XH standard (CAN-H, CAN-L, GND) as the motor board, with the same `NUP2105L` TVS protection at the connector (see main board §3.4 for the part). This adapter becomes the second physical end of the bus, so its `120Ω` termination jumper needs to be enabled.

**USB-UART bridge to STM32:**
```
Bridge TX  → STM32 PA10 (USART1_RX)
Bridge RX  → STM32 PA9  (USART1_TX)
Bridge GND → STM32 GND
Bridge USB connector → laptop
```

### 8.3 Firmware and PC side

**Firmware approach — decided, deferred until after both PCBs are designed:** write the SLCAN-over-UART bridge firmware from scratch rather than adapting an existing open-source implementation. Chosen deliberately over the faster path, since it's a small, well-scoped, self-contained firmware task (parse text commands over UART → translate to `bxCAN` register calls → encode received frames back to text) that's genuine CAN/embedded skill-building in its own right, consistent with the rest of this project's purpose. **⚠ Do not start this until schematic + layout for both the motor board and this adapter board are finished — this is a reminder for both of us to come back to it, not a task to pick up mid-PCB-design.**

Flash via SWD/ST-Link, same process as any other STM32 board. On the PC, the bridge module enumerates as an ordinary COM/serial port. `python-can` talks to it directly once the firmware exists:

```python
import can
bus = can.interface.Bus(channel='COM5', bustype='slcan', bitrate=500000)

# Send a command: node 1, position mode, setpoint = 90.0 degrees
msg = can.Message(arbitration_id=0x101, data=[...float32(90.0)..., 0x00], is_extended_id=False)
bus.send(msg)

# Read telemetry back (arrives on 0x201)
response = bus.recv(timeout=1.0)
```

### 8.4 Full data path

```
Python script → USB → CH340/FT232 → UART → STM32 (SLCAN firmware)
    → bxCAN peripheral → SN65HVD230 → CAN-H/CAN-L → motor board's SN65HVD230
    → motor board's STM32 → motor board's control loop firmware
```

---

## 9. AS5600 Mounting Bracket (Mechanical Spec — SolidWorks Task)

**Confirmed motor dimensions** (cross-checked across manufacturer manual and independent listings): body diameter **25mm**. Front output shaft: 4mm diameter × 9.5mm length (not relevant to this bracket — it's the rear shaft stub that matters here). **Rear shaft stub dimensions are not published anywhere** — checked the manufacturer manual and multiple retailer listings, only the front shaft is dimensioned. Design around this gap rather than guessing a number (see adjustment allowance below).

**Structure:** a small clamp/collar sized to the motor's confirmed 25mm body OD, with an arm extending to the rotation axis holding the AS5600 sub-PCB at the correct standoff. The rear end of this motor has no mounting holes to bolt to — the datasheet's `2× M3, 4.5mm deep, 17mm spacing` holes are on the gearbox/front end, for chassis mounting, not the bare motor can — so this has to be a clamp, not a bolt-on bracket.

**Magnet mounting:** glued flat onto the shaft stub's end face, centered on the rotation axis — see §3.3 sensing table for why this avoids needing the unpublished shaft diameter.

**Air gap target:** 1.0–1.5mm between the magnet and the AS5600 die (within the datasheet's 0.5–3mm working range, biased toward the lower/tighter end for best signal strength).

**Handling the unconfirmed shaft length:** design the standoff arm with a **slot rather than a fixed hole**, giving a few millimeters of adjustment travel. Tune the actual air gap once the motor physically arrives and the real shaft protrusion can be measured directly — consistent with the same "confirm against the physical part" approach already used elsewhere for this build (e.g. motor socket ID, battery bay sizing on other projects).

**Status:** Spec drafted (motor dimensions, clamp approach, magnet mounting method, air gap target) but **finalization deliberately paused** until the sub-PCB layout is confirmed — the bracket needs to interface with real mounting holes and the cable exit point on the actual AS5600 sub-PCB, neither of which exist yet. Same logic as the CAN adapter firmware deferral in §8.3: this is a reminder to return to it once the PCB is laid out, not a task to finish speculatively now.
