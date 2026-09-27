# DDD-003 — DRV8874 VTG-actuator drive: endstop/stall detection via IPROPI

- Date / authoring session: 2026-09-21
- Status: Decided

## Change

Drive the electric VTG actuator with the **DRV8874PWPR** (U2 on `DRV8874.SchDoc`),
PH/EN control mode, using integrated IPROPI current sensing for **endstop / stall
detection**. **ITRIP set below the mechanical stall current** so cycle-by-cycle
current regulation engages just before stall (protects actuator + driver), and a
**12 V-class bidirectional TVS** on the H-bridge outputs (not the 5 V-logic
PESD5V0S2BT).

Stall current at endstop: **3.7 A**.

## Change vs. adjacent decisions

- **PESD5V0S2BT (DDD-002) applies ONLY to 5 V logic I/Os.** It is neither the right
  voltage (5 V standoff) nor power class (130 W) for the 12 V H-bridge outputs.
  OUT1/OUT2 swing on Ubatt → need a 12 V-system bidirectional TVS (see below).

## Key parameters

| Element | Value | Role |
|---------|-------|------|
| Driver | DRV8874PWPR (HTSSOP16) | 6 A peak, 4.5–37 V VM, 200 mΩ HS+LS |
| Control mode | PH/EN (PMODE = GND) | PH/IN2 = direction, EN/IN1 = PWM enable |
| IPROPI resistor | **RIPROPI = 1.5 kΩ** (1%) | maps load current to ADC voltage |
| **ITRIP** | **≈ 2.8 A** (below 3.7 A stall) | current-regulation trip |
| VREF | **≈ 1.9 V** (divider from VCC) | sets ITRIP: `ITRIP×450 µA/A = VREF/RIPROPI` |
| IMODE | **Quad-Level 2 = 20 kΩ → GND** | cycle-by-cycle + nFAULT-as-chopping-indicator |
| nFAULT pullup | 4.7–10 kΩ to VCC | open-drain fault/indicator |
| nSLEEP | active-high enable | tie high to run; low = <1 µA sleep |
| VCP ↔ VM cap | 100 nF, 16 V, X5R/X7R | charge-pump (mandatory) |
| CPH ↔ CPL cap | 22 nF, VM-rated, X5R/X7R | charge-pump |
| VM bypass | 0.1 µF + bulk (VM-rated) | supply decoupling |
| IPROPI noise cap | ~10 nF optional | filter motor transients |

## Why (ITRIP below stall — the core decision)

- **Protect the actuator & driver:** current regulation exists expressly to limit
  output current on *"motor stall, high torque, or other high current load events"*
  (DRV8874 Rev A §7.3.3). Setting ITRIP below the 3.7 A mechanical stall means the
  driver clips the current before a hard endstop slam, instead of letting the motor
  run into the full stall and heat the HTSSOP/pump the supply.
- **Does NOT limit normal torque:** cycle-by-cycle chopping only engages when
  `VIPROPI ≥ VVREF`. Normal running current stays well below ITRIP, so torque in
  normal travel is unaffected — the limit only acts near the endstop as intended.
- **TI app-note guidance is explicit:** SLVAFQ3 *"Integrated Stall Detection"* and
  the TI E2E FAQ: *"the STALL threshold must be set … **below the ADC value of the
  current at the ITRIP threshold**"* and *"the initial inrush and stall current are
  **limited to the ITRIP threshold**."*

### Sizing math (datasheet Eq. 3: `ITRIP × AIPROPI = VVREF / RIPROPI`, AIPROPI = 450 µA/A)

- **RIPROPI = 1.5 kΩ**, ITRIP = 2.8 A → `VVREF = 2.8 × 0.45 × 1.5 = 1.89 V` → divider
  to **~1.9 V**.
- (Alternative at VREF = 2.5 V: `RIPROPI = 2.5/(2.8×0.45) ≈ 1.98 kΩ`.)

### Detection paths (both active)

1. **Analog:** IPROPI → MCU ADC. Normal running reads low; endstop drives it toward
   VREF (~1.9 V). Firmware flags stall on IPROPI above a threshold *sustained* past
   the startup inrush.
2. **Digital:** in cycle-by-cycle (IMODE Q2), **nFAULT pulls low on current chopping**
   = endstop reached. Distinguish from a real fault: chopping-indicator only asserts
   while commanding a drive state.

### Firmware notes (inrush vs. stall — SLVAFQ3 / E2E FAQ)

- Startup **inrush exceeds ITRIP too** — blank/ignore for a few ms after enabling,
  then assert stall only on *prolonged* ITRIP/nFAULT while driving.
- On endstop detection: back off / reverse / stop rather than holding full torque
  (avoids sustained chopping at ~2.8 A). Decelerate briefly before reversing to avoid
  supply pump-up on rapid direction change (TI forum).

## H-bridge output protection (12 V system, NOT the 5 V PESD)

- **Bidirectional TVS, Vrwm ≈ 16 V, SMB package (600 W)** across OUT1–OUT2 (and/or
  OUT↔GND) at the actuator connector.
- Keeps the diode off during normal ~14.4 V charging; clamps ISO 16750-2 / ISO 7637-2
  load dump (12 V-system pulse 5a: up to 35 V @ ≤4 Ω ≤400 ms). Verify energy against
  the load-dump power-vs-time curve (Littelfuse "TVS Diodes to Meet Automotive Load
  Dump Standard" app note).
- The PESD5V0S2BT stays **only** on 5 V logic signal lines per DDD-002.

## Open / verify

- TVS part number to stock (e.g. SMBJ15CA/16CA/18CA; pick standoff so it's off at
  max valid Ubatt ~14.4 V but clamps load dump).
- Final ITRIP tuned per actuator (SLVAFQ3: threshold is application-specific;
  verify running current + inrush empirically).
- Confirm ripple-counting/hall is not needed (position detection is via endstop-stall
  only).

## Files

- `DRV8874.SchDoc`
- `DRV8874.Harness` (`MotCtrl = PWM,DIR,nFAULT,nSLEEP`)

## References

- DRV8874 datasheet (Rev A / B) — §§7.3.2 PMODE, 7.3.3 current sense & regulation,
  Table 7-6 IMODE, recommended component table.
- TI **SLVAFQ3** *Integrated Stall Detection for Brushed DC Motors* (app brief).
- TI E2E FAQ *"How to detect motor stall using current sensing"* (DRV8251A technique,
  portable to any IPROPI driver).
- Littelfuse application note *"TVS Diodes to Meet Automotive Load Dump Standard"*
  (ISO 16750-2 / ISO 7637-2).

## Backlinks

- [Documentation Hub](../Main.md)