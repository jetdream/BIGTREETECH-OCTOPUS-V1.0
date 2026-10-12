# Laser integration plan — BTT Octopus V1.0 + Marlin

Goal: run two laser sources from the Octopus with power controlled by the board, alongside the existing 3D-printer / mill / drag-knife uses.

- **CO2:** 40 W tube, MCWlaser PSU (MYJG family) with `TH TL WP G IN 5V` connector
- **Diode:** 12 V module with a single 5 V TTL/PWM control input

Reference manuals in this folder:
- `CO2 PSU MYJG-50 manual (Cloudray).pdf`: terminal truth table, IN spec
- `CO2 PSU MYJG-40-130 manual (Jinan Mingyue).pdf`: 40 W model specs

---

## 0. Decisions

| # | Topic | Decision |
|---|---|---|
| D1 | Power control method | **PWM 20 kHz into PSU `IN`**. `TL` is a clean fire/enable signal. No analog (the board has no usable 0–5 V DAC) |
| D2 | Marlin version | **Upgrade to upstream Marlin 2.1.2.8** (latest stable, June 2026) from `MarlinFirmware/Marlin`, then add the laser. Board `BOARD_BTT_OCTOPUS_V1_0`, env `STM32F446ZE_btt`. See §3.1 |
| D3 | Inline power | **Yes**: `M3 I` continuous inline mode + `LASER_POWER_TRAP`. **Not `M4 I`**: in Marlin 2.1 it derives power from feedrate (F>>8) and ignores S |
| D4 | G-code software | **LightBurn**: Marlin device, "Inline" laser mode, start `M3 I`, end `M5 I` |
| D5 | Aux outputs | **Air assist on FAN4 (M8/M9)**, **fume extraction on FAN5 (M10/M11)**, each via relay/SSR |
| D6 | Tube max current | Unknown, so **cap S255 at 16 mA** in hardware (R2), measured with an ammeter |
| D7 | Ammeter | **To be added** in the tube cathode line (0–30 mA) |
| D8 | WP interlock loop | **Water flow switch** only (see the safety note in §6) |
| D9 | Laser selection | **Separate plug-in harnesses** on a shared connector. Only one laser can be connected |
| D10 | Diode supply / routing | **12 V module**, **separate cable** (payload connector spare wires stay unused) |
| D11 | EEPROM | **Back up with `M503`** before flashing; restore tuned values afterwards |

Still open (not blocking): the diode module's model and its maximum PWM frequency (check before step 9 of §5).

---

## 1. Key facts this plan is based on

### PSU (from the manuals)

| Pin | Meaning |
|---|---|
| TL | Fires when pulled low (≤ 0.3 V). Not firing when high (≥ 3 V) |
| TH | Fires when high. Unused here |
| WP | Must be low (tied to G) to allow firing. Open or high means no laser |
| IN | Power set-point: 0–5 V analog **or** PWM at ≥ 20 kHz with ≤ 5 V peak. **IN left open gives about 40 % power** |
| 5V | 5 V output, 20 mA max |
| G | Signal ground |

- Response time is ≤ 1 ms from TL to 90 % of the set current.
- The 40 W supply's maximum output is 20 mA.

### Octopus fan outputs

Verified against BTT's **corrected** schematic `Hardware/BIGTREETECH Octopus - SCH.pdf` (dated 2021-09-18). In 2024 BTT replaced the original `BIGTREETECH Octopus.pdf` that is still in this fork, as "wrong sch".

- **Circuit:** every fan port is a low-side MOSFET (ES3400).
  - It is driven through a 74HCT365 buffer, with 4.7 k pull-downs on the buffer inputs, then a 10 Ω series resistor and a 12 k gate pull-down.
- **Boot:** all fan MOSFETs are **off** during boot and reset, so `FANx−` floats.
- **Output:** `FANx−` can only be **pulled to GND or left floating**; it never drives a voltage.
- **Paths back to the `V_FANx` jumper rail:**
  - an SS54 flyback diode (`FANx−` → `V_FANx`);
  - a **4.7 k resistor + green LED from `V_FANx` to `FANx−`**. This is a built-in pull-up to whatever voltage the jumper selects.
