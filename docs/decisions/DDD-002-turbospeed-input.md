# DDD-002 — Turbo-speed input conditioning (5 V pulse → 3.3 V, Schmitt + RC)

- Date / authoring session: 2026-09-20, turbo-speed input design session
- Status: Decided

## Change

Added a conditioning stage for the Jaquet turbo-speed sensor output
(5 V square pulse, `f_out = f_blade / 8`, 375 Hz–3.125 kHz) before the
Stamp-S3A GPIO:

`Connector → PESD5V0S2BT (bidir TVS, across signal↔GND) → 10 kΩ → node → SN74LVC1G17 (Schmitt buffer, VCC = 3.3 V) → free GPIO`

At the node: **BAT54S** dual Schottky (diode to 3V3 + diode to GND, limiting input
swing both directions), **470 pF C0G to GND** (RC low-pass with the 10 kΩ → ~34 kHz
corner).

> **Board-wide convention:** the PESD5V0S2BT (LCSC C49338) is the standard connector
> ESD/TVS diode for **all 5 V I/Os** on this design, not just the turbo-speed input.

## Why

- **Level shift** the 5 V pulse into the 3.3 V domain at the gate's output.
- **Schmitt trigger** (SN74LVC1G17) reshapes slow/noisy edges and adds hysteresis,
  so the MCU sees clean logic.
- **10 kΩ series** is the current limiter that makes the clamps safe
  (an 80 V surge ⇒ only ~7 mA through the diode).
- **BAT54S dual Schottky** clamps the node in **both directions** (→ 3V3 for
  positive, → GND for negative), covering ESD/transient polarity both ways.
- **470 pF RC** (corner ≈ 34 kHz) strips HF noise/ringing while staying well above
  the 3.125 kHz max signal → no measurable edge lag on a frequency-measurement input.
- The 1G17 input is already 5.5 V-tolerant (HBM ±2 kV, CDM ±1 kV), so the clamps
  and RC are protective margins on top of that baseline.
- **Connector TVS** (PESD5V0S2BT, SOT-23) covers automotive-grade ESD: ±30 kV
  IEC 61000-4-2 Level 4, 12 A / 130 W IEC 61000-4-5 surge, AEC-Q101. A small
  low-capacitance ESD diode is the correct class for a *signal* line — a large
  600 W SMB TVS is only needed on power/bus rails, per TI/Nexperia application notes.

## Values / parameters

| Element | Value | Role |
|---------|-------|------|
| Connector TVS | **PESD5V0S2BT** (SOT-23) | bidir ESD/TVS, ±30 kV IEC-4-2, AEC-Q101; at connector |
| R series | 10 kΩ | current limit for clamps |
| Clamp | **BAT54S** (dual Schottky) | diode 1 → 3V3 (hi clamp), diode 2 → GND (neg clamp) |
| RC cap | **470 pF** C0G/NPO → GND | HF noise filter, ~34 kHz corner |
| Schmitt buffer | SN74LVC1G17, VCC = 3.3 V | clean logic out at 3.3 V |

Output lands on a **free, non-strapping GPIO** (avoid G0/G3/G46; see
[`docs/MCU.md`](../MCU.md)) — interrupt or PCNT both fine at ≤3.125 kHz.

## Open items / follow-up

- **Connector ESD** — resolved: PESD5V0S2BT (LCSC C49338) added (see values table).
- Edge delay at 470 pF is negligible for frequency counting; only raise the cap if
  confirmed in-band noise appears.
- The PESD5V0S2BT is the standard TVS for all 5 V I/Os; extend this chain pattern
  (TVS → limiter → clamp → RC → Schmitt/level stage) to other 5 V inputs as drawn.

## Files

- `Turbospeed.SchDoc`
- Buffer: SN74LVC1G17 (Nexperia): overvolt-tolerant inputs to 5.5 V, Schmitt action
- Sensor: Jaquet, ≈ VAG 05A927321F

## Backlinks

- [Turbo Speed Input](../Turbospeed.md)
- [Documentation Hub](../Main.md)