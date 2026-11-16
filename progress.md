# Progress - SV08 original toolhead + stock eddy on mainline Klipper

Branch: `sv08-original-sovol-eddy` from `upstream/master`
`29d4ec2d6d840ee839333306003a400da186d41d` (2026-11-10).

## Done

* Branch created from `upstream/master` (not `v0.13.0`, which lacks
  `descend_z`, `tap_threshold`, `max_sensor_hz` and eddy Z homing).
* `config/printer-sovol-sv08-2023.cfg` - full mainline config for the SV08
  with the original toolhead and the native `[probe_eddy_current]` /
  `ldc1612` driver (software I2C on `extra_mcu:PB6/PB7`).
* `test/klippy/sv08.test` - harness entry that loads the config with both
  MCU dictionaries.  It is deliberately *not* in `printers.test`: that
  suite drives every example config with `move.gcode`, whose first command
  is a full `G28`, and `G28 Z` needs an eddy calibration that must not ship
  inside an example config.  `SV08-MIGRATION.md` explains the trade-off.
* `SV08-MIGRATION.md` - host install, flash backups, Katapult, firmware,
  commissioning order, eddy calibration.
* `plan.md` - staged plan with gates.

## Verification actually run

| Check | Result |
| --- | --- |
| `scripts/check_whitespace.sh` | pass |
| `klippy/klippy.py --import-test` (Python 3.14) | pass |
| stm32f103 firmware + data dictionary built from this tree | pass (`out/klipper.dict`) |
| `scripts/test_klippy.py -d /tmp/kdicts test/klippy/sv08.test` | pass (exit 0, "All 1 test cases passed") |
| `test/klippy/printers.test` back to upstream state (SV08 entry removed) | `git diff` against the pre-change commit is empty |
| `G28 X` / `G28 Y` / moves / fans / LED through the homing override | pass (exit 0) |
| `G28 Z` alone | homes X and Y, parks the sensor over the bed, then stops with "Must calibrate probe_eddy_current first" - expected for an uncalibrated eddy, proves the override template and probe wiring run |
| `G28` (all axes) | same expected stop; no config or template errors |

**Correction (I2C).**  The config comment originally claimed Klipper does
not work around the STM32F1 I2C errata.  It does: `i2c_busy_errata()` in
`src/stm32/i2c.c` clears the F1 lock-up at bus init by bit-banging nine
SCL pulses with GPIO until the peripheral reports idle.  The real reason to
use software I2C is speed - `i2c_setup()` programs `CCR`/`TRISE` for a
fixed 100 kHz and never reads `rate`, so `i2c_speed` is ignored on an
`[i2c_bus]`, while `ldc1612` defaults to 400 kHz and needs three transfers
per sample at 400 Hz.  Comment fixed in the config; full rationale in
`SV08-MIGRATION.md`.

Toolchain note: no cross-compiler and no `sudo` here, so `gcc-arm-none-eabi`
was downloaded with `apt-get download` and unpacked into `/tmp/armtc/prefix`
(plus `libnewlib-dev` headers and a small `arm-none-eabi-gcc` wrapper adding
`-isystem`/`-L` paths).  Python deps in a venv at `/tmp/klippyvenv`
(cffi, jinja2, greenlet, pyserial).  Nothing from that setup is in the repo.

## Not done / open

* Pin map is transcribed from community configs, **not** verified against the
  user's stock `printer.cfg` or a board schematic.  Needs the stock config
  bundle and a `klippy.log`.
* `max_sensor_hz: 8426835` and `reg_drive_current: 22` are someone else's
  numbers; both must be re-derived on this machine.
* No print-start/end macros ported yet (stock Sovol macros are not
  compatible as-is; the community `sovol-macros.cfg` is an option).
* Firmware images have not been built for the real boards (no menuconfig
  `.config` committed on purpose - build settings are documented in
  `SV08-MIGRATION.md` stage 5).
* Branch pushed to `origin/sv08-original-sovol-eddy`
  (`92080aa4e`, `43c64f664`).  GitHub Actions reports `total_count: 0` for
  the fork, so CI has not actually run there - enable Actions on
  `johndahl/klipper` (Settings → Actions) if you want `sv08.test` validated
  by CI, or open a PR against upstream to get it.

## Next

1. User supplies stock config bundle + `klippy.log`; reconcile the pin map.
2. Confirm board revision and Katapult route; flash backups first.
3. Commission per stage 6, then calibrate the eddy (stage 7).
