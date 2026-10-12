# Handoff

**Snapshot: 2026-10-12.** Update this file at the end of each work session.

## Where we are

Goal: add CO2 and diode laser control (with board-controlled power), air assist and fume extraction to the Octopus CNC. Design: `Laser integration plan.md`.

**Done:**

- **Design and decisions:** written up in the plan (§0, D1–D11). The why behind them is in `Project history.md`.
- **Firmware:** moved to upstream **Marlin 2.1.2.8** in `Firmware/Marlin-2.1.2.8/`.
  - The machine config was ported from the 2.0 fork.
  - Laser, air-assist and extraction settings were added.
  - Commits `c473dc1` (pristine upstream) and `5665f80` (our changes), pushed to `origin/master`.
- **Two images built** (git-ignored `Firmware/Marlin-2.1.2.8/.pio/out/`):
  - `firmware-2.1.2.8-machine-only.bin`: the port only. Flash first.
  - `firmware-2.1.2.8-laser.bin`: port + laser/air.
- **Resolved config checked** with a preprocessor dump of the real build flags (plan §4).

**Not done yet:** nothing has been flashed, and no laser hardware has been built or wired.

## Next steps (on the machine, by the user)

Plan §7, steps 3 onward:

1. **Before flashing:**
   - Check that no DIAG jumpers are fitted on the driver sockets (build warning; physical endstops are used).
   - Copy `FIRMWARE.CUR` off the SD card last used for flashing. It is the rollback image, because the 2.0 project no longer builds.
   - Send `M503` and save the output as `My setup/M503 backup <date>.txt`.
2. **Flash `firmware-2.1.2.8-machine-only.bin`** (copied as `firmware.bin`), then `M502`, `M500`.
   - Verify homing, moves, extruder, temperatures, display and backlight.
   - Re-apply any tuned values from the `M503` backup.
3. **Flash `firmware-2.1.2.8-laser.bin`.** With nothing connected, meter the FAN ports with `M3 S…`/`M5`, `M8`/`M9`, `M10`/`M11`.
4. **Build the CO2 harness** (R1/R2, connector), remove the FAN1/FAN2 jumpers, fit a 0–30 mA meter. Then run plan §5 steps 1–9.
5. **Air-assist and exhaust** relays/SSRs on FAN4/FAN5.
6. **Diode adapter and harness;** plan §5 step 10.
7. **LightBurn device profiles** (CO2, diode); plan §3.3.

## Open questions

| # | Question | Why it matters |
|---|---|---|
| 1 | Is anything wired to FAN1/FAN2 by the **old** assignment in `hw-config.md` (enable FAN1, PWM FAN2)? Was the laser ever used with the 2.0 firmware? | The 2.1 laser build swaps the roles: PWM on FAN1, enable on FAN2. Rewire before flashing the laser image. `hw-config.md` should then be updated or marked as 2.0-only |
| 2 | Diode module model and its maximum PWM frequency | The board drives 20 kHz (needed by the CO2 PSU). Some cheap diode drivers don't follow 20 kHz |
| 3 | CO2 tube rated current | Unknown, so S255 is capped at 16 mA with R2. Read the tube label and adjust |
| 4 | Lid switch + key/arm switch in the WP loop? | Currently only a water flow switch (D8); strongly recommended |
| 5 | Add Marlin's `LASER_COOLANT_FLOW_METER` (Hall sensor on a spare input)? | Optional second interlock, plus a flow readout |
| 6 | Part-cooling fan needed for printing? | The laser build has no `M106` (all six fan ports are used). Option: move the controller fan to an always-on header |
| 7 | Once 2.1 is verified on the machine: delete `Firmware/Marlin-bugfix-2.0.x/`? | Keep until then as reference |

## How to resume

- **Open the repo root** (`C:\dev\hardware\CNC\BIGTREETECH-OCTOPUS-V1.0`) in VS Code and start Claude there. `CLAUDE.md` loads automatically.
- **For PlatformIO IDE work,** open `Firmware/Marlin-2.1.2.8` in a separate window; or build from the command line (see `CLAUDE.md`).
- **Full transcript:** the conversation that produced all of this ran with `Firmware/Marlin-bugfix-2.0.x` as its working folder. It can be reopened from that folder's session history.
