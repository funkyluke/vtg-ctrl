# DDD-004 — DRV8874 current limit: firmware-set VREF via MCP4725, ITRIP above stall

- Date / authoring session: 2026-10-04
- Status: Decided
- Supersedes: [DDD-003](DDD-003-drv8874-vtg-actuator.md) (fixed ITRIP ≈ 2.8 A below stall,
  cycle-by-cycle mode). Driver choice (DRV8874PWPR, PH/EN) from DDD-003 is unchanged.

## Change

- **VREF is driven by an MCP4725A0T-E/CH** 12-bit I²C DAC (supplied from 3.3 V) instead
  of a fixed divider → ITRIP is set by firmware at runtime.
- **RIPROPI = 1.2 kΩ, 1 %** (was 1.5 kΩ).
- **IMODE = GND → Quad-Level 1: fixed off-time, automatic retry** (was 20 kΩ / cycle-by-cycle).
- **ITRIP is no longer set below stall.** It is a firmware-scheduled limit: high for
  breakaway, lower during travel/hold. Endstop/stall detection moves to firmware via the
  IPROPI ADC reading.

## Why

- **Limiting below stall (DDD-003) costs breakaway torque:** standstill current = stall
  current (3.7 A), so a 2.8 A ITRIP caps starting torque at ~76 % of stall — exactly when a
  sooted/stuck VTG mechanism needs full torque.
- **Cycle-by-cycle mode doesn't support 100 % duty** (DRV8874 §7.3.3.2.2): after a chop the
  bridge brakes until the next EN/PH edge. Since every start exceeds a sub-stall ITRIP, it
  forced permanent PWM and made nFAULT pulse on every start. Fixed off-time (tOFF = 25 µs)
  re-enables automatically and supports 100 % duty.
- **Adjustable VREF gives both:** full torque at breakaway, a tight limit during travel (a
  jam/endstop shows up as a current plateau at ITRIP), and a low thermally safe hold limit.
- **MCP4725 over PWM+RC:** true DC (no ripple/response trade-off, 6 µs settling) and the
  EEPROM power-on code gives a defined ITRIP before firmware runs. AEC-Q100 Grade 1,
  −40…+125 °C, SOT-23-6.

## Calculations

DRV8874: `ITRIP × AIPROPI = VVREF / RIPROPI`, AIPROPI = 450 µA/A (typ).
MCP4725: reference = its VDD = 3.3 V → `VOUT = code / 4096 × 3.3 V`.

- Sense gain: `450 µA/A × 1.2 kΩ` = **0.54 V/A**
- DAC resolution: 0.806 mV/LSB → **≈ 1.5 mA ITRIP per LSB**
- Full-scale (code 4095, 3.3 V) → 6.1 A — above OCP min (6 A); firmware must clamp (below)
- IIPROPI at 6 A = 2.7 mA (≤ 3 mA rec. max) ✓; VREF ≤ 3.6 V rec. max ✓

| Use | ITRIP | VREF | DAC code |
|-----|-------|------|----------|
| Hold at endstop | 1.0 A | 0.540 V | 670 |
| Travel limit (example) | 2.0 A | 1.080 V | 1341 |
| (old DDD-003 value) | 2.8 A | 1.512 V | 1877 |
| Nominal stall (reference) | 3.7 A | 1.998 V | 2480 |
| **Breakaway / backstop = EEPROM default** | **4.5 A** | **2.430 V** | **3016** |
| **Firmware max clamp** | **5.0 A** | **2.700 V** | **3351** |

General: `code = round(ITRIP × 0.54 V/A / 3.3 V × 4096)` = `round(ITRIP[A] × 670.3)`.
Travel limit = ~1.5 × measured worst-case running current (TBD on the actuator).

### ADC (IPROPI → ESP32-S3)

- `VIPROPI = 0.54 V/A × IOUT`; the internal IPROPI clamp limits it to ≈ VREF, so with the
  5 A clamp VIPROPI ≤ 2.7 V — inside the ESP32-S3 ADC range at 12 dB attenuation.
- Stall (3.7 A) reads ≈ 2.0 V; hold (1 A) ≈ 0.54 V.
- Use an **ADC1 pin (GPIO1–10)**. ADC2 is shared with Wi-Fi.
- IPROPI only senses low-side FET current. In PH/EN with PWM on EN, the off phase is
  low-side brake, so the reading is continuous (drive + decay).

### ITRIP accuracy (budget, not yet worst-case verified)

- AIPROPI error: ±5.5 % (2–4 A), ±6 % (1–2 A), ±7.5 % (0.4–1 A); fixed ±30 mA below 0.4 A.
- RIPROPI: ±1 %.
- MCP4725 offset/gain error: see datasheet. The 3.3 V rail tolerance goes straight into
  VREF, because the DAC reference is VDD.
- → expect roughly ±8–10 % on ITRIP. This is fine for a limit. The stall threshold is
  calibrated per actuator anyway.

### Thermal (DRV8874, RθJA = 36 °C/W, RDS(on) HS+LS ≈ 0.25–0.30 Ω hot)

