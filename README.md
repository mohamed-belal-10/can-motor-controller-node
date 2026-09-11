# CAN-Connected Closed-Loop DC Motor Controller Node

A single PCB "smart joint" for robot arms, rovers, or any multi-actuator robotic
system: it bolts onto a DC gearmotor, closes a cascaded current → speed →
position control loop locally on an STM32, and takes setpoints over a shared
CAN bus instead of running dedicated wires back to a central controller.
Multiple nodes share one twisted-pair CAN bus, each addressed independently.

A portfolio project by **Mohamed Belal ("Belo")** — Mechatronics Engineering,
GUC.

---

## Why this project

- **First board with a real power stage.** Current in the amps range, not the
  milliamps of a typical digital-signal board — Kelvin sensing considerations,
  thermal copper, and return-path separation under switching noise.
- **Introduces CAN bus** as a communication protocol, end to end: transceiver
  hardware, bxCAN peripheral configuration, bit-timing, and a companion
  USB-CAN adapter to test it from a laptop.
- **Introduces cascaded multi-loop control** (current → speed → position),
  the conceptual step up from a single-loop PID.
- Built from and cross-checked against two open reference designs: CVRA's
  `motor-control-board` (KiCad, CC-BY) and Cormack's STM32L433 motor driver.

---

## System overview

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

Two boards are designed here:

| Board | Role | Location |
|---|---|---|
| **`motor_controller`** | Main node — MCU, power stage, H-bridge, CAN interface | [`hardware/motor_controller/`](hardware/motor_controller/) |
| **`AS5600_Encoder_Node`** | Small sub-PCB carrying the bare AS5600 angle sensor, linked to the main board over a 4-pin JST-SH cable | [`hardware/AS5600_Encoder_Node/`](hardware/AS5600_Encoder_Node/) |

