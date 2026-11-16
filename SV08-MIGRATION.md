# SV08 (original toolhead) → mainline Klipper, with the stock Sovol eddy

Working branch: `sv08-original-sovol-eddy` (based on `upstream/master`
`29d4ec2d6d840ee839333306003a400da186d41d`, 2026-11-10).

Companion config in this tree: `config/printer-sovol-sv08-2023.cfg`.

## Read this first

* This procedure is adapted from the community guide
  [Rappetor/Sovol-SV08-Mainline](https://github.com/Rappetor/Sovol-SV08-Mainline)
  and the eddy config in
  [asnajder/sv08-config](https://github.com/asnajder/sv08-config).  Those are
  third-party sources; nothing here has been verified against this printer yet.
  Treat every pin, address and image name as "confirm on the machine".
* Two MCUs are involved.  Swapping their serials in `printer.cfg` drives the
  hotend heater from the wrong MCU at full power.  Always use the fixed
  `/dev/serial/by-id/usb-Klipper_stm32f103xe_...` names, never `ttyACMx`.
* Do not skip the flash backups in step 3.  Without them there is no route
  back to the stock firmware.
* The eddy probe must be calibrated before it can be used for Z (step 8).
  Until then, do not let any macro home Z.

## Stage 0 - what this checkout gives you

```
git remote -v                 # origin = your fork, upstream = Klipper3d
git log -1 --oneline          # 29d4ec2d6 ...
```

Why `master` and not `v0.13.0`: the release has no `descend_z`, no
`tap_threshold`, no `max_sensor_hz`, and its `docs/Eddy_Probe.md` states an
eddy probe cannot be used for Z homing.  On `master` the driver takes
`max_sensor_hz` as a config option, so the "patch `ldc1612.py` line 94"
workaround that older SV08 configs ask for is **not** needed - do not apply it.

## Stage 1 - host media

You have a spare eMMC, so keep the stock eMMC as-is and build the new system
on the spare (or on an SD card) - that keeps a known-good fallback.

1. Write the BTT CB1 image to the spare media (Makerbase eMMC adapter, or
   SD → eMMC copy as in the community guide).
2. Copy `sun50i-h616-sovol-emmc.dtb` into `/dtb/allwinner/` on the BOOT
   partition.
3. Edit `BoardEnv.txt` (keep everything below the `#--------#` separator):

   ```ini
   bootlogo=false
   overlay_prefix=sun50i-h616
   fdtfile=sun50i-h616-sovol-emmc
   console=display
   overlays=uart3
   overlays=ws2812
   overlays=spidev1_1
   ```

   `fdtfile=sun50i-h616-biqu-sd` forces SD-only boot if the eMMC keeps
   stealing priority.
4. Set Wi-Fi/hostname in `system.cfg` (escape `$` in the password as `\$`).
5. Boot, SSH in (default user/pass on the CB1 image is `biqu`/`biqu`).
6. Debian 11 mirrors are dead; point `/etc/apt/sources.list` at
   `archive.debian.org` with `[check-valid-until=no]`, then
   `sudo apt update && sudo apt upgrade`.
7. Install KIAUH, then Klipper + Moonraker + Mainsail.  In KIAUH, select
   "custom repository" and point Klipper at this fork and at branch
   `sv08-original-sovol-eddy`, so the host and the firmware come from the
   same tree.

Gate: the new host boots, is reachable over the network, and no MCU firmware
has been touched yet.

## Stage 2 - stock configuration backup

From the stock web UI (Machine → select all → download) or over SFTP:

* `printer.cfg` plus every included file, and `saved_variables.cfg`
  (contains the calibration the stock machine relies on).
* the `klippy.log` of a completed print,
* the stock `[mcu]` / `[mcu extra_mcu]` serial strings.

## Stage 3 - MCU flash backup (do this before any flashing)

Printer **powered off** while the ST-Link is connected; the MCU is powered by
the programmer.

Wiring, SV08 header → ST-Link: `3V3 → 3.3V`, `IO → SWDIO`, `CK → SWCLK`,
`G → GND`.

In STM32CubeProgrammer: connect, **Read all**, **Save as…**
`mainboard_original_firmware.bin` / `toolhead_original_firmware.bin`.

Check the sizes before trusting them: **≥ 512 KiB** mainboard,
**≥ 128 KiB** toolhead.  A ~1 KiB file means the read failed and there is no
way back.

## Stage 4 - Katapult bootloader on both MCUs

```bash
cd ~ && git clone https://github.com/Arksine/katapult
pip3 install pyserial
cd ~/katapult && make menuconfig      # STM32F103, USB serial interface
make clean && make
```

Take `~/katapult/out/katapult.bin` to the PC.  In STM32CubeProgrammer:
**Erasing & Programming → Full chip erase** (manual, not automatic), then
**Memory & File editing → Open file → katapult.bin → Download** (base address
`0x80000000`).  Repeat for the other MCU.

The printer will boot silently afterwards (no fans, no LED, no splash) - that
is expected, not a brick.

## Stage 5 - Klipper firmware on both MCUs

Identify the MCUs first.  After Katapult they enumerate as
`usb-Katapult_stm32f103xe_...`.  Unplug the toolhead USB connector, run
`ls /dev/serial/by-id/` again, and the serial that disappears is the
toolhead - write both down before touching anything.

Build settings for both MCUs (menuconfig):

* Enable support for STM32 → STM32F1 → STM32F103, 72 MHz crystal,
  USB (not serial/CAN),
* **Bootloader offset: 8 KiB** (required because Katapult is installed),
* optional "Additional GPIO pins to set at startup": `PA1,PA3` on the
  mainboard (MCU fan, light), `PA6` on the toolhead (hotend fan) - without
  this the printer boots with the fan off.

```bash
cd ~/klipper
make clean && make menuconfig && make
make flash FLASH_DEVICE=/dev/serial/by-id/usb-Katapult_stm32f103xe_<serial>
```

Re-run menuconfig for the second MCU with its own settings, then flash it the
same way.  Record `sha256sum out/klipper.bin` for each build.

Gate: `ls /dev/serial/by-id/` shows two `usb-Klipper_stm32f103xe_...` entries
and Klipper starts with both MCUs connected.

## Stage 6 - commissioning order

Start from `config/printer-sovol-sv08-2023.cfg` in this tree, copy it to
`~/printer_data/config/printer.cfg`, and fill in the two serials.  Then, in
this order, one check at a time:

1. `GET_MCU_CONFIG` / `STATUS` - both MCUs respond, no "Communication timeout".
2. Temperatures read plausibly at room temperature (bed ~25 °C, hotend
   ~25 °C).  A rail-rail reading means the thermistor table or pullup is wrong.
3. `SET_STEPPER_ENABLE` each axis, mark the shaft, `SET_KINEMATIC_POSITION`
   and jog: confirm direction and that no Z motor tilts the gantry the wrong
   way.
4. Fans: `M106`, `M107`, `TEST_FANS` if available.  Heaters: short pulses via
   `M104`/`M140` with the bed/hotend cold, confirm which element answers.
5. Eddy comms: `QUERY_PROBE`-style errors should be "Must calibrate
   probe_eddy_current first", **not** an I²C error.  An I²C failure means the
   SCL/SDA pins or the toolhead wiring are wrong - fix that before anything
   else.
6. Calibration (stage 7).
7. `G28` (X/Y sensorless, Z via the probe), `QUAD_GANTRY_LEVEL`, repeat
   `G28 Z`, check repeatability.

## Stage 7 - eddy calibration (native driver)

Follow `docs/Eddy_Probe.md`.  Because Z homing uses the probe, break the
chicken-and-egg exactly as the doc describes:

1. Keep `[force_move] enable_force_move: True` in the config (it is present
   in the shipped file; remove it after the first successful calibration).
2. Park the toolhead by hand ~20 mm above the bed centre, then
   `LDC_CALIBRATE_DRIVE_CURRENT CHIP=my_eddy_probe` and `SAVE_CONFIG`.
3. Home X and Y only (`G28 X Y`), move over the centre, then
   `SET_STEPPER_ENABLE STEPPER=stepper_z`, `SET_KINEMATIC_POSITION Z=25`,
   `PROBE_EDDY_CURRENT_CALIBRATE CHIP=my_eddy_probe`, do the paper test,
   `ACCEPT`, then `SAVE_CONFIG`.
4. Verify `x_offset` / `y_offset` with `Probe_Calibrate.md`; if they change,
   re-run `PROBE_EDDY_CURRENT_CALIBRATE`.
5. Check the reported "Total frequency range" against `max_sensor_hz` in the
   config; the driver warns if the value must be raised.
6. Calibrate at the temperature you print at - eddy sensors drift with heat.

Tap probing stays disabled (`#tap_threshold`) until
`PROBE_EDDY_CURRENT_TAP_CALIBRATE` has been run and `position_min` in
`[stepper_z]` has been reviewed.

## Stage 8 - printing

`BED_MESH_CALIBRATE` (scan mode) only after reliable single probes and a
clean `QUAD_GANTRY_LEVEL`.  Re-check `mesh_min`/`mesh_max` against the actual
probe offsets before a full-bed mesh, otherwise the first scan runs into the
rails.

## Why the eddy sits on software I2C

The toolhead header exposes PB6/PB7, which on the F103 *are* the hardware
I2C1 pins, so hardware I2C is physically available and is not avoided for
wiring reasons.  It is avoided for speed: Klipper's STM32 hardware I2C
driver ignores the requested rate - `i2c_setup()` in `src/stm32/i2c.c`
programs `CCR`/`TRISE` for a fixed 100 kHz and never reads `rate`, so
`i2c_speed` has no effect on an `[i2c_bus]`.  The `ldc1612` driver defaults
to 400 kHz and performs three transfers per sample at its 400 Hz sample
rate, which does not fit comfortably in a 100 kHz budget.  Software I2C
(`i2c_software_scl_pin` / `i2c_software_sda_pin`) does honour `i2c_speed`,
so the bus runs at 400 kHz.

The F1 I2C lock-up errata is *not* a reason to avoid hardware I2C here:
`src/stm32/i2c.c` works around it (`i2c_busy_errata()`, applied at bus init
on F1 parts) by clocking nine SCL pulses with GPIO until the peripheral
reports idle.  It only matters if those pins are shared with something else.

## Reproducing the checks that were run in this repo

The config has its own harness file, `test/klippy/sv08.test`, and CI runs
every `test/klippy/*.test` (see `scripts/ci-build.sh`).  It is deliberately
*not* listed in `printers.test`: that suite drives every example config
with `move.gcode`, which starts with a full `G28`, and `G28 Z` on this
machine uses the eddy probe - which refuses to report a position until
`PROBE_EDDY_CURRENT_CALIBRATE` has been run on the real machine.  Shipping
a calibration in the example config would be worse (wrong Z on every
machine that uses it), so `sv08.test` homes X and Y sensorlessly and sets Z
with `SET_KINEMATIC_POSITION` instead.  Upstream does the same kind of
thing the other way round: `config/sample-cartographer-v3.cfg` is not in
`printers.test` either, and eddy is covered by `test/klippy/eddy.test`
against a fixture config (`test/klippy/eddy.cfg`) that does carry a
`calibrate =` block.

```bash
# 0. host python deps for the harness (cffi, jinja2, greenlet, pyserial)
python3 -m venv /tmp/kenv && /tmp/kenv/bin/pip install cffi jinja2 \
    greenlet pyserial

# 1. build the stm32f103 data dictionary (needs gcc-arm-none-eabi)
cp test/configs/stm32f103.config .config && make olddefconfig && make
mkdir -p /tmp/kdicts && cp out/klipper.dict /tmp/kdicts/stm32f103.dict

# 2. run the config through klippy's own test harness
/tmp/kenv/bin/python scripts/test_klippy.py -d /tmp/kdicts \
    test/klippy/sv08.test

# 3. whitespace + import checks used by CI
./scripts/check_whitespace.sh
python3 klippy/klippy.py --import-test
```

Known limitation: this repository has no cross-compiler installed by default,
and `sudo` is not available, so the dictionary above has to be built from a
toolchain the user installs (or from CI).
