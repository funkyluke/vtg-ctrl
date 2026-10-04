# Power & Protection

> Draft — requirements + concept. Concept decision: [DDD-005](decisions/DDD-005-12v-supply-protection.md) (Proposal).

## Role

Input protection, main PSU, and e-fuse rails for VTG-Ctrl on the car's **12 V system**
(alternator + electric starter, lead/Li battery). All board power enters through **one
central input protection stage** with reverse-polarity protection. It produces the
protected rail `VBAT_P`, which feeds `PSU_5V`, the e-fuses, the HSS and the DRV8874 VM.

## Key Decisions

- [DDD-005 — 12 V supply protection concept (central input protection + DRV8874 VM)](decisions/DDD-005-12v-supply-protection.md) — Proposal
- Related: [DDD-004 — DRV8874 current limit / bulk cap](decisions/DDD-004-drv8874-dac-vref-current-limit.md), [Motor Driver](Motor-Driver.md)
- 5 V logic I/O ESD: PESD5V0S2BT per [DDD-002](decisions/DDD-002-turbospeed-input.md) (signal lines only, not power)

## Design rules derived from the requirements

1. **Everything on `VBAT_P` is rated ≥ 40 V** (worst-case clamp ≈ 32 V with SM8S20CA), and
   capacitors are rated ≥ 50 V.
2. **Every connector pin** survives a short to GND and to battery (16 V, ideally 18 V)
   indefinitely, plus ISO 10605 ESD.
3. **No part conducts at 16 V continuous / 18 V for 60 min**, so the TVS standoff must
   be ≥ 18–20 V.
4. **Defined reset behaviour** below the MCU/PSU minimum. Clean restart after cranking.
5. **Inductive loads** (actuator, HSS loads) return their energy in a controlled way
   (brake / clamp) and don't pump the rail.

## Requirements — automotive 12 V ECU (general)

Sources: ISO 16750-2:2012 / :2023 (electrical loads), ISO 7637-2 (conducted transients
on supply lines), ISO 7637-3 (coupling to signal lines), ISO 10605 (ESD), CISPR 25
(emissions), ISO 11452 (immunity). Severity levels and functional classes have to be
agreed per project. The values here are the standard defaults; a proposed VTG-Ctrl
target is given where useful.
Functional status classes (ISO 16750-1): **A** = all functions as designed;
**B** = deviations allowed during the test, then automatic recovery; **C** = function
lost during the test, automatic recovery afterwards; **D** = function lost, recovers
after reset; **E** = permanently damaged (repair needed).

### ISO 16750-2 — supply voltage

| ID | Test | 12 V parameters | Typical requirement | VTG-Ctrl target / note |
|----|------|-----------------|---------------------|------------------------|
| PP-01 | DC supply range | Code A 6–16 V · B 8–16 V · C 9–16 V · D 10.5–16 V | Class A in range | **Proposal: Code B (8–16 V) class A** |
| PP-02 | Long-term overvoltage (regulator failure) | **18 V, 60 min**, at Tmax − 20 °C | min. class C | TVS must stay off → VWM ≥ 18–20 V |
| PP-03 | Jump start | 2012: **24 V / 60 s** RT · 2023: **26 V / 60 s** RT + Tmin | 2012 class D, 2023 class C | **Not required.** What it would take: see DDD-005 |
| PP-04 | Transient overvoltage (2023) | 18 V, 400 ms, 5× | class B | — |
| PP-05 | Superimposed AC | 2012: 1/4/2 Vpp, 50 Hz–25 kHz · 2023: up to 6 Vpp, 10 Hz–200 kHz | class A | Alternator ripple. Affects ADC/IPROPI readings. Filter analog references. |
| PP-06 | Slow decrease/increase | 0.5 V/min down to 0 V and back | class A in range, defined behaviour outside | Brown-out / reset thresholds |
| PP-07 | Drops / micro-interruptions | short drops to 0 V | class B/C | Hold-up on 3.3 V/5 V or clean reset |
| PP-08 | Reset behaviour at voltage drop | stepped dips | defined reset, no latch-up | MCU supervisor / brown-out |
| PP-09 | **Starting profile (cranking)** | US1 = **8 / 4.5 / 3 / 6 V** (levels I–IV: warm / cold-good / cold-aged / RT) | per level, typically class A–C | **Electric starter present.** DRV8874 UVLO 4.35–4.6 V → actuator off below that. MCU should survive level II (4.5 V) via buck dropout, or reset cleanly. |
| PP-10 | **Load dump** | **Test A** unsuppressed: 79–101 V, Ri 0.5–4 Ω, td 40–400 ms, 10× @ 1 min · **Test B** suppressed: clamped **35 V** | min. class C | **Alternator present → applies.** Test A vs B depends on the alternator. **OPEN**, see DDD-005 |
| PP-11 | Reversed voltage | **−14 V, 60 s** | class A/C after test, no damage | Central reverse-polarity protection (confirmed) |
| PP-12 | Ground offset | ±1 V between ECU ground and other grounds | class A | Matters for sensor/signal grounds (ADC, turbo speed) |
| PP-13 | Open circuit | single / multiple line interruption, incl. **loss of ground** | class C | No back-powering through I/O. Watch the motor/HSS return paths. |
| PP-14 | Short circuit | each in/output to **GND and to Ubatt**, 60 s | class C (no damage) | DRV8874 OCP + e-fuse/HSS. MCU inputs need series R + clamp. |

