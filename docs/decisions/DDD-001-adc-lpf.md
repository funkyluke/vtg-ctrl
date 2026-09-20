# DDD-001 — ADC LPF: values + RC-before-active topology

- Date / authoring session: 2026-09-20, LPF simulation session
- Status: Decided

## Change

Locked the LPF input stage to `47k/47k/10k + 10n/22n/22n` with the AD8552 as a
unity-gain follower, and placed the RC stage **before** the active Sallen-Key
(rather than after it, as originally drawn).

## Why

- Suppression above ~200 Hz for the ADC input stage, with few BOM lines and
  common values.
- **Impedance invariance** vs. a SAR track-and-hold ADC (time-varying input
  impedance at sample rate): with the RC *after* the follower the ADC loaded
  through `R3/C3`; reordering makes the AD8552 follower the last stage, so the
  ADC only sees a low, stable source impedance.
- Earlier `C3 = 100n` put the RC pole at 159 Hz (below the Sallen-Key corner),
  dragging overall −3 dB to ~136 Hz and shallowing rolloff — wrong-side placement.

## Values / parameters

| Element | Value | Role |
|---------|-------|------|
| R3 | 10 kΩ | RC input stage |
| C3 | 22 nF | RC shunt to ground |
| R1 | 47 kΩ | Sallen-Key upper resistor |
| R2 | 47 kΩ | Sallen-Key lower resistor |
| C1 | 10 nF | Sallen-Key upper cap |
| C2 | 22 nF | Sallen-Key feedback cap |
| op-amp | AD8552 | unity-gain follower |

Distinct passive values: **47k, 10k, 10n, 22n** (4 lines). Sim-verified in
`LPF.asc`: −3 dB @ ~202 Hz, −20 dB @ ~590 Hz, ~−45 dB/dec (≈3rd order) rolloff.

## Supersedes / notes

- Replaces the earlier draft in `docs/Analog-ADC.md` which specified a TLV9001
  @ 250 Hz Butterworth; the working file uses AD8552 and is authoritative.