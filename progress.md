# Progress - SV08 original toolhead + stock eddy on mainline Klipper

Branch: `sv08-original-sovol-eddy` from `upstream/master`
`29d4ec2d6d840ee839333306003a400da186d41d` (2026-11-10).

## Done

* Branch created from `upstream/master` (not `v0.13.0`, which lacks
  `descend_z`, `tap_threshold`, `max_sensor_hz` and eddy Z homing).
* `config/printer-sovol-sv08-2023.cfg` - full mainline config for the SV08
  with the original toolhead and the native `[probe_eddy_current]` /
  `ldc1612` driver (software I2C on `extra_mcu:PB6/PB7`).
* Registered that config in `test/klippy/printers.test`
  (`DICTIONARY stm32f103.dict extra_mcu=stm32f103.dict`).
* `SV08-MIGRATION.md` - host install, flash backups, Katapult, firmware,
  commissioning order, eddy calibration.
* `plan.md` - staged plan with gates.

## Verification actually run

| Check | Result |
| --- | --- |
| `scripts/check_whitespace.sh` | pass |
| `klippy/klippy.py --import-test` (Python 3.14) | pass |
| stm32f103 firmware + data dictionary built from this tree | pass (`out/klipper.dict`) |
| `scripts/test_klippy.py` on `printer-sovol-sv08-2023.cfg` (both MCUs) | pass |
| `G28 X` / `G28 Y` / moves / fans / LED through the homing override | pass (exit 0) |
| `G28` (all axes) | stops with "Must calibrate probe_eddy_current first" - expected for an uncalibrated eddy, proves the override template and probe wiring run |

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
* Branch not pushed to `origin` yet.

## Next

1. User supplies stock config bundle + `klippy.log`; reconcile the pin map.
2. Confirm board revision and Katapult route; flash backups first.
3. Commission per stage 6, then calibrate the eddy (stage 7).
