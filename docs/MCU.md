# MCU

> Draft — pin/interface notes for the Stamp-S3A pseudo-module (ESP32-S3FN8).

## Role

Onboard processing for VTG-Ctrl: signal conditioning inputs (incl. turbo-speed
pulse), sensing interfaces, and control outputs. Module = **M5Stack Stamp-S3A**,
based on the **ESP32-S3FN8** (Xtensa LX7 dual-core @ 240 MHz, 8 MB SPI flash).

## Pins broken out on the Stamp-S3A

The module exposes **23 GPIOs** (2.54 mm / 1.27 mm headers):

`G0 G1 G2 G3 G4 G5 G6 G7 G8 G9 G10 G11 G12 G13 G14 G15 G39 G40 G41 G42 G43 G44 G46`

## Special / strapping pins — READ BEFORE ASSIGNING

The ESP32-S3 has four strapping pins: **GPIO0, GPIO3, GPIO45, GPIO46**. Their
voltage at reset sets boot mode. **Do not** route unconditioned external
dividers/pull-ups onto these or hold them at the wrong level at boot.

`G0` and `G46` are the boot-mode pins **on the module** (per Stamp-S3A docs):
- `G0` — pulled **up** by default; also the **module's physical button**.
- `G46` — internally pulled **down** by default. **Never pull `G46` high before
  boot** or the chip fails to start.

Rules of thumb for pin assignment on this module:
| Category | Pin(s) | Notes |
|-------|--------|-------|
| Strapping / boot | `G0`, `G3`, `G46` | avoid for signals; `G0` = module button, `G46` must stay low at boot |
| Strapping (not broken out here) | `G45` | on chip, not on this module's breakouts |
| Default USB-serial / console | `G43` (U0 TX), `G44` (U0 RX) | used for USB serial/programming; reserve |
| Module peripherals | RGB LED (WS2812B), boot button, LCD-FPC (8P/12P) on back | check schematic before freeing these GPs |
| Flash/PSRAM (chip-level) | GPIO26–32 | SPI1/SPI0 for external flash/PSRAM; on-chip flash here, but treat as reserved |

### Practical guidance
- Any **free, non-strapping GPIO** is fine for general inputs/outputs, interrupts,
  and the **PCNT (pulse counter)** — the ESP32-S3 GPIO matrix routes peripherals to
  any pin, so there is **no dedicated PCNT / timer pin**.
- For the **turbo-speed pulse input** (375 Hz–3.125 kHz, see
  [`Turbospeed.md`](Turbospeed.md)): a plain free GPIO interrupt or PCNT works; just
  avoid `G0/G3/G46` and don't hang an unconditioned 5 V divider on a strapping pin.
- Always check the Stamp-S3A **schematics** against the assignment before layout
  (module front/back, RGB LED, button, FPC).

## Files

- `MCU.SchDoc`
- `MCU.Harness`
- Module: M5Stack Stamp-S3A datasheet — https://docs.m5stack.com/en/core/Stamp-S3A

## Backlinks

- [Documentation Hub](Main.md)