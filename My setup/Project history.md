# Project history

A dated record of what was done and **why**, including options that were considered and rejected. Add new entries at the end of the timeline. Record a decision here, not only in a chat.

## Timeline

| Date | Commit | What |
|---|---|---|
| 2021-06-18 … 07-22 | `7701e5a` … `35390de` | BTT publishes the Octopus repo: manuals, schematics, Klipper/RRF, and a Marlin `bugfix-2.0.x` snapshot (updated to "2.0.9.1.x" on 2021-07-08; its source date is 2021-07-07) |
| 2021-07-22 | `77278bb`, `c9cb7d8` | Fork and first machine config (details below) |
| 2024-09-04 | `099a818` | Tuning pass (details below) |
| 2025-12-18 | `248aeea` | Tool-head payload connector drawing (`Payload connector.svg`). Spare BROWN/GREEN/24V wires exist in the tool-head cable |
| 2026-01-04 | `6cb3fba` | `hw-config.md`: notes on the 2.0 laser settings (enable FAN1/PE5, PWM FAN2/PD12, PWM255, no inline power, G0 doesn't switch the laser off) |
| 2026-10-11/12 | `f04417d`, `d79ab84` | Laser integration study → `Laser integration plan.md`, CO2 PSU manuals, BTT's corrected schematic |
| 2026-10-12 | `c473dc1`, `5665f80` | Upstream Marlin 2.1.2.8 imported unmodified, then the machine config ported and laser/air support added. Two images built, none flashed yet |

**2021-07-22, first machine config:**
- TMC5160 on X/Y/Z/E0 plus Y2 on MOTOR 3 (by renaming the pins file's Z2 pins); Rsense 0.075;
- FYSETC 12864 v2.1 display; hotend auto-fan on FAN0;
- `LASER_FEATURE` enabled with enable on FAN1/PE5 and PWM on FAN2/PD12.

**2024-09-04, tuning pass:**
- PID 19.00 / 1.29 / 70.09;
- accelerations raised from 30 test values to 700;
- feedrates 133/50/50/100 mm/s;
- controller fan on FAN3, `FAST_PWM_FAN` 78.1 kHz;
- spreadCycle on XY, 24 V chopper timing.

## Decisions (2026-10 laser session)

1. **Can the existing 2.0 firmware drive a laser without a rebuild? Mostly no, and it is unsafe as configured.**
   - **Enable signal:** the enable was active-LOW. Through a low-side fan MOSFET, that makes "laser on" equal to the boot/reset state, so the laser fires during boot.
   - **PWM:** non-inverted, so through a MOSFET plus pull-up it comes out inverted.
   - **Frequency:** the laser PWM (PD12, TIM4) shared a timer with the controller fan on FAN3. The fan's `analogWrite` reset TIM4 to 1 kHz every 2.5 s.
   - **The one safe no-rebuild option** was pulsing the PSU's `TL` from FAN2, with a pot on `IN` setting the maximum. It was superseded by the requirement for board-controlled power on `IN`.
2. **PSU behaviour,** from the MYJG manuals now in this folder:
   - `TL` fires when low;
   - `WP` must be low to fire;
   - `IN` takes 0–5 V analog or PWM ≥ 20 kHz with ≤ 5 V peak;
   - **`IN` left open gives about 40 % power.**
   The 5V pin supplies 20 mA max.
3. **Analog 0–5 V on IN from the board: rejected.** The STM32's DACs are on PA4 (LCD SD chip select) and PA5 (SPI clock shared with the TMC drivers), only reach 3.3 V, and Marlin has no DAC laser path.
4. **Chosen:**
   - **PWM 20 kHz into `IN`, plus `TL` as a clean enable.** Pulsing `TL` (the K40 style) gives better greyscale and is the fallback alternative.
   - **Pins:** PWM on **FAN1/PE5**, the only fan pin on its own timer (TIM9); enable on **FAN2/PD12**; air assist FAN4, extraction FAN5.
   - **Signal polarity:** PWM inverted in firmware, because of the MOSFET plus the pull-up to PSU 5V; enable active-HIGH.
5. **Corrected schematic:** BTT replaced the schematic in 2024 ("wrong sch").
   - The corrected one shows a 4.7 k + LED path from `V_FAN` to `FANx−` on every fan port.
   - So **removing the FAN1/FAN2 voltage jumpers is mandatory**, not optional.
   - It also confirmed that all fan MOSFETs are off during boot and reset (pull-downs on the buffer inputs and gates).
6. **Marlin version:** the user chose to upgrade to 2.1.x.
   - Octopus support has been in upstream Marlin since 2021; BTT's repo only reached `bugfix-2.0.9.3.x`.
   - So we took **upstream 2.1.2.8**, the latest stable release (June 2026); its laser code matches `bugfix-2.1.x`.
   - Board define `BTT_OCTOPUS_V1_0` (the fork had V1_1; the two are nearly identical). Env `STM32F446ZE_btt`.
7. **One unavoidable source patch.**
   - Upstream refuses `SPINDLE_LASER_PWM_PIN` equal to `FAN1_PIN`, and the Octopus pins file defines its fan pins unconditionally.
   - The fix wraps them in `#ifndef`, so the config can set `FAN1/2/4/5_PIN -1`.
   - Y2 on MOTOR 3 needed **no** patch: upstream honours `Y2_*_PIN` defined in `Configuration.h`.
8. **Inline power:** `M3 I` + `LASER_POWER_TRAP` + `LASER_POWER_SYNC`.
   - **`M4 I` rejected:** in 2.1 it computes power from the feedrate (`F >> 8`) and ignores S. Marlin's code marks it experimental.
   - LightBurn: Marlin device, "Inline" mode.
9. **User answers** (plan §0):
   - **Software:** LightBurn.
   - **Air:** assist and extraction both under board control.
   - **Tube current:** rating unknown, so capped at 16 mA with R2 (a 0–30 mA meter to be added).
   - **Interlocks:** water flow switch only.
   - **Laser selection:** separate plug-in harnesses, so only one laser can be connected.
   - **Diode laser:** a 12 V module on a separate cable.
   - **EEPROM:** back up with `M503` right before flashing.
10. **Port details worth remembering:**
    - **Full settings list:** taken by diffing the fork against **untouched upstream 2021-07-07 defaults** (not BTT's copy), which surfaced BTT's own changes too: baud 115200, USB serial on, bed thermistor, S-curve, 235 mm bed, EEPROM/SD/speaker.
    - **Fan PWM:** `FAST_PWM_FAN_FREQUENCY` 78100 → **25000**, because the 2.1 STM32 HAL takes `uint16_t`.
    - **NeoPixel:** `NEO_RGB` is required by 2.1 for the FYSETC 2.1 panel. 2.0 forced it anyway, so the fork's `NEO_GRB` never applied.
    - **LED menu:** `LED_CONTROL_MENU` set explicitly (2.0 enabled it automatically).
    - **No `M106`/`M107`** in the laser build: `FAN_COUNT 0`, because every fan port has another job.
    - **Laser switches off on `G0`** in 2.1, unlike 2.0.
11. **Rollback:** the 2.0 project no longer builds with PlatformIO 6.2 (its 2021 dependency script reinstalls libraries every run and then can't find them). The rollback image is therefore `FIRMWARE.CUR` from the SD card.