| Current | P (RDS only) | ΔTJ ≈ |
|---------|-------------|-------|
| 1.0 A (hold) | 0.25–0.3 W | ~10 K — OK continuous |
| 2.8 A | 2.0–2.4 W | ~80 K |
| 3.7 A (stall) | 3.4–4.1 W | ~140 K → TSD |
| 4.5 A (breakaway) | 5.1–6.1 W | ~200 K → TSD in seconds |

→ The breakaway/high limit must be **time-limited by firmware** (≤ a few hundred ms, TBD),
then step down. Only the hold level is thermally continuous. TSD (auto-recovery) is the
last line of defence for the driver, not for the actuator motor. Check the actuator's
rated stall time.

## Firmware strategy

1. Wake with nSLEEP high (≥ 1 ms tWAKE). The DAC is already at the EEPROM backstop
   (4.5 A).
2. **Start:** VREF = breakaway level. Blank stall detection ~20–50 ms (startup current =
   stall current).
3. **Travel:** drop to the travel limit. **Endstop/jam = IPROPI ≥ ~90 % of the active
   limit, sustained 10–30 ms.**
4. **At endstop:** set the hold level (1 A), or stop / reduce duty. Ramp duty down before a
   direction reversal (avoids VM pump-up).
5. Never write a code > 3351 (5.0 A clamp). Read back the DAC register after each write.
   An I²C failure leaves the old limit active.
6. nFAULT now only reports real faults (OCP, UVLO, CPUV, TSD). There is no chopping
   indicator in fixed off-time mode.

## MCP4725 implementation notes

- I²C address 0x60 (A0 = GND) / 0x61 (A0 = VDD). SDA/SCL pull-ups to 3.3 V.
  Avoid strapping pins G0/G3/G46.
- VDD: 100 nF + the datasheet's recommended 10 µF bulk, close to the pin.
- **Program the EEPROM once** (code 3016 = 4.5 A backstop), not at runtime. An EEPROM
  write takes 25–50 ms and blocks other commands. Runtime updates use the fast-write
  (DAC register only) command.
- Send an **I²C General Call Reset after power-up**. The EEPROM value may not load if VDD
  ramps slower than 1 V/ms (DS §5.4.2).
- **VOUT → series ~1 kΩ → VREF pin with 100 nF to GND** at the DRV8874. VREF is
  high-impedance, and the series R keeps the DAC buffer away from a large capacitive load.
  τ = 100 µs.
- Alternative default: storing a power-down state (1 kΩ to GND) in EEPROM would boot
  with VREF ≈ 0 (no drive until firmware sets a limit). Not chosen, because nSLEEP's
  internal pulldown already keeps the driver off at boot.

## Other part values (carried over / corrected from DDD-003)

| Element | Value |
|---------|-------|
| PMODE | GND (PH/EN) |
| IMODE | **GND** (Quad-Level 1) — no RIMODE resistor needed |
| RIPROPI | **1.2 kΩ 1 %**, close to pin 6 |
| IPROPI → ADC | series ~1 kΩ + 10 nF at the ADC pin. Optional ≤ 10 nF directly at IPROPI only if false trips are seen (slows regulation, DS §7.3.3.2) |
| nFAULT pull-up | 10 kΩ to **3.3 V** (ESP32 not 5 V-tolerant) |
| VCP–VM | 100 nF, 16 V, X7R |
| CPH–CPL | 22 nF, **≥ 50 V**, X7R |
| VM bypass | 0.1 µF **≥ 50 V** X7R + bulk (value TBD per DS §9.1, ≥ 50 V rating) |

## Open / verify

- Conditions of the 3.7 A stall measurement (VM, temperature). Cold winding at 14.4 V can
  exceed 4 A → re-check that the breakaway level gives full torque.
- Actuator worst-case running current → travel-limit value.
- Max breakaway duration vs. driver thermal and actuator stall rating.
- Bulk-cap value; VM clamp (load-dump energy enters on VM, not on OUT; must stay < 40 V
  abs max). Output TVS only for ESD/harness transients; standoff vs. jump-start
  requirement still open (carried over from DDD-003). ISO 16750-2 12 V load dump:
  Test A unsuppressed 79–101 V / 0.5–4 Ω / 40–400 ms; Test B suppressed 35 V (DDD-003
  conflated these).
- **Harness:** `DRV8874.Harness` (`MotCtrl = PWM,DIR,nFAULT,nSLEEP`) lacks **IPROPI** and
  **SDA/SCL**. Must be updated in Altium (not changed by this doc).

## Files

- `DRV8874.SchDoc` — **not cross-checked** against this decision (Altium files untouched
  when this was written); verify MCP4725, RIPROPI 1.2 kΩ, IMODE = GND in the schematic
- `DRV8874.Harness`

## References

- DRV8874 datasheet SLVSF66A — Table 6 (IMODE), Eq. 3, §7.3.3, Rec. Op. Cond.
- MCP4725 datasheet DS20002039 — §5.4 POR/EEPROM, VDD as reference.
- TI SLVAFQ3 *Integrated Stall Detection for Brushed DC Motors*.

## Backlinks

- [Motor Driver](../Motor-Driver.md)
- [Documentation Hub](../Main.md)
