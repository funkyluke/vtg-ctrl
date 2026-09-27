# VTG-Ctrl — Documentation Hub

Landing page for design decisions and documentation for the **VTG-Ctrl** board
(Altium project, `VTG-Ctrl.PrjPcb`).

> **Start here.** Use the links below to jump to a subsystem's page. Each page is
> a stub for now and will gain design-decision notes, calculations, and rationale
> as the work progresses.

## Project Overview

- **Project:** VTG-Ctrl (VTC controller board)
- **Toolchain:** Altium Designer (`VTG-Ctrl.PrjPcb`)
- **Role in system:** TBD — document the board's place in the larger system here.

## Subsystems

| Page | Schematics | Status |
|------|-----------|--------|
| [MCU](MCU.md) | `MCU.SchDoc` | Stub |
| [Power & Protection](Power-Protection.md) | `Input Protection.SchDoc`, `PSU_5V.SchDoc`, `EFuse_5V.SchDoc`, `EFuse_12V.SchDoc` | Stub |
| [High-Side Switch](HSS.md) | `HSS.SchDoc` | Stub |
| [Motor Driver](Motor-Driver.md) | `DRV8874.SchDoc` | Stub |
| [Analog Input / ADC](Analog-ADC.md) | `ADC.SchDoc`, `LPF.SchDoc` | Stub |
| [Thermocouple Interface](Thermocouple.md) | `Max31856.SchDoc` | Stub |
| [DTM13](DTM13.md) | `DTM13.SchDoc` | Stub |
| [Turbo Speed](Turbospeed.md) | `Turbospeed.SchDoc` | Draft |

## Design Decisions Log

Individual decisions get their own pages under `decisions/`. Convention: one file
per decision, named `decisions/DDD-NNN-page.md`, linked from the relevant subsystem
page and from below.

- [DDD-001 — ADC LPF: values + RC-before-active topology](decisions/DDD-001-adc-lpf.md) — Decided
- [DDD-002 — Turbo-speed input conditioning](decisions/DDD-002-turbospeed-input.md) — Decided
- [DDD-003 — DRV8874 VTG-actuator drive: endstop/stall detection via IPROPI](decisions/DDD-003-drv8874-vtg-actuator.md) — Decided

## Coordination

- For agents/sessions working on this repo: read [AGENTS.md](../AGENTS.md) first
  (coordination contract, binary-file ownership, decision log, handoff ritual).

## Getting Started / Conventions

- Docs live in `docs/`, one markdown file per subsystem or decision.
- Cross-link pages freely (these are plain relative links).
- Mark stubs with `> Stub` and fill in as designs firm up.