# Turbo Speed Input

> Draft — input-conditioning spec and interface notes.

## Sensor

- **Unit:** Jaquet Sensor+Electronics turbo-speed sensor (similar to VAG turbo-speed
  sensor `05A927321F`).
- **Function:** outputs a 5 V square/pulse signal at **1/8 of the measured blade
  frequency**.

## Signal & timing math

Compressor wheel has **6 blades**.

| Quantity | Relation | Value |
|----------|----------|-------|
| Output pulse frequency | `f_out = f_blade / 8` | e.g. 375 Hz @ min blade freq |
| Shaft (compressor) speed | `RPM = f_blade / 6 × 60` | 3 kHz blade ⇒ 30,000 RPM |
| **Maximum speed** | — | **250,000 RPM** |
| Max blade frequency | `f_blade = RPM/60 × 6` | 25 kHz @ 250k RPM |
| Max output pulse freq | 25 kHz ÷ 8 | **3.125 kHz** |
| Minimum blade frequency | datasheet | **3 kHz** |
| Minimum output pulse freq | 3 kHz ÷ 8 | **375 Hz** |

> These are the sizing inputs for the conditioning stage and for MCU timer input
> (frequency counting / pulse measurement).

## Electrical interface (datasheet)

| Parameter | Value |
|-----------|-------|
| Supply | **5 V ± 10 %** |
| Max supply current | **10 mA** |
| Signal | 5 V logic square/pulse, 1/8 blade freq |

## Conditioning requirements (decided)

- Level-shift the 5 V pulse to the ~3.3 V MCU domain via a **SN74LVC1G17** Schmitt
  buffer (VCC = 3.3 V), with a **PESD5V0S2BT** connector TVS (SOT-23, ±30 kV IEC-4-2,
  AEC-Q101), a **10 kΩ** series limiter, **BAT54S** dual clamp (→ 3V3 and → GND),
  and **470 pF** RC to GND (~34 kHz corner).
- The PESD5V0S2BT is the standard ESD/TVS for **all 5 V I/Os** on this design.
- Full rationale and values: [DDD-002 — Turbo-speed input conditioning](decisions/DDD-002-turbospeed-input.md)

## Files

- `Turbospeed.SchDoc`

## Backlinks

- [Documentation Hub](Main.md)