### ISO 7637-2 — supply-line transients (12 V)

Levels depend on edition and severity class. Typical levels are given here; confirm
against the edition that applies.

| ID | Pulse | Origin | Typical 12 V level | Protection |
|----|-------|--------|--------------------|------------|
| PP-20 | 1 | disconnect of parallel inductive load | **−75 … −150 V**, Ri 10 Ω, 2 ms | bidirectional input TVS + reverse FET |
| PP-21 | 2a | sudden interruption of series current (harness inductance) | **+37 … +112 V**, Ri 2 Ω, 50 µs | input TVS |
| PP-22 | 2b | DC motors acting as generators after ignition off | 10 V, Ri 0–0.05 Ω, 0.2–2 s | TVS / rail rating |
| PP-23 | 3a / 3b | switching transients (fast bursts) | **−112 … −220 V / +75 … +150 V**, Ri 50 Ω, 0.1 µs | TVS + ceramic decoupling at the connector |

(TI TLV1805-Q1 ISO report SNOAA13 shows an example test set: P1 −100 V/10 Ω/2 ms,
2a +37…50 V/2 Ω/50 µs, 3a/3b ±100 V/50 Ω/100 ns.)

### Other

| ID | Requirement | Note |
|----|-------------|------|
| PP-30 | **ESD ISO 10605** on all connector pins (typ. ±8 kV contact / ±15 kV air, powered + unpowered) | Logic I/O: PESD5V0S2BT (DDD-002). Motor outputs: small TVS (DDD-005). |
| PP-31 | **ISO 7637-3** transient coupling onto signal lines | Series R + clamp on MCU inputs |
| PP-32 | **CISPR 25** conducted/radiated emissions | DRV8874 spread-spectrum, input π-filter, PWM edge control |
| PP-33 | **ISO 11452** immunity (BCI, radiated) | Filtering of analog inputs, ground concept |
| PP-34 | Quiescent current in sleep | DRV8874 nSLEEP < 1 µA. Define a board budget. |
| PP-35 | Fusing / harness protection | Upstream fuse sized to the harness. `EFuse_12V` for the actuator branch. |

## Concept summary (detail in DDD-005)

`connector → SM8S20CA (bidir. load-dump TVS) → LM74810-Q1-type ideal diode + back-to-back
N-FETs (reverse polarity, optional OV cut-off ≈ 20–22 V) → VBAT_P → PSU_5V / EFuse_12V /
HSS / DRV8874 VM (0.1 µF + 1–2.2 µF + 100–220 µF, all ≥ 50 V)`

## Open points

- **Alternator load-dump suppression → ISO 16750-2 Test A or Test B** (sizes the input TVS).
- Supply code (proposal: Code B), cranking level to survive, functional classes per test.
- Max ambient temperature at the mounting location (TVS derating).
- Schematics (`Input Protection`, `PSU_5V`, `EFuse_5V`, `EFuse_12V`) **not cross-checked**
  against this doc. Altium files untouched.

## Files

- `Input Protection.SchDoc`
- `PSU_5V.SchDoc`
- `EFuse_5V.SchDoc`
- `EFuse_12V.SchDoc`

## Backlinks

- [Documentation Hub](Main.md)
- [Motor Driver](Motor-Driver.md)
