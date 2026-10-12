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
| D2 | Marlin version | **Upgrade to Marlin 2.1.x first**, then add the laser |
| D3 | Inline power | **Yes**: power carried in motion (`G1 … S…`), M3 constant / M4 dynamic modes |
| D4 | G-code software | **LightBurn** (Marlin device profile) |
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

- Every fan port is a low-side MOSFET (ES3400), driven through a 74HCT365 buffer.
- Each port has an SS54 flyback diode from `FANx−` to the `V_FANx` jumper rail.
- So `FANx−` can only be **pulled to GND or left floating**; it never drives a voltage.
- At boot and reset all MOSFETs are off, so `FANx−` floats.
- With the voltage jumper fitted and the board unpowered, the diode can drag an externally pulled-up line toward 0 V. **Remove the voltage jumper on any fan port used for laser signals** (FAN1, FAN2).

### Hardware timers

| Fan port | MCU pin | Timer | Notes |
|---|---|---|---|
| FAN0 | PA8 | TIM1 | E0 auto-fan (extruder) |
| FAN1 | PE5 | **TIM9** | Only user of TIM9, so it can run its own 20 kHz |
| FAN2 | PD12 | TIM4 | Used as a digital pin only (enable) |
| FAN3 | PD13 | TIM4 | Controller fan (its `analogWrite` resets TIM4 to 1 kHz every 2.5 s) |
| FAN4 | PD14 | TIM4 | Digital pin only (air assist) |
| FAN5 | PD15 | TIM4 | Digital pin only (fume extraction) |

Marlin 2.0.9 uses TIM10 (stepper), TIM14 (temperature), TIM5 (servo) and TIM6 (tone). **Re-check this on 2.1.x.**

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

### 3.1 Port to Marlin 2.1.x (D2)

Put the new firmware next to the old one: `Firmware/Marlin-2.1.x/`. The old `Marlin-bugfix-2.0.x` stays as a rollback.

Settings to carry over from the current fork (diff against BTT stock):

- **`Configuration.h`**
  - board `BTT_OCTOPUS_V1_1`, `SERIAL_PORT 1`, no USB `SERIAL_PORT_2`
  - 1 extruder, TEMP_SENSOR_0 = 1, others 0, `BED_MAXTEMP 100`
  - PID 19.00 / 1.29 / 70.09, `EXTRUDE_MAXLENGTH 120`
  - TMC5160 on X, Y, Y2, Z, E0; `ENDSTOP_NOISE_THRESHOLD 3`
  - steps 400/400/400/400, feedrate 133/50/50/100, accel 700/700/100/10000, default accels 700 / 10000 / 700
  - direction inversions: Y false, Z true, E0 true
  - `Z_MAX_POS 100`, homing 10 mm/s on all axes
  - preheat fan 255, no LCD feedback beep
  - FYSETC_MINI_12864_2_1 display with NeoPixel (NEO_GRB, 3 px, brightness 255)
  - `FAST_PWM_FAN`
- **`Configuration_adv.h`**
  - controller fan on FAN3 (speed 70, idle 3 s, editable/menu)
  - `FAST_PWM_FAN_FREQUENCY 78100`, E0 auto-fan on PA8
  - `Y_DUAL_STEPPER_DRIVERS`, `QUICK_HOME`
  - TMC: 1200 mA on X/Y/Y2/Z, 800 mA on E0, Rsense 0.075, 16 microsteps, Y2 interpolate
  - spreadCycle on XY, `CHOPPER_DEFAULT_24V`
  - `DIRECT_PIN_CONTROL`, `PINS_DEBUGGING`
- **Pins file (`pins_BTT_OCTOPUS_V1_common.h`)**
  - MOTOR 3 is used as Y2: the fork renamed its Z2 pins to Y2
  - FAN0/1/2 removed from the M106 fan list
  - NeoPixel pin override for the FYSETC 2.1 display
  - In 2.1.x, prefer doing these from the config (`Y2_*_PIN`, `FAN*_PIN` overrides) rather than editing the pins file, if the pins file allows `#ifndef` overrides.