- **So the voltage jumper on FAN1 and FAN2 must be removed (mandatory):**
  - jumper on 24 V or 12 V: the LED path pushes `IN` above its 5 V limit;
  - jumper fitted while the board is unpowered: the diode/LED path drags `TL` toward 0 V, so the laser could fire.

### Hardware timers

| Fan port | MCU pin | Timer | Notes |
|---|---|---|---|
| FAN0 | PA8 | TIM1 | E0 auto-fan (extruder) |
| FAN1 | PE5 | **TIM9** | Only user of TIM9, so it can run its own 20 kHz |
| FAN2 | PD12 | TIM4 | Used as a digital pin only (enable) |
| FAN3 | PD13 | TIM4 | Controller fan |
| FAN4 | PD14 | TIM4 | Digital pin only (air assist) |
| FAN5 | PD15 | TIM4 | Digital pin only (fume extraction) |

- **Timer use in Marlin 2.1.2.8 on the Octopus:**
  - stepper on **TIM8** (the Octopus pins file sets `STEP_TIMER 8`; it was TIM10 in the old fork);
  - temperature on TIM14, servo on TIM5, tone on TIM6;
  - **TIM9 is free.**
- **Frequency handling:** in 2.1.x the STM32 HAL keeps the PWM frequency per timer. The 2.0.9 problem, where the controller fan's `analogWrite` reset TIM4 to 1 kHz every 2.5 s, is gone. FAN1/TIM9 is still the right choice: it has its own timer, separate from `FAST_PWM_FAN` on TIM4.
- **Fan port budget:** after this plan, all six fan ports are used (E0 hotend fan, laser PWM, laser enable, controller fan, air assist, exhaust). **No M106 part-cooling fan output is left.** If you need one later, move the controller fan to an always-on header.

---

## 2. Target design

### Signal assignment

| Function | Octopus port | MCU pin | Firmware role |
|---|---|---|---|
| Laser power (PWM 20 kHz) | **FAN1−** | PE5 / TIM9 | `SPINDLE_LASER_PWM_PIN` (inverted) |
| Laser fire / enable | **FAN2−** | PD12 | `SPINDLE_LASER_ENA_PIN`, active HIGH |
| Air assist | **FAN4−** | PD14 | `AIR_ASSIST_PIN`, active HIGH (M8/M9) |
| Fume extraction | **FAN5−** | PD15 | `AIR_EVACUATION_PIN`, active HIGH (M10/M11) |
| Controller fan | FAN3 | PD13 | unchanged |

