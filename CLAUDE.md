# BTT Octopus CNC: project context

This repo is a fork of `bigtreetech/BIGTREETECH-OCTOPUS-V1.0`. It holds the firmware and docs for a handmade multi-tool CNC (3D printer, mill, drag knife, CO2 and diode laser).

- **Remotes:** `origin` = `jetdream/BIGTREETECH-OCTOPUS-V1.0`, `upstream` = `bigtreetech/...`.
- **Docs to read first:**
  - `My setup/Handoff.md`: current state and next steps.
  - `My setup/Laser integration plan.md`: laser design, wiring, firmware settings, tests, work order.
  - `My setup/Project history.md`: past decisions and why, including rejected options.

## Machine

- **Board:** BTT Octopus **V1.0** (STM32F446ZET6), 24 V.
- **Steppers:** TMC5160 over SPI, Rsense 0.075 Ω, 16 µsteps, 400 steps/mm (8 mm lead).

  | Axis | Socket | Current |
  |---|---|---|
  | X | MOTOR 0 | 1200 mA |
  | Y | MOTOR 1 | 1200 mA |
  | Z | MOTOR 2 | 1200 mA |
  | Y2 | MOTOR 3 (upstream calls this socket Z2) | 1200 mA |
  | E0 | MOTOR 4 | 800 mA |

- **Machine:** bed 235 × 235, Z 100. Physical endstops, no probe.
- **Display:** FYSETC Mini 12864 v2.1, NeoPixel backlight.
- **Fan ports:**
  - FAN0 (PA8): hotend auto-fan.
  - FAN3 (PD13): controller fan.
  - In the laser build: FAN1 laser PWM, FAN2 laser enable, FAN4 air assist, FAN5 fume extraction.
- **Tool-head cable:** `My setup/Payload connector.svg` (stepper, heater, thermistor, FAN0, spare BROWN/GREEN/24V wires).
- **CO2:** 40 W tube, MCWlaser PSU (MYJG family, `TH TL WP G IN 5V`).
- **Diode laser:** 12 V module with a 5 V TTL input.

## Repo layout

| Path | What |
|---|---|
| `Firmware/Marlin-2.1.2.8/` | **Active firmware.** Upstream Marlin 2.1.2.8 + our config |
| `Firmware/Marlin-bugfix-2.0.x/` | Old BTT snapshot (bugfix-2.0.x of 2021-07-07) + user config. Probably what is on the board now. Reference only: it **no longer builds** with current PlatformIO, and its laser settings are unsafe (see history) |
| `My setup/` | The user's own docs: plan, handoff, history, CO2 PSU manuals, payload connector, `hw-config.md` (notes on the **old 2.0** laser pins) |
| `Hardware/` | BTT schematics and pinout. Use **`BIGTREETECH Octopus - SCH.pdf`** (corrected, 2021-09-18). `BIGTREETECH Octopus.pdf` is the version BTT removed as "wrong sch" |
| everything else | BTT's material (manuals, Klipper, RRF, Voron) |

## Build, verify, flash

- **Build:** `~/.platformio/penv/Scripts/pio.exe run -d Firmware/Marlin-2.1.2.8 -e STM32F446ZE_btt` (PlatformIO Core 6.2.0, about 1 min).
  - Output: `Firmware/Marlin-2.1.2.8/.pio/build/STM32F446ZE_btt/firmware.bin`.
  - Named copies for the user go in `Firmware/Marlin-2.1.2.8/.pio/out/` (git-ignored).
- **Verify the resolved config,** not just that it compiles:
  1. Get flags with `pio project metadata -d Firmware/Marlin-2.1.2.8 -e STM32F446ZE_btt --json-output`.
  2. Run `arm-none-eabi-g++ -E -dM -x c++ <cxx_flags> <-D defines> <-I includes> Marlin/src/inc/MarlinConfig.h`.
  3. Grep the result for the pins and options in question.
- **Flash:** **the user flashes**; Claude never does.
  - Copy the image to the SD card root as `firmware.bin` and power-cycle. The bootloader at 0x8000 renames it to `FIRMWARE.CUR`, which is also the rollback image.
  - Never DFU: it erases the bootloader.

## Firmware conventions

- **Upstream stays untouched** apart from:
  - `Marlin/Configuration.h` and `Marlin/Configuration_adv.h`;
  - `platformio.ini` (`default_envs`);
  - one pins patch: `FAN0..FAN5_PIN` wrapped in `#ifndef` in `Marlin/src/pins/stm32f4/pins_BTT_OCTOPUS_V1_common.h`, so the config can release fan ports.
- **Seeing local changes:** commit `c473dc1` is pristine upstream, so `git diff c473dc1 -- Firmware/Marlin-2.1.2.8` shows every local change.
- **Marking:** mark our own changes with `// PS:` comments (the user's convention).
- **Renamed options in 2.1** (the build's sanity checks catch most):
  - PID names are upper-case (`DEFAULT_KP/KI/KD`);
  - `SPINDLE_LASER_PWM` → `SPINDLE_LASER_USE_PWM`;
  - `Y_DUAL_STEPPER_DRIVERS` and `LASER_POWER_INLINE` are gone.
- **Config is the source of truth for pins:** the laser/air pins and the Y2 pins are defined in the config, not in the pins file.

## Pitfalls already paid for

- **Fan ports only sink current.** Each is a low-side MOSFET with an SS54 diode and a 4.7 k + LED path to the `V_FAN` jumper rail, so a port can only pull to GND.
  - **Remove the voltage jumper** on any port used as a laser signal (FAN1, FAN2). Otherwise IN sees more than 5 V, and an unpowered board can drag TL low.
  - All fan MOSFETs are off during boot and reset.
- **Laser pins are swapped vs the old 2.0 config:**
  - PWM: **FAN1/PE5** (TIM9, 20 kHz, inverted) → PSU `IN`.
  - Enable: **FAN2/PD12** (active HIGH) → PSU `TL`.
  - `My setup/hw-config.md` still describes the old assignment.
- **FAN1 is on while the laser is off** (inverted PWM). Never connect anything else to FAN1.
- **No `M106`/`M107` in the laser build** (`FAN_COUNT 0`).
- **STM32 PWM frequency is 16-bit in 2.1** (`set_pwm_frequency(uint16_t)`); values above 65535 wrap silently. `FAST_PWM_FAN_FREQUENCY` is 25000.
- **Never use `M4 I`:** in 2.1 it derives power from the feedrate and ignores S. Use `M3 I` with `G1 … S…`; LightBurn uses "Inline" mode, start `M3 I`, end `M5 I`.
- **`G0` turns the laser off in 2.1** (inline: power 0; standard: `apply_power(0)`). In 2.0 it did not.
- **The FYSETC 2.1 panel requires `NEOPIXEL_TYPE NEO_RGB`.** 2.0 forced it silently.
- **Downloads go to the scratchpad, never into the repo.** Stray upstream `.cpp` files once landed in a pins folder and would have broken the build.

## Working agreements

- **Split of work:** the user does wiring, measurements and flashing. Claude prepares configs, builds and docs, and says when something needs a meter or a scope check.
- **Git:** commit and push only when asked. End commits with the Co-Authored-By line from the system instructions.
- **End of a work session:** update `My setup/Handoff.md`, and add decisions to `My setup/Project history.md`.