- **`lcd/dogm/marlinui_DOGM.cpp`**: contrast forced to 255. In 2.1.x, set `LCD_CONTRAST_INIT`/`MAX` in config instead.

Bring the machine up as a printer/mill on 2.1.x first (homing, moves, extruder, temps) **before** enabling the laser.

### 3.2 Laser settings (`Configuration_adv.h`, Spindle & Laser section)

Option names below are from 2.0.9. **Map them to the 2.1.x names at port time**: for example `SPINDLE_LASER_PWM` became `SPINDLE_LASER_USE_PWM`, and inline/dynamic power is restructured.

```cpp
#define LASER_FEATURE
#define SPINDLE_LASER_ENA_PIN       PD12    // FAN2- → PSU TL / diode gate
#define SPINDLE_LASER_PWM_PIN       PE5     // FAN1- → PSU IN / diode TTL (TIM9, own timer)
#define SPINDLE_LASER_ACTIVE_STATE  HIGH    // pin HIGH → MOSFET on → TL low → fire
#define SPINDLE_LASER_USE_PWM               // (2.0.9: SPINDLE_LASER_PWM true)
#define SPINDLE_LASER_PWM_INVERT    true    // MOSFET + pull-up inverts the signal
#define SPINDLE_LASER_FREQUENCY     20000   // PSU manual: IN PWM ≥ 20 kHz
#define CUTTER_POWER_UNIT           PWM255  // LightBurn S-value max = 255

// Inline power (D3): in 2.1.x, M3 = constant inline, M4 = dynamic inline,
// power taken from S on G1/G2/G3. Enable the 2.1.x equivalents of:
//   LASER_POWER_INLINE, LASER_POWER_INLINE_TRAPEZOID, LASER_MOVE_POWER
// and consider LASER_SAFETY_TIMEOUT_MS (turns the laser off if motion stalls).

#define AIR_ASSIST                          // D5
#define AIR_ASSIST_ACTIVE           HIGH
#define AIR_ASSIST_PIN              PD14    // FAN4
#define AIR_EVACUATION                      // D5
#define AIR_EVACUATION_ACTIVE       HIGH
#define AIR_EVACUATION_PIN          PD15    // FAN5
```

Also remove FAN4 and FAN5 from the M106 fan list, so Marlin does not initialise them as fans.

### 3.3 LightBurn device settings (D4)

- **Device type:** Marlin. **S-value max:** 255.
- **Check a generated file** before the first job: power should be on `G1 … S…` (inline), with `M3`/`M4` and `M5` around jobs.
- **Air assist:** LightBurn's per-layer air assist emits M8/M9, if enabled in device settings.
- **Two device profiles,** with their own min/max power and speeds:
  - **CO2:** min power just above tube ignition.
  - **Diode.**

---

## 4. Build and flash

1. Before flashing, connect to the current firmware and send `M503`. Save the output to `My setup/M503 backup <date>.txt` (D11).
2. Build env `BIGTREE_OCTOPUS_V1` (or the 2.1.x env name for the Octopus V1.1 board) in PlatformIO.
3. Copy `firmware.bin` to the SD card and power-cycle. Use the bootloader method, not DFU, so the bootloader stays intact.
4. `M502` then `M500` to initialise EEPROM. Then re-apply any value from the backup that differs from the new config, and `M500`.

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
9. **Motion:** a square at low power with inline `G1 … S…`, then a small LightBurn job.
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

1. Back up the current firmware settings (`M503`).
2. Port the config to Marlin 2.1.x, build, flash, and verify the machine works as before.
3. Add the laser config (§3.2), build and flash.
4. Build the CO2 harness (R1, R2 site, connector), remove the FAN1/FAN2 jumpers, install the mA meter.
5. Run §5 steps 1–9 for the CO2.
6. Wire the air-assist and exhaust relays/SSRs on FAN4/FAN5.
7. Build the diode adapter + harness; run §5 step 10.
8. Set up the LightBurn device profiles (CO2, diode).
