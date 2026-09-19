# Analog Input / ADC

> Draft — signal chain notes being captured as design progresses.

## Role

Analog acquisition chain: an 8-channel LPF bank (`LPF.SchDoc`, repeated `LPF0..LPF7`)
feeding the analog front-end / ADC, plus channel scaling and interface details.

## Active Filter — 250 Hz 2nd-Order Low-Pass (Decision)

### Chosen device: TLV9001

Selected over the TLV313 as the active-filter op-amp. Both are 1.8–5.5 V,
rail-to-rail in/out, ~60–65 µA, 1 MHz gain-bandwidth parts; the decision came down to:

- **Capacitive-load stability:** TLV9001 is specified for **500 pF capacitive load**
  with resistive open-loop output impedance, making it far easier to stabilize with
  the filter's feedback/output capacitors. TLV313 only specs driving ≤10 kΩ resistive
  loads and offers no capacitive-load guarantee — a risk in a Sallen-Key output stage.
- **Better input offset (±0.4 mV typ)** — the stage's DC error feeds the ADC directly.
- Equal power and bandwidth, so neither is a differentiator for this filter.

### Corner vs. GBW headroom

- Filter corner: **250 Hz**, 2nd order.
- GBW margin: 1 MHz ÷ 250 Hz ≈ **4000×** — op-amp bandwidth is a complete non-factor,
  valid through Butterworth (Q = 0.707) and up to higher-Q stages.

### Sallen-Key sizing template (unity-gain)

- Choose cap pair `C1` (upper) / `C2` (feedback): for a Butterworth response start with
  roughly **2:1** (C larger on the feedback side).
- Resistors sized from `fc = 1 / (2π·√(R1·R2·C1·C2))`.
- Target R in the **10 kΩ–100 kΩ** band to keep caps in the nF range.
  Balanced starting point (R1 = R2 = R, C2 = 2·C1): **22 nF / 11 nF** with R ≈ **47–68 kΩ**
  gives ~250 Hz Butterworth.

### Implementation notes

- Use 1% (or better) resistors — Q is set by the R/C ratios.
- Prefer **C0G/NPO** dielectric for the filter caps where the value allows (nF-range C0G
  is available); avoid X7R if accuracy/stability of the corner matters.
- Keep the feedback cap and output cap close to the op-amp.
- *(TODO: fill exact R/C component values once Q and preferred cap values are locked.)*

## Key Decisions

- See **Active Filter** above; add further decisions as they arise.

## Files

- `ADC.SchDoc`
- `LPF.SchDoc`

## Backlinks

- [Documentation Hub](Main.md)