# Analog Input / ADC

> Draft — signal chain notes being captured as design progresses.

## Role

Analog acquisition chain: an 8-channel LPF bank (`LPF.SchDoc`, repeated `LPF0..LPF7`)
feeding the analog front-end / ADC, plus channel scaling and interface details.

## LPF — ~200 Hz 3rd-Order Low-Pass (Locked Design)

### Goal & constraints
- **ADC input stage** (feeds the analog front-end's SAR ADC). Needs good suppression
  **above ~200 Hz** (anti-alias) with **few BOM lines** and **common component values**.
- **ADC:** ADC128S052-class SAR converter with an internal **track-and-hold** — its
  input impedance is time-varying at the sample rate, which is the key driver of the
  topology below.

### Final values (`LPF.asc`, sim-verified)
| Element | Value | Role |
|---------|-------|------|
| `R3`  | **10 kΩ** | RC input stage |
| `C3`  | **22 nF** | RC shunt to ground (pairs with R3) |
| `R1`  | **47 kΩ** | Sallen-Key upper resistor |
| `R2`  | **47 kΩ** | Sallen-Key lower resistor |
| `C1`  | **10 nF** | Sallen-Key upper cap (from `+in` to gnd) |
| `C2`  | **22 nF** | Sallen-Key feedback cap (out → R1/R2 junction) |
| op-amp| **AD8552** | unity-gain follower (output feeds ADC) |
| .AC   | `dec 020 10 10000000` | 20 pts/dec, 10 Hz–10 MHz |

Distinct passive values: **47k, 10k, 10n, 22n** (4 lines). `C2` and `C3` share 22 nF;
`R1`/`R2` share 47 kΩ.

**Measured response (AC sim):** DC ≈ 0 dB, **−3 dB @ ~202 Hz**, −20 dB @ ~590 Hz,
−16.8 dB @ 500 Hz, rolloff ≈ **−45 dB/dec** (≈3rd order) above the corner.

### Opening the doc — effective input impedance (measured)
Seen from the source (`V(vin)/I(R3)`, AC sim with flat 1.0 V source):

| Freq | \|Zin\| | Phase |
|---|---|---|
| 10 Hz | ~363 kΩ | ≈ −93° (≈ capacitive) |
| 100 Hz | ~45 kΩ | ≈ −117° |
| 200 Hz | ~30 kΩ | ≈ −128° |
| 1 kHz | ~13 kΩ | ≈ −148° |
| 2.8 kHz | ~10.5 kΩ | ≈ −166° |

**Behavior:** DC/low-freq the source sees the series ladder `R3+R1+R2 ≈ 104 kΩ`,
but `C1`/`C3` dominate below ~300 Hz → tens-to-hundreds of kΩ, mostly capacitive.
Above a few kHz the caps shunt away and it settles on **`R3 = 10 kΩ`** as the floor.

**Net effect:** a light, high-Ω load — insensitive to measurement point in-band
(the point of the RC-first reorder). Quote it as **≈10 kΩ resistive minimum,
tens of kΩ in the passband, not purely resistive below a few hundred Hz**.

### Topology & why the RC is first (Key Decision)
Signal path: **source → `R3`/`C3` (RC) → `R1`/`R2`+`C1`/`C2` (Sallen-Key) → AD8552 follower → `vout` (ADC).**

The RC stage is placed **before** the active stage — deliberately **not** after it —
for **impedance invariance**:
- A SAR track-and-hold ADC presents a **time-varying input impedance** (≈30 pF sample cap
  re-acquired each conversion at 200–500 kSPS). With the RC *after* the follower, the ADC
  loads through `R3`/`C3`, which can modulate the filter corner at the sample rate.
- Reordering makes the **AD8552 follower the last stage**, so the ADC only ever sees a low,
  stable source impedance — the T/H loading can't perturb the filter.
- The filter itself stays invariant too: the RC now terminates into the fixed Sallen-Key
  network (`R1` 47k), not into a time-varying ADC.
- **No response cost:** 2nd-order active × 1st-order RC is a linear cascade, so |H| is
  identical either order — this preserves the ~200 Hz corner and ~3rd-order rolloff.

### Corner / impedance choices
- Sallen-Key unity-gain stage: `fc = 1/(2π·√(R1·R2·C1·C2))` ≈ **228 Hz** with the values
  above; damping **Q ≈ 0.35** → two real, heavily damped poles, **no peaking** (good for
  suppression).
- RC stage: `f = 1/(2π·R3·C3)` ≈ **723 Hz** — sits *above* the active corner so it only
  tames the active stage's HF pass-through rather than re-drawing the filter corner.
  (Earlier `C3 = 100n` put this pole at 159 Hz *below* the Sallen-Key corner, which dragged
  the overall −3 dB down to ~136 Hz and shallowed the rolloff — that was the wrong-side
  placement; fixed here.)
- **R/C vs. AD8552:** at Q≈0.35 the loop is trivially stable vs. the AD8552's specs
  (GBP 1.5 MHz, 130 dB gain). Johnson noise of 2×47 kΩ ≈ 1.2 nV/√Hz sits well below the
  amp's 42 nV/√Hz — resistor values are noise-balanced; no change needed.

### Implementation notes
- Use 1% (or better) resistors — Q is set by the R/C ratios.
- Prefer **C0G/NPO** dielectric for the filter caps where the value allows (nF-range C0G
  is available); avoid X7R if accuracy/stability of the corner matters.
- Keep the feedback cap and output cap close to the op-amp.

## Key Decisions

- **RC stage placed before the active stage** — for impedance invariance vs. the SAR
  ADC's time-varying track-and-hold input. No response cost (linear cascade). See
  **Topology & why the RC is first** above.
- **Values locked** to common parts: 47k/47k/10k + 10n/22n/22n (4 distinct values).
  See **Final values** above.
- Full formal write-up: [DDD-001 — ADC LPF](decisions/DDD-001-adc-lpf.md).

## Files

- `ADC.SchDoc`
- `LPF.SchDoc`
- `LPF.asc` (LTspice sim source for these values; sim-verified via ltspice-mcp-bridge)

## Backlinks

- [Documentation Hub](Main.md)