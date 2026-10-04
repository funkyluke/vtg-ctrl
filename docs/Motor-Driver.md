# Motor Driver (DRV8874)

> Draft — current concept per [DDD-004](decisions/DDD-004-drv8874-dac-vref-current-limit.md).

## Role

Drives the electric **VTG actuator** (brushed DC, bidirectional, ~3.7 A stall) with a
**DRV8874PWPR** H-bridge in PH/EN mode. Position reference is via endstop/stall detection
(no position sensor assumed — see open points).

## Interface to MCU

| Signal | Dir | Notes |
|--------|-----|-------|
| PWM → EN/IN1 | out | PWM duty (100 % allowed; fixed off-time mode) |
| DIR → PH/IN2 | out | direction |
| nSLEEP | out | high = run; internal pulldown keeps driver off during MCU boot |
| nFAULT | in | open drain, 10 kΩ to 3.3 V; real faults only (OCP/UVLO/CPUV/TSD) |
| IPROPI | analog in | 0.54 V/A → ESP32-S3 **ADC1** pin (GPIO1–10) |
| SDA/SCL | I²C | MCP4725 DAC setting VREF (= ITRIP) |

## Key Decisions

- [DDD-004 — Current limit: VREF from MCP4725, RIPROPI 1.2 kΩ, ITRIP above stall](decisions/DDD-004-drv8874-dac-vref-current-limit.md) — **Decided** (current)
- [DDD-005 — 12 V supply protection (VM from central input protection, bulk cap, output ESD)](decisions/DDD-005-12v-supply-protection.md) — Proposal
- [DDD-003 — Endstop/stall detection via IPROPI, ITRIP below stall](decisions/DDD-003-drv8874-vtg-actuator.md) — Superseded by DDD-004

## Key values (summary — details/calcs in DDD-004)

- RIPROPI 1.2 kΩ 1 % → 0.54 V/A; ITRIP = VREF / 0.54 V/A
- MCP4725A0T-E/CH (3.3 V): ≈ 1.5 mA ITRIP per LSB; `code ≈ ITRIP[A] × 670`
- Levels: hold 1.0 A (code 670), breakaway/backstop 4.5 A (code 3016 = EEPROM default),
  firmware clamp 5.0 A (code 3351)
- IMODE = GND (fixed off-time, auto retry), PMODE = GND (PH/EN)

## Doc ↔ file disagreements (per AGENTS.md §1)

- `DRV8874.Harness` = `MotCtrl = PWM,DIR,nFAULT,nSLEEP` — **missing IPROPI and SDA/SCL**
  required by DDD-004.
- `DRV8874.SchDoc` not yet cross-checked against DDD-004.

## Open points

- Stall-current measurement conditions, worst-case running current, max breakaway time.
- VM supply/protection per [DDD-005](decisions/DDD-005-12v-supply-protection.md): VM from
  `VBAT_P` (central protection, no local TVS), 0.1 µF + 1–2.2 µF + 100–220 µF (≥ 50 V).
  Brake before sleep. Output ESD TVS part still to select.
- Confirm no position sensor / ripple counting is needed.

## Files

- `DRV8874.SchDoc`
- `DRV8874.Harness`

## Backlinks

- [Documentation Hub](Main.md)
- [MCU](MCU.md)
- [Power & Protection](Power-Protection.md)
