# DDD-005 — 12 V supply protection concept (central input protection + DRV8874 VM)

- Date / authoring session: 2026-10-04
- Status: **Proposal**. Architecture inputs are confirmed by the user (central input protection
  with reverse-polarity protection; vehicle has an alternator + electric starter; 24 V jump
  start is **not** a requirement). Part numbers/values below are proposals.
- Related: [DDD-004](DDD-004-drv8874-dac-vref-current-limit.md) (motor-driver current limit),
  [Power & Protection](../Power-Protection.md) (requirements list), [Motor Driver](../Motor-Driver.md)

## Change

1. All board power goes through **one central input protection stage** (`Input Protection.SchDoc`):
   bidirectional load-dump TVS at the connector → reverse-polarity blocking (ideal-diode /
   back-to-back N-FET) → protected rail `VBAT_P`. Optional OV cut-off.
2. **The DRV8874 VM is fed from `VBAT_P`.** It has **no local TVS**. Locally it only gets
   decoupling and bulk capacitance per TI.
3. The **DDD-003 output TVS (16 V standoff across OUT1–OUT2) is dropped** as load-dump
   protection. The motor outputs only get connector-ESD protection (see below).

## Why / what TI recommends (sources)

- **DRV8874 DS (SLVSF66A):** 0.1 µF low-ESR X7R + "sufficient bulk capacitance, VM-rated",
  both close to the pins. The bulk value is set by system test (factors include supply
  inductance, ripple and the braking method, §9.1). VM abs max **40 V**, recommended
  ≤ 37 V. OUT is clamped by the body diodes to −0.9 V … VM + 0.9 V, so output transients
  end up on VM. Integrated OCP, TSD, UVLO and CPUV.
- **SLVAFT0** (bulk sizing): `C_BULK > k·ΔI·T_PWM/ΔV`, k ≈ 3 for real ESR. Rule of thumb:
  1–4 µF/W.
- **SLVAF66** (system design): a few 100–330 µF + 1–2.2 µF ceramics. Rate capacitors at
  1.5–2× working voltage. TVS diodes "clamp below abs max", but "should be used in
  conjunction with other mitigation techniques and not be relied upon". **Avoid coasting
  a spinning motor** (it pumps VM); brake instead.
- **SLVA835:** reverse polarity must be blocked upstream (series diode or P/N-FET).
  Otherwise the H-bridge body diodes short GND to VM.
- **TIDA-01357** (TI automotive actuator reference design): TVS standoff chosen above the
  jump-start voltage, so the TVS is invisible in normal operation.
- **LM7481-Q1** (TI ideal-diode controller, back-to-back FETs, adjustable OV cut-off, 65 V):
  TI's standard pattern for reverse-polarity + overvoltage protection of a 12 V ECU input.

## Load-dump assumption (OPEN — decides TVS sizing)

The car has an alternator, so load dump applies. **Which ISO 16750-2 test applies depends
on the alternator:**

| Case | Test | 12 V parameters |
|------|------|-----------------|
| Alternator with integrated load-dump suppression (avalanche/Zener rectifier) | **Test B (suppressed)** | clamped to **US\* = 35 V**, Ri 0.5–4 Ω, td 40–400 ms, 10 pulses @ 1 min |
| Alternator without suppression | **Test A (unsuppressed)** | **US = 79–101 V**, Ri 0.5–4 Ω, td 40–400 ms, tr 10 ms, 10 pulses @ 1 min |

(Values: ISO 16750-2 via Littelfuse/Vishay load-dump app notes.)

**Proposed baseline: Test B.** Confirm the alternator type. If it turns out to be Test A,
the input TVS must handle roughly (101 V − 30 V)/0.5 Ω ≈ 140 A for up to 400 ms. One
SM8S cannot do that. You need paralleled SM8S, an SLD-class part, or an active surge
stopper.

## Proposed central input protection (no jump-start requirement)

| Element | Proposal | Rationale |
|---------|----------|-----------|
| Input TVS (connector side, before reverse FET) | **SM8S20CA** (bidirectional, DO-218AB, AEC-Q101): VWM 20 V, VBR 22.2–24.5 V, VC 32.4 V @ 204 A (10/1000 µs) | Off at 16 V USmax and at the 18 V / 60 min overvoltage test. Clamps Test B (35 V) and ISO 7637-2 pulses 2a/3b to ≤ ~32 V, which is below the DRV8874's 37 V rec. / 40 V abs. max. Bidirectional so it stays off at −14 V reverse battery and clamps pulse 1/3a. |
| Reverse-polarity protection | Ideal-diode controller + back-to-back N-FETs (e.g. **LM74810-Q1**), FET VDS ≥ 40 V (60 V preferred) | Blocks −14 V / 60 s reversed battery and negative pulses. Low drop. Reverse-current blocking also keeps motor regeneration off the harness. |
| OV cut-off (optional) | LM74810-Q1 OV threshold ≈ 20–22 V | Disconnects `VBAT_P` during load dump / regulator failure. Gives margin to 40 V. Allowed by ISO (functional class C for these tests). |
| Fuse / e-fuse | upstream fuse + `EFuse_12V` for the actuator branch | Short-circuit protection of the harness. |