The AS5600 lives on its own tiny PCB — not a generic breakout module —
specifically so the main board's own calculated I²C pull-up resistors are
the *only* pull-ups on the bus (most off-the-shelf breakout modules bundle
their own, which would silently change the bus's effective pull-up value).

---

## Repository structure

```
.
├── README.md
├── LICENSE
├── docs/                                       Design documentation
│   ├── CAN_Motor_Controller_Node_Spec.md           Full BOM, design decisions, CAN protocol, pin map
│   └── CAN_Motor_Controller_Node_PCB_Layout_Guidelines.md   Placement, routing, plane strategy, DRC targets
└── hardware/
    ├── motor_controller/                       Main CAN motor controller node
    │   ├── motor_controller.kicad_pro/.kicad_sch/.kicad_pcb/.kicad_prl
    │   └── manufacturing/
    │       ├── gerbers/                            Gerber + drill files
    │       └── motor_controller-gerbers.zip         Fab-ready zip (JLCPCB format)
    └── AS5600_Encoder_Node/                    AS5600 magnetic encoder sub-PCB
        ├── AS5600_Encoder_Node.kicad_pro/.kicad_sch/.kicad_pcb/.kicad_prl
        └── manufacturing/
            ├── gerbers/
            └── AS5600_Encoder_Node-gerbers.zip
```

All KiCad projects are **KiCad 8+** format.

---

## Main board — `motor_controller`

### Core components

| Subsystem | Part | Notes |
|---|---|---|
| MCU | STM32F103C8T6 (LQFP-48) | 72 MHz, bxCAN, runs all 3 PID loops + CAN stack |
| Clock | 8 MHz HSE crystal, C_L = 20 pF | Required for reliable 500 kbps CAN bit timing (HSI is too loose) |
| Motor driver | DRV8874 (HTSSOP-16, PowerPAD) | Integrated current sense (IPROPI) and cycle-by-cycle current chopping — no external shunt/amplifier needed |
| Power stage | LM2596S-5V (buck) → AMS1117-3.3 (LDO) | Two-stage: switching for efficiency, then linear to give the current-sense ADC a clean reference |
| CAN transceiver | SN65HVD230 (SOIC-8) | 3.3V-only, high-speed mode, `NUP2105L` TVS array for bus ESD protection |
| Encoder | AS5600 (on its own sub-PCB) | 12-bit absolute angle over I²C — see below |
| Actuator | JGA25-370, 12V, 133 RPM, 20 kg·cm | Brushed DC gearmotor, 1:78 gearbox |

### Electrical specs

- **Motor supply:** 12 V nominal (6–18 V DRV8874 range)
- **Current limit:** 2.0 A hardware trip (`I_TRIP`, VREF divider-set), cycle-by-cycle chopping
- **PWM switching frequency:** 20 kHz (H-bridge), 150 kHz (buck)
- **Stackup:** 4-layer — L1 signal, L2 solid GND, L3 split power (VM/3.3V), L4 signal
- **CAN bus:** 500 kbps, standard 11-bit IDs, JST-XH 3-pos daisy-chain connectors (IN/OUT), jumper-selectable 120 Ω termination

### CAN protocol

| Frame | Direction | ID | Payload |
|---|---|---|---|
| Command | host → node | `0x100 + node_id` | `float32` setpoint, `uint8` mode (0=position, 1=speed, 2=current) |
| Telemetry | node → host | `0x200 + node_id` | `float32` position, `uint16` current (mA), `uint8` fault flags |

`node_id` is assigned in firmware at flash time — no address jumpers on the
board.

### Design status

- ✅ All five schematic sheets complete (Power Stage, MCU Core, Motor Drive,
  CAN Bus, Sensing), ERC clean.
- ✅ Every component value and part number closed, across four audit passes
  (full history in [`docs/CAN_Motor_Controller_Node_Spec.md`](docs/CAN_Motor_Controller_Node_Spec.md)).
- ✅ Layout complete; gerbers generated (see `manufacturing/`).
- ⚠️ **Flagged, unresolved:** DRV8874 power-up sequencing (VCC vs. VM order)
  not verified against the datasheet.
- ⚠️ **Open before re-ordering:** cross-check the main board's AS5600 link
  connector pin order against the sub-PCB's `J1`/`J7` connector — both
  footprints were placed independently.
- 🚚 **Externally sourced, not yet in hand:** DRV8874 (not stocked locally —
  ordered internationally), diametrically-magnetized magnet (must confirm
  magnetization type with seller before buying).

---

## Sub-board — `AS5600_Encoder_Node`

A minimal breakout for the bare AS5600 magnetic rotary position sensor
(SOP-8), designed specifically to avoid the double-pull-up problem that
off-the-shelf AS5600 modules introduce.

| Item | Detail |
|---|---|
| Sensor | AS5600 (SOP-8), 12-bit absolute angle over I²C |
| Power wiring | `VDD5V` and `VDD3V3` tied together, fed from the main board's 3.3V rail (required for correct 3.3V single-supply operation per datasheet) |
| Decoupling | 100 nF + 10 µF in parallel across the tied VDD node → GND (matches datasheet Fig. 13, 3.3V-mode) |
| Direction select | 3-pad solder jumper (`DIR`: GND = CW-increasing, VDD = CCW-increasing) — bridged at assembly time once motor orientation is known |
| Link to main board | JST-SH, 1.0 mm pitch, 4-position — carries `+3V3`, `SCL`, `SDA`, `GND` |
| Mounting | Glued flat to the motor's rear shaft-stub end face, read through a 1.0–1.5 mm air gap by a 6 mm diametric NdFeB magnet |
| Encoder resolution | 4096 counts/rev at the motor shaft → 319,488 counts/rev at the JGA25-370's output shaft (through its 1:78 gearbox) |

The AS5600 mounting bracket (clamp-style, sized to the motor's 25 mm body OD)
is specified mechanically in [`docs/CAN_Motor_Controller_Node_Spec.md`](docs/CAN_Motor_Controller_Node_Spec.md#9-as5600-mounting-bracket-mechanical-spec--solidworks-task)
but deliberately not finalized until the physical sub-PCB and motor are in
hand to confirm dimensions.

---

## Documentation

| Document | Covers |
|---|---|
| [`docs/CAN_Motor_Controller_Node_Spec.md`](docs/CAN_Motor_Controller_Node_Spec.md) | Full BOM with sourcing/pricing notes, design-decision rationale, CAN protocol definition, STM32 pin assignment tracker, CAN-adapter companion tool, AS5600 mounting bracket spec |
| [`docs/CAN_Motor_Controller_Node_PCB_Layout_Guidelines.md`](docs/CAN_Motor_Controller_Node_PCB_Layout_Guidelines.md) | Trace sizing, via strategy, floorplan, placement rules, ground/power plane strategy, CAN routing, thermal, DRC targets, pre-routing checklist |

---

## Build sequence

1. Schematic capture (CAN node + power stage + DRV8874 + AS5600) — **done**
2. ERC clean — **done**
3. Layout — **done**
4. DRC clean, JLCPCB stackup check, order
5. Bring-up: power rails → CAN enumeration → AS5600 sanity read → close current loop → speed loop → position loop
6. Bring up the companion CAN-USB adapter and drive the node from a laptop with real command/telemetry frames

Full detail on each step, including the CAN-USB adapter's own BOM, wiring,
and firmware plan, is in [`docs/CAN_Motor_Controller_Node_Spec.md`](docs/CAN_Motor_Controller_Node_Spec.md#7-build-sequence).

---

## Fabrication

Both boards are designed for JLCPCB (4-layer main board, 2-layer sub-PCB).
Ready-to-upload gerber/drill packages are checked in under each board's
`manufacturing/` folder. Re-export from KiCad's Plot dialog after any layout
change rather than editing the checked-in gerbers by hand.

## Reference designs

- [CVRA `motor-control-board`](https://github.com/cvra) — KiCad, CC-BY, near-identical spec; source for the CAN daisy-chain connector and termination-jumper approach.
- Cormack's STM32L433 motor driver — single-board layout reasoning, undervoltage-lockout-as-safety-feature technique.

## License

MIT — see [`LICENSE`](LICENSE).

This repository contains PCB hardware design files (KiCad schematics,
layouts, and manufacturing outputs) alongside documentation. The MIT License
is applied for simplicity and consistency with the author's other portfolio
repositories; no warranty is made or implied about the fitness of any
hardware design herein for use with live mains, high-current, or
safety-critical applications. Build and operate at your own risk.