- **Jumpers:** remove on FAN1 and FAN2.
- **FAN4/FAN5 with a relay module or SSR:** use FAN+ (jumper set to that coil's voltage) and FAN− as a switched-low output. A bare relay coil is fine because the port has a flyback diode. For an SSR input, use FAN+ (5 V or 12 V jumper) to SSR+ and FAN− to SSR−.

### CO2 harness

```
PSU 5V ──R1 1k──┬──────────── PSU IN
                ├──────────── Octopus FAN1−        (power PWM)
                └──R2──────── PSU G                (hardware cap: S255 → 16 mA)

PSU TL ────────────────────── Octopus FAN2−        (fire: board pulls low)
PSU G  ────────────────────── Octopus GND          (endstop/probe header GND)
PSU WP ── water-flow switch ── PSU G
PSU TH ── not connected
Tube cathode ── mA meter (0–30 mA) ── PSU current return
```

How it behaves in each state:

| State | FAN1 (IN) | FAN2 (TL) | Laser |
|---|---|---|---|
| Board off or booting, PSU on | IN pulled up | TL high (PSU pull-up) | **No laser** (TL gates) |
| `M5` / idle | MOSFET held on, IN = 0 V | TL high | No laser |
| `M3 S128` | ~50 % duty at 20 kHz | TL low | ~50 % set-point |
| `M3 S255` | IN = 5 V × R2/(R1+R2) | TL low | ~16 mA (capped by R2) |

R2 sizing:
1. Start with no R2 and fire short pulses at rising S while watching the mA meter.
2. Fit R2 so S255 gives 16 mA. Start with 4.7 k (about 4.1 V) and adjust. The IN input impedance also loads the divider, so always measure.
3. Write down the S → mA table in this folder.

Power below roughly S40–S50 (about 15–20 %) may not ignite the tube; that is normal for CO2. LightBurn's "Min power" for the CO2 device should be set to just above the ignition point.

### Diode harness (gated by enable)

```
5V (probe/BLTouch header) ──1k──┬──────────────── Octopus FAN1−   (power PWM)
                                └──► diode TTL input
                                     │
                                     C   NPN (2N3904 / BC547)
                                     E ── GND
                                     B ──10k──┬── Octopus FAN2−   (enable)
                                              └──10k── 5V
Octopus GND ──────────────── diode GND (and 12 V supply −)
12 V supply + ────────────── diode 12 V +
```

- **FAN2 floating** (board off, booting, or laser disabled): the NPN is on and holds TTL at 0 V, so the laser is off.
- **FAN2 low** (laser enabled): the NPN is off and TTL follows the PWM on FAN1.
- **12 V source:** a separate 12 V PSU, or the Octopus 12 V rail if the module draws well under its 4 A budget. Check the module's current rating.
- **Cable:** a dedicated cable to the tool head (3 wires: 12 V, GND, TTL). The adapter board sits at the Octopus end, next to the shared connector.

### Laser selection

- One connector at the board carries FAN1−, FAN2−, GND and 5 V. Either the **CO2 harness** or the **diode harness** plugs into it, never both.
- Use a connector type that cannot be confused with the extruder or fan plugs.

---

## 3. Firmware

### 3.1 Port to Marlin 2.1.2.8 (D2)

**Where the Octopus support comes from:**
- Octopus support has been native in upstream Marlin since mid-2021. This fork's `Marlin-bugfix-2.0.x` is simply upstream `bugfix-2.0.x` from 2021-07-07, snapshotted by BTT.
- BTT's own repository has since moved only to `bugfix-2.0.9.3.x`, not 2.1. So take the source from upstream **`MarlinFirmware/Marlin`, tag `2.1.2.8`**.
- The Spindle/Laser section of `bugfix-2.1.x` is functionally identical, so the stable release is enough.
- `MarlinFirmware/Configurations` has **no ready Octopus machine config**, only a `linear_axes/Octopus 8 LINEAR_AXES` demo. Start from the default `Configuration.h`/`Configuration_adv.h` and apply the settings below.

Put it next to the old firmware: `Firmware/Marlin-2.1.2.8/`. The old `Marlin-bugfix-2.0.x` stays as a rollback.

**Upstream 2.1.2.8 versus this fork's patches:**

| Fork patch (2.0.x) | In upstream 2.1.2.8 |
|---|---|
| Board `BTT_OCTOPUS_V1_1` | Use **`BOARD_BTT_OCTOPUS_V1_0`** (your board). V1.0 and V1.1 differ only by an LCD serial-port default |
| Env `BIGTREE_OCTOPUS_V1` | Renamed **`STM32F446ZE_btt`**. Same 0x8000 offset, so SD-card flashing and the bootloader are unchanged |
| MOTOR 3: Z2 pins renamed to Y2 in the pins file | **No pins patch needed.** Upstream only auto-assigns Y2 pins if they are undefined, so define them in `Configuration.h`: `Y2_STEP_PIN PG4`, `Y2_DIR_PIN PC1`, `Y2_ENABLE_PIN PA0`, `Y2_CS_PIN PC7` |
| `FAN_PIN` (FAN0) commented out so it can be the E0 auto-fan | **Not needed.** 2.1 automatically excludes a fan pin that equals `E0_AUTO_FAN_PIN` from M106; the same applies to `CONTROLLER_FAN_PIN` (FAN3) |
| FAN1/FAN2 commented out | **Still needed, now also FAN4/FAN5**: see §3.2. This is the **only remaining source patch** |
| `#undef NEOPIXEL_PIN` for FYSETC 2.1 | **Not needed.** Upstream sets `NEOPIXEL_PIN` for FYSETC_MINI_12864_2_1 and keeps the board RGB header separate (`BOARD_NEOPIXEL_PIN`) |
| `marlinui_DOGM.cpp` contrast forced to 255 | **Don't patch code.** Set `LCD_CONTRAST_INIT` in config or use `M250` |
| EEPROM 32 KB definition | Upstream uses 4 KB software-I2C (AT24C32 is 4 KB). EEPROM is reset anyway (§4) |
| TMC software-SPI pins | Upstream defines `TMC_SPI_*` pins; nothing to do |

**Settings to carry over** (option names may differ slightly in 2.1; the build's sanity checks flag renamed options):

Settings were taken from a diff of the 2.0 fork's config against **untouched upstream `bugfix-2.0.x` of 2021-07-07**, so BTT's own changes are included too. **Done (2026-10-12)**, as ported:

- **`Configuration.h`**
  - **Board and serial:**
    - board `BTT_OCTOPUS_V1_0`; author string updated;
    - `SERIAL_PORT 1` (UART header), **`SERIAL_PORT_2 -1` (USB virtual COM, enabled)**, `BAUDRATE 115200`.
  - **Steppers:**
    - TMC5160 on X, Y, Y2, Z, E0; Y2 pins on MOTOR 3 (`PG4/PC1/PA0/PC7`);
    - steps 400/400/400/400, max feedrate 133/50/50/100, max accel 700/700/100/10000;
    - accels 700 (print) / 10000 (retract) / 700 (travel), `S_CURVE_ACCELERATION`;
    - `ENDSTOP_NOISE_THRESHOLD 3`.
  - **Motion and machine:**
    - direction inversions: Y false, Z true, E0 true;
    - bed 235 × 235, `Z_MAX_POS 100`, homing 10 mm/s on all axes;
    - no probe: `Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN` off, `NOZZLE_TO_PROBE_OFFSET {0,0,0}`.
  - **Hotend and bed:**
    - 1 extruder; `TEMP_SENSOR_0 1`, `TEMP_SENSOR_BED 1`; `BED_MAXTEMP 100`;
    - PID `DEFAULT_KP/KI/KD` 19.00 / 1.29 / 70.09 (2.1 uses upper-case names);
    - `EXTRUDE_MAXLENGTH 120`; preheat fan 255.
  - **Storage and UI:**
    - `EEPROM_SETTINGS`, `SDSUPPORT`, `SPEAKER` with LCD feedback duration 0 / 1000 Hz;
    - `FYSETC_MINI_12864_2_1`. Contrast now defaults to 255 for this panel in 2.1, so the old code patch is not needed.
  - **Display backlight:** `NEOPIXEL_LED`, **`NEO_RGB`**, 3 px, brightness 255.
    - 2.1 refuses anything but `NEO_RGB` for this panel.
    - 2.0 silently forced `NEO_RGB` too, so the fork's `NEO_GRB` never took effect; colours are unchanged.
- **`Configuration_adv.h`**
  - **Fans:**
    - controller fan on FAN3 (speed 70, idle 3 s, M710 editable); E0 auto-fan on PA8;
    - `FAST_PWM_FAN` at **25 kHz**. The fork's 78100 no longer fits: 2.1's STM32 HAL takes a 16-bit frequency, and 78100 would wrap to about 12.6 kHz, which is audible.
  - **Motion:** `QUICK_HOME`. `Y_DUAL_STEPPER_DRIVERS` no longer exists; Y2 comes from `Y2_DRIVER_TYPE`.
  - **TMC:**
    - 1200 mA on X/Y/Y2/Z, 800 mA on E0, Rsense 0.075, 16 µsteps (`UNIFIED_MICROSTEPS`/`TMC5160_RSENSE` helpers kept);
    - Y2 interpolate; spreadCycle on XY; `CHOPPER_DEFAULT_24V`; `TMC_DEBUG`.
  - **LED menu:** `LED_CONTROL_MENU`, which 2.0 enabled automatically for this panel and 2.1 does not.
  - **Debug:** `DIRECT_PIN_CONTROL`, `PINS_DEBUGGING`.
  - **Not carried over:** `MICROSTEP_MODES {256…}`. It only applies to boards with MS pins (none on the Octopus), and 256 is outside its 1–16 range.
- **`platformio.ini`:** `default_envs = STM32F446ZE_btt`.
- **Pins file:** see the table above. The only patch is the fan-pin change in §3.2.

**Build warnings:**
- *"Motherboard DIAG jumpers must be removed when SENSORLESS_HOMING is disabled."* **Check that no DIAG jumpers are fitted** on the X/Y/Z/E0 driver sockets. With them fitted, the TMC DIAG outputs fight the physical endstop switches.
- Two FYSETC panel "works best with `LED_USER_PRESET_STARTUP`" warnings. That option was also off in 2.0, so behaviour is unchanged.

Bring the machine up as a printer/mill on 2.1.2.8 first (homing, moves, extruder, temps) **before** enabling the laser.

### 3.2 Laser settings (Marlin 2.1.2.8)

**Pins patch: the one source change.**

- **Why it's needed:**
  - Upstream `pins_BTT_OCTOPUS_V1_common.h` defines `FAN0_PIN`…`FAN5_PIN` unconditionally.
  - With the laser PWM on PE5, the build stops with `#error "SPINDLE_LASER_PWM_PIN conflicts with FAN1_PIN."`.
  - FAN2, FAN4 and FAN5 would also be initialised and driven as M106 fans, fighting the laser-enable and air outputs.
- **The patch:** wrap each fan define in `#ifndef`, so the config can switch the port off:

```cpp
// pins_BTT_OCTOPUS_V1_common.h (patch: make fan pins overridable)
#ifndef FAN0_PIN
  #define FAN0_PIN                          PA8   // Fan0
#endif
#ifndef FAN1_PIN
  #define FAN1_PIN                          PE5   // Fan1
#endif
// … same for FAN2_PIN … FAN5_PIN
```

**`Configuration_adv.h`:**

```cpp
// Release fan ports used by the laser and air outputs (needs the #ifndef patch above)
#define FAN1_PIN -1                          // FAN1 → laser PWM
#define FAN2_PIN -1                          // FAN2 → laser enable
#define FAN4_PIN -1                          // FAN4 → air assist
#define FAN5_PIN -1                          // FAN5 → fume extraction

#define LASER_FEATURE
#define SPINDLE_LASER_ENA_PIN       PD12    // FAN2- → PSU TL / diode gate (2.1: pin defined here, not in pins file)
#define SPINDLE_LASER_PWM_PIN       PE5     // FAN1- → PSU IN / diode TTL (TIM9, own timer)
#define SPINDLE_LASER_ACTIVE_STATE  HIGH    // pin HIGH → MOSFET on → TL low → fire
#define SPINDLE_LASER_USE_PWM               // 2.0.9 name was SPINDLE_LASER_PWM
#define SPINDLE_LASER_PWM_INVERT    true    // MOSFET + pull-up inverts the signal
#define SPINDLE_LASER_FREQUENCY     20000   // PSU manual: IN PWM ≥ 20 kHz (STM32 HAL supports it)
#define CUTTER_POWER_UNIT           PWM255  // LightBurn S-value max = 255

#define LASER_SAFETY_TIMEOUT_MS     1000    // default; laser off if no motion for 1 s
#define LASER_POWER_SYNC                    // M3 S-values queued with motion, no planner stall
#define LASER_POWER_TRAP                    // scale power with accel/decel (replaces 2.0.9 TRAPEZOID)
// SPEED_POWER_MIN: set to the CO2 ignition level found in §5 step 6, so TRAP never drops below it

#define AIR_ASSIST                          // D5
#define AIR_ASSIST_ACTIVE           HIGH
#define AIR_ASSIST_PIN              PD14    // FAN4
#define AIR_EVACUATION                      // D5
#define AIR_EVACUATION_ACTIVE       HIGH
#define AIR_EVACUATION_PIN          PD15    // FAN5
```

**What changed from 2.0.9:**

- **Removed options:** `LASER_POWER_INLINE`, `LASER_POWER_INLINE_TRAPEZOID` and `LASER_MOVE_POWER` no longer exist. Inline power is built in and switched on at run time:
  - `M3 I`: continuous inline. `G1 … S…` sets power per move.
  - `M5 I`: leaves inline mode.
- **Don't use `M4 I`:** in 2.1 it computes power from the **feedrate** (`F >> 8`), ignores S, and is marked experimental in the code.
- **Optional, new in 2.1: `LASER_COOLANT_FLOW_METER`** with `FLOWMETER_SAFETY`.
  - Hardware: a Hall-effect flow sensor on a free interrupt-capable input (e.g. a spare endstop or DET pin).
  - Marlin then refuses to fire below a minimum flow (L/min).
  - It works *in addition to* the WP flow switch, and shows the flow rate on the display.

### 3.3 LightBurn device settings (D4)

- **Device:** type Marlin. **Laser mode: Inline** (not M03/M05). **S-value max:** 255.
- **Start G-code:** `M3 I`. **End G-code:** `M5 I`.
- **Check a generated file** before the first job: power must be on `G1 … S…`, with no `M4`.
- **Air assist:** LightBurn's per-layer air assist emits M8/M9, if enabled in device settings.
- **Framing caveat:** LightBurn users report that in Inline mode, framing with "laser on" doesn't fire until a job has run. Frame with the laser off, or with a low-power pointer.
- **Two device profiles,** with their own min/max power and speeds:
  - **CO2:** min power just above tube ignition.
  - **Diode.**

---

## 4. Build and flash

**Built binaries** (2026-10-12, env `STM32F446ZE_btt`, in git-ignored `Firmware/Marlin-2.1.2.8/.pio/out/`):

| File | Content | Use |
|---|---|---|
| `firmware-2.1.2.8-machine-only.bin` | §3.1 port only; FAN1/2/4/5 are still M106 fans | First flash: verify the machine works as before |
| `firmware-2.1.2.8-laser.bin` | §3.1 + §3.2 (laser, air assist, extraction) | After the machine-only build checks out |

`laser.bin` is also the current `.pio/build/STM32F446ZE_btt/firmware.bin`.

The laser build has **no `M106`/`M107`** (`FAN_COUNT 0`): every fan port is either the hotend auto-fan, the controller fan, or a laser/air output. Slicer `M106` lines are just reported as unknown commands.

The resolved configuration was checked with a preprocessor dump of the real build flags:
- laser PWM `PE5` inverted at 20 kHz; laser enable `PD12`, active HIGH;
- air assist `PD14` and extraction `PD15`, both active HIGH;
- `FAN1/2/4/5_PIN -1`; Y2 on `PG4/PC1/PA0/PC7`; `STEP_TIMER 8`.

**Rollback:** the 2.0 fork no longer builds with the current PlatformIO 6.2 (its 2021 dependency script reinstalls libraries on every run and then fails to find them). Before copying the new file, **look for `FIRMWARE.CUR` on the SD card you last flashed with and copy it to the PC**. The bootloader renames each flashed `firmware.bin` to `FIRMWARE.CUR`, so it is the image currently on the board.

**Steps:**
1. Copy `FIRMWARE.CUR` off the SD card (rollback image).
2. Connect to the current firmware and send `M503`. Save the output to `My setup/M503 backup <date>.txt` (D11).
3. Copy the chosen `.bin` to the SD card root **as `firmware.bin`** and power-cycle. Use the bootloader method, not DFU, so the bootloader stays intact.
4. `M502` then `M500` to initialise EEPROM. Then re-apply any value from the backup that differs from the new config, and `M500`.
5. Rebuilding: `pio run -d Firmware/Marlin-2.1.2.8 -e STM32F446ZE_btt`, or the PlatformIO IDE with that folder open.

---

## 5. Bring-up and test sequence

Do these steps with the tube's HV disconnected, or the PSU's laser switch off, until step 6. Measure each pin against PSU G.

1. **Board off, PSU on:** TL ≥ 3 V. This confirms the FAN2 jumper is removed and no diode pull-down.
2. **Board on, idle / `M5`:** TL ≥ 3 V, IN ≈ 0 V.
3. **`M3 S255`:** TL ≤ 0.3 V, IN ≈ max (R2-capped value).
4. **`M3 S128`:** IN reads ≈ half on a DMM (averaged). Scope check: 20 kHz, clean edges, ≤ 5 V peak.
5. **`M5`:** back to step 2. Also reset the board during `M3` and confirm TL goes high.
6. **Reconnect the tube.**
   - PSU TEST button first (tube and PSU OK).
   - Then `M3 S…` steps while watching the mA meter: find the ignition S and fit R2 for 16 mA at S255.
7. **Interlock:** stop the water flow and confirm no laser at `M3 S255`.
8. **Aux outputs:** `M8`/`M9` switch the air pump; `M10`/`M11` switch the exhaust.
9. **Motion:**
   - Send `M3 I`, then a square at low power with `G1 … S…`, then `M5 I`.
   - Confirm the laser stops within 1 s if motion stalls (`LASER_SAFETY_TIMEOUT_MS`).
   - Then run a small LightBurn job.
10. **Diode harness:**
    - Repeat 1–5 on the TTL output: low at boot, idle and reset; follows PWM when enabled.
    - Confirm the module responds properly at 20 kHz (power should scale with S).
    - Wear goggles rated for its wavelength.

---

## 6. Safety checklist

- **Water cooling** running before any CO2 firing; the flow switch is in WP.
- **Interlocks:** only a water flow switch is in the WP loop (D8). Strongly consider adding a **lid switch** and a **key/arm switch** in series later. They cost little and stop the CO2 from firing while the enclosure is open.
- **Optics:** the CO2 enclosure must block 10.6 µm (acrylic or polycarbonate panels). Use diode goggles for the diode wavelength.
- **Air:** air assist and fume extraction for every cut.
- **Fire:** extinguisher nearby; never leave the machine unattended while cutting.
- **High voltage:** the PSU HV leg holds charge after power-off; wait before touching it.
- **Earthing:** PSU case, machine frame and Octopus GND are earthed together.

---

## 7. Work order

1. ~~Download upstream Marlin 2.1.2.8 into `Firmware/Marlin-2.1.2.8/` and port the config (§3.1).~~ **Done, built.**
2. ~~Apply the fan-pin `#ifndef` patch and add the laser config (§3.2).~~ **Done, built.**
3. Check that the DIAG jumpers are removed. Save `FIRMWARE.CUR` from the SD card and back up `M503` (§4).
4. Flash `firmware-2.1.2.8-machine-only.bin` and verify homing, moves, extruder, temps and display.
5. Flash `firmware-2.1.2.8-laser.bin`. Re-check the machine, then check `M3`/`M5`/`M8`/`M10` on the FAN ports with a meter before connecting anything.
6. Build the CO2 harness (R1, R2 site, connector), remove the FAN1/FAN2 jumpers, install the mA meter.
7. Run §5 steps 1–9 for the CO2.
8. Wire the air-assist and exhaust relays/SSRs on FAN4/FAN5.
9. Build the diode adapter + harness; run §5 step 10.
10. Set up the LightBurn device profiles (CO2, diode).