Resulting `VBAT_P` worst case ≈ **32 V** without OV cut-off, ≤ ~22 V with it. **Every part
on `VBAT_P` must be rated ≥ 40 V** (DRV8874 40 V abs, PSU_5V buck, e-fuses, HSS,
capacitors ≥ 50 V).

> Test B check: the SM8S20CA starts conducting at ~22–24 V, so during Test B it carries
> roughly (35 V − ~28 V)/Ri ≈ up to ~14 A for up to 400 ms. That is well inside SM8S
> load-dump capability, but verify with the datasheet load-dump curve at the max
> ambient temperature (derates ~36 % at 150 °C case).

## DRV8874 VM (local)

| Element | Value | Basis |
|---------|-------|-------|
| C_VM1 | 0.1 µF X7R **≥ 50 V**, at pin 11 | DS Table 1 |
| C_VM2 ceramic | 1–2.2 µF X7R **≥ 50 V** (100 V helps with DC-bias derating) | SLVAF66 |
| C_BULK | **100–220 µF electrolytic/polymer, ≥ 50 V**, close to device, + spare footprint | 1–4 µF/W × ~50 W → 50–200 µF. SLVAFT0 eq.: 3 × 0.5 A × 50 µs / 0.3 V ≈ 250 µF (ΔI assumed; actuator inductance unknown). Final value by test. |
| Local TVS on VM | **none** (central protection) | — |

**Firmware (from TI SLVAF66):** stop with **brake** (EN = 0, low-side slow decay). Drop
nSLEEP only after the current has decayed. Ramp down before reversing. Example: ½·L·I²
at 1 mH / 4.5 A ≈ 10 mJ into 100 µF lifts 14 V → ~20 V. Harmless, but verify with the
real actuator inductance.

## Motor outputs (OUT1/OUT2) — connector ESD only

- **Not from TI; own recommendation:** the DRV8874 is rated only ±2 kV HBM, while
  automotive connector pins are typically tested to ISO 10605 (±8 kV contact).
- Small bidirectional TVS **OUTx → GND** with standoff ≥ max `VBAT_P` (so it never
  conducts in operation), low capacitance, e.g. SMAJ/SMBJ-class ≥ 22–24 V standoff
  (to select).
- **No large capacitors on OUTx.** TI: capacitance at the motor terminals causes current
  spikes on each PWM edge that can falsely trigger current regulation (DS §7.3.3.2).

## What 24 V jump start would additionally require (NOT a requirement — reference only)

ISO 16750-2:2012: **24 V for 60 s** at RT (class D min). ISO 16750-2:2023: **26 V for
60 s** at RT and Tmin (class C min).

1. **Input TVS must not conduct for 60 s at 26 V.** VWM ≥ 26 V → e.g. **SM8S26CA** (VBR
   28.9 V min, VC 42.1 V @ 157 A). The clamp then exceeds the DRV8874's 40 V abs max.
   **So the OV cut-off becomes mandatory** (LM74810-Q1, threshold ~28–30 V), or every
   downstream part must be ≥ 45–50 V rated. The DRV8874 is not.
2. **Back-to-back FETs ≥ 60 V VDS.** The TVS clamp (~42 V) plus pulse margin appears
   across them while cut off.
3. **Reverse jump (−24/−26 V):** the reverse-blocking FET and bidirectional TVS must
   withstand it. The TVS VBR must be > 26 V in both polarities.
4. **If the board must operate at 26 V** (no cut-off): PSU_5V buck ≥ 42 V (prefer 60 V)
   with thermal check; e-fuse/HSS rated ≥ 40 V and their loads checked at double
   voltage.
5. **Actuator at 26 V:** stall current roughly doubles (~7–8 A), which exceeds the
   DRV8874's 6 A peak / OCP min. ITRIP (DDD-004) limits it, but firmware should
   **inhibit actuation above ~18 V**. That needs a Ubatt ADC measurement.
6. **Capacitors ≥ 50 V** (already specified).

## Open / verify

- **Alternator: integrated load-dump suppression? (Test A vs Test B)**
- Confirm `VBAT_P` → `EFuse_12V` → DRV8874 VM topology in the schematics (not checked;
  Altium files untouched).
- Max ambient temperature at the ECU location → TVS derating.
- Choose the OUTx ESD TVS part. Decide whether to use the OV cut-off.
- `Input Protection.SchDoc` not cross-checked against this proposal.

## Backlinks

- [Power & Protection](../Power-Protection.md)
- [Motor Driver](../Motor-Driver.md)
- [Documentation Hub](../Main.md)
