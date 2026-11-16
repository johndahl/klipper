# SV08 (original toolhead + stock Sovol eddy) → mainline Klipper

Working branch: `sv08-original-sovol-eddy`, based on `upstream/master`
`29d4ec2d6d840ee839333306003a400da186d41d` (2026-11-10).

This is a **full printer migration**, not just adding an eddy config: host
install, firmware on both MCUs, then the eddy probe. The community procedure
(Rappetor/Sovol-SV08-Mainline, asnajder/sv08-config) is the reference route.

## Known constraints

- Recovery hardware available: **spare eMMC** and **an STM32 programmer
  (ST-Link class)**. Stock wiring is **unmodified** (original toolhead,
  original Sovol eddy sensor on the toolhead).
- `v0.13.0` is not usable for this sensor: it has no `descend_z`,
  `tap_threshold`, `max_sensor_hz`, and its docs state an eddy probe cannot be
  used for Z homing. Everything below assumes `upstream/master`.

## Gates (each must pass before the next stage)

1. **Backup** — stock config bundle + `klippy.log` off the printer; full flash
   reads of both MCUs (≥512 KiB mainboard, ≥128 KiB toolhead) verified.
2. **Host** — new mainline install boots and is reachable; stock media still
   bootable. No motion yet.
3. **Config** — one coherent `printer.cfg` (see
   `config/printer-sovol-sv08-2023.cfg`) with no leftover Sovol homing/probe
   macros; validated by Klipper's own config test.
4. **Firmware** — both MCU images built from the pinned commit, checksums
   recorded, Katapult route confirmed for the actual board revision.
5. **Commission** — comms → temps → motors → heaters/fans → eddy comms →
   drive-current + Z calibration → homing/GFL.
6. **Print** — meshes and print-start macros only after reliable probing.
   Tap is optional and last.

## Outstanding input from user

- Complete stock configuration bundle and a recent `klippy.log` (needed to
  confirm pin map, thermistor tables, travel limits, MCU serials).
- Board/toolhead revision photos if the pin map in the config disagrees with
  the stock `printer.cfg`.

## 1. Inventory and back up the stock printer

Collect before changing anything:

- Complete configuration download, including included files and saved calibration.
- Recent `klippy.log`, current Sovol version, and MCU identities.
- Host storage and board revisions.
- Details of installed screen, camera, and other accessories.

Prepare __two separate recovery paths__:

1. Host OS/configuration recovery.
2. Mainboard and toolhead firmware recovery.

The migration guide includes stock MCU firmware backup before bootloader replacement. We should complete and verify those backups before flashing. ([raw.githubusercontent.com](https://raw.githubusercontent.com/Rappetor/Sovol-SV08-Mainline/main/README.md))

__Gate:__ configuration and recoverable firmware backups exist outside the printer.

### 2. Prepare your fork from upstream master

During implementation:

- Fetch upstream and create `sv08-original-sovol-eddy` from the latest `upstream/master`.

- Record the exact commit, so the migration is reproducible rather than following a moving target.

- Preserve the existing release-based branch.

- Create:

  - `/home/john/klipper/plan.md`
  - `/home/john/klipper/progress.md`

- Resolve the required `build-test-repair` skill before code changes; it was not found during the earlier inspection.

__Default approach:__ native Klipper eddy support, with no Eddy-NG dependency or speculative driver patches.

### 3. Prepare a separate mainline host installation

Plan a clean installation rather than updating Sovol’s existing environment in place.

- Choose boot media after checking your hardware and recovery options.
- Prefer separate media where practical, leaving the original stock installation intact.
- Select a compatible host image and required SV08 board configuration.
- Install Klipper, Moonraker, and Mainsail.
- Point Klipper at your fork and the selected branch.
- Configure networking, services, and update management explicitly.

The SV08 migration project documents both eMMC and SD operation using a CB1-based installation. We will verify the appropriate image and boot configuration before writing any media. ([raw.githubusercontent.com](https://raw.githubusercontent.com/Rappetor/Sovol-SV08-Mainline/main/README.md))

__Gate:__ the new host boots and is accessible; stock recovery remains available. Printer motion is not yet enabled.

### 4. Prepare the original-hardware printer configuration

Use your working stock configuration as hardware evidence, then adapt it to current mainline interfaces.

Verify:

- Motor pins, directions, currents, and four-Z mapping.
- Heater and thermistor settings.
- Fans, endstops, filament sensor, and accelerometer.
- Travel limits, gantry-leveling coordinates, and mesh boundaries.
- MCU connections and actual device identifiers.
- Compatibility requirements for your screen and accessories.

For the Sovol eddy sensor, configure native `probe_eddy_current`. Published original-toolhead configurations use software I²C on toolhead PB6/PB7 and offsets around −16.43/+10.22 mm; those remain __reference values to verify__, not blindly copy. ([raw.githubusercontent.com](https://raw.githubusercontent.com/asnajder/sv08-config/main/config/options/probe/sovol-eddy.cfg))

Do not reuse another printer’s calibration tables, drive current, or tap threshold.

__Gate:__ one coherent configuration with no conflicting Sovol homing, probe, or automatic-Z macros.

### 5. Build and validate both MCU firmware images

- Confirm MCU types, clock settings, communication interfaces, and bootloader offsets.
- Select the flashing/bootloader route based on the actual boards.
- Build mainboard and toolhead firmware from the same selected Klipper revision.
- Record build settings and firmware checksums.
- Run applicable repository build, import, configuration, and regression tests.
- Add focused tests if source changes become necessary.

The migration guide uses MCU backups, Katapult installation, and subsequent Klipper flashing. We will verify that route against your board revisions before adopting it. ([raw.githubusercontent.com](https://raw.githubusercontent.com/Rappetor/Sovol-SV08-Mainline/main/README.md))

__Gate:__ successful builds and tests, verified backups, and explicit flashing instructions.

### 6. Flash and commission conservatively

Physical commissioning proceeds in small steps:

1. Flash and confirm communication with both MCUs.
2. Check plausible temperature readings and endstop states.
3. Verify individual motors and directions before homing.
4. Validate fans and controlled heater operation.
5. Verify eddy communication.
6. Perform initial probe calibration using the selected revision’s documented bootstrap procedure.
7. Validate Z homing, repeatability, and ordinary gantry leveling.

An uncalibrated eddy probe must not be treated as ready for normal Z homing; upstream documents a special initial-calibration procedure for this situation. ([klipper3d.org](https://www.klipper3d.org/Eddy_Probe.html))

__Gate:__ reliable ordinary probing and homing before scanned meshes or automatic print-start routines.

### 7. Validate printing, then consider tap

- Calibrate at controlled temperatures.
- Enable conservative scanned meshes after gantry leveling.
- Check repeated meshes and first layers across the bed.
- Validate start, end, pause, resume, cancel, and filament-runout behavior.
- Recheck extrusion, PID tuning, input shaping, and pressure advance as needed.

__Tap is optional and last.__ It requires separate calibration and hardware validation; native scanning and tap also have different clearance, temperature, and cleanliness requirements. ([klipper3d.org](https://www.klipper3d.org/Eddy_Probe.html))

### What I need next

Provide your __complete stock configuration bundle and a recent `klippy.log`__. Also identify what recovery hardware you already have: spare boot media, an eMMC reader, and/or an ST-Link programmer.
