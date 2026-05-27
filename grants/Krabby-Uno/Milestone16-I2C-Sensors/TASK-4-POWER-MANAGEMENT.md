# Task 4 — Low-power shutdown, auto-recovery, and Orin power control

Goal: Use the Task 3 battery measurements to protect the pack. The firmware runs a voltage state machine that gracefully shuts the robot down on low voltage, signals the Orin, sleeps, and auto-recovers when the pack rebounds — plus a one-way over-voltage cutout. This task **also includes the Orin-side work**: a power daemon that listens for the shutdown signal and cleanly powers off, started by the same entry point that brings up the rest of the stack, and the hardware path to actually power the Orin back on (which does not exist today).

Outputs
- A five-threshold power state machine in firmware, with constants in `firmware/arduino/sensors_config.h`.
- Power-state messages (`POWERING_DOWN`, `SHUTDOWN_ACK`, `RESUMING`, `OVER_VOLTAGE_SHUTDOWN`) defined in `firmware/interfaces/`.
- A dedicated low-power Arduino mode that polls only the Pack INA228, blinks the Task 2 red LED + a periodic OLED dead-battery splash, and auto-resumes.
- Orin power-control hardware (MCU-driven supply switch and/or PWR_BTN optocoupler) so `RESUMING` can actually boot the Orin, with a 60 s no-response force-off.
- An Orin-side power daemon, started alongside the HAL server + inference client by the production entry point.
- Appendix-C thresholds validated against the real M12 battery pair.

Acceptance Criteria
- **4a** — Five thresholds (WARN, SOFT_CUT, HARD_CUT, RECOVERY, OVER_VOLT) are named constants in `sensors_config.h`, tunable without touching unrelated code.
- **4b** — `POWERING_DOWN` (with reason code), `SHUTDOWN_ACK`, `RESUMING`, `OVER_VOLTAGE_SHUTDOWN` defined with a `schema_version` byte and committed under `firmware/interfaces/`.
- **4c** — SOFT_CUT: send `POWERING_DOWN`, park the robot (belly down, then drop EN lines), wait ≤ 60 s for `SHUTDOWN_ACK` or timeout, then sleep.
- **4d** — HARD_CUT: immediately drop motor EN and sleep — no park, no handshake.
- **4e** — OVER_VOLT: send `OVER_VOLTAGE_SHUTDOWN`, drop motors, one-way sleep with no auto-recovery.
- **4f** — Low-power state polls **only the Pack INA228** (no IMU, no normal OLED UI); recovery checked ~every 30 s with ≥ 0.4 V hysteresis above SOFT_CUT.
- **4g** — Every ~10 s in low-power state the firmware splashes a dead-battery icon on the OLED and blinks the Task 2 red LED; below the HARD_CUT floor it **stops** splashing the OLED and relies on the LED.
- **4h** — Appendix-C thresholds validated against the actual M12 pair (2× 12 V / 100 Ah LiFePO4) and updated; resting-vs-loaded difference noted.
- **4i** — Orin power-control hardware built (MCU-driven supply switch preferred, and/or PWR_BTN optocoupler) so `RESUMING` powers the Orin; if the Orin doesn't ack or stop telemetry within 60 s of a shutdown command, the MCU force-cuts it. Wiring documented.
- **4j** — Orin-side power daemon implemented (listens for `POWERING_DOWN`, sends `SHUTDOWN_ACK`, runs a clean poweroff) and started by the production entry point alongside HAL server + inference client.
- **4k** — Bench test with an adjustable lab PSU: ramp down through SOFT_CUT and back up through RECOVERY, and separately through OVER_VOLT; timing observed and documented.

---

## 1. State machine (firmware)

Thresholds are named constants in `firmware/arduino/sensors_config.h` (4a). On every tick, read `pack_v` (Task 3) and drive transitions:

- **WARN** — set the warn `power_state`, telemetry only, no behavior change.
- **SOFT_CUT** — begin graceful shutdown (§2).
- **HARD_CUT** — immediate emergency stop (§2).
- **OVER_VOLT** — one-way protective cutout (§2).
- **RECOVERY** — auto-resume gate out of low-power sleep, with ≥ 0.4 V hysteresis above SOFT_CUT to stop chatter.

Set `power_state` in the `BATT` frame (Task 3 §4) so the Orin sees the current state.

## 2. Shutdown paths

**SOFT_CUT (4c):**
1. Send `POWERING_DOWN` (reason `under_voltage_soft`) up the existing serial link.
2. Park: lower the body to its belly, then de-energize — reuse the de-energize path in `ActuatorManager::holdAll()` (`firmware/arduino/actuator_manager.h` ~L304–314, which drops EN + PWM on every actuator).
3. Wait up to **60 s** for `SHUTDOWN_ACK` from the Orin, or timeout.
4. Enter low-power sleep (§3).

**HARD_CUT (4d):** skip park + handshake; immediately drop motor EN and sleep. The Orin discovers it out of band (telemetry stops).

**OVER_VOLT (4e):** send `OVER_VOLTAGE_SHUTDOWN`, drop motors, enter a one-way sleep with no auto-resume (manual reset). Protects against a charger/BMS fault over-charging the pack.

## 3. Low-power state — a dedicated Arduino mode (4f/4g)

This is a special mode, not the normal loop. **Do not poll the IMU and do not run the normal OLED UI.** The Mega wakes on a timer, reads the **Pack INA228 only**, decides, and sleeps:

- **Recovery check (~30 s):** read `pack_v`; if `> RECOVERY` (≥ 0.4 V above SOFT_CUT), resume (§4).
- **Low-battery indication (~10 s):** briefly wake, splash a dead-battery icon on the OLED and blink the Task 2 red LED. **Once `pack_v` falls below the HARD_CUT floor, stop splashing the OLED** (it drains an already-critical pack) and rely on the red-LED blink alone, or go fully dark.

(Use a timer/`millis()` cadence; a true MCU sleep is a nice-to-have but the behavioral contract is "only INA polling + indicator, nothing else.")

## 4. Resume + the Orin power-on problem (researched)

On recovery: send `RESUMING` (reason `voltage_recovered`), restore normal polling/OLED/motors.

**But `RESUMING` does nothing to the Orin today — there is no way for the firmware to soft-power-on the Orin.** The Orin is a **Seeed reComputer J4012** (Jetson Orin NX 16 GB) on the **J401 carrier**. Research:

- The J401 has a **12-pin control header** exposing the power button, reset, and recovery pins, plus a button header that governs auto-power-on behavior. ([J401 getting-started wiki](https://wiki.seeedstudio.com/recomputer_jetson_robotics_j401_getting_started/), [J401 schematic PDF](https://files.seeedstudio.com/wiki/J401/reComputer_J401_SCH_V1.0.pdf))
- You **cannot back-power the board through the 40-pin GPIO** (insufficient current). ([Seeed GPIO wiki](https://wiki.seeedstudio.com/reComputer_Jetson_GPIO/))
- The board can **auto-power-on when DC is applied** (default behavior on the Orin dev kit; selectable via the button-header jumper). ([NVIDIA forum](https://forums.developer.nvidia.com/t/how-to-enable-auto-power-on-for-jetson-orin-nano-developer-kit-no-jumper/347245))

So to make `RESUMING` actually boot the Orin, add hardware — preferred order:

1. **Switched supply + auto-power-on (recommended, robust).** Enable J401 auto-power-on, and feed the Orin's 9–19 V supply through a **MCU-controlled high-side MOSFET / solid-state relay** (this is the "power switch" — conceptually the soft dip switch). The MCU cuts the Orin supply on shutdown and restores it on `RESUMING`; the Orin boots itself when power returns. This also gives a clean hard power-off.
2. **PWR_BTN optocoupler (graceful path).** Drive an **optocoupler across the J401 power-button pins** from a Mega GPIO: a momentary pulse = soft power-on / graceful power-off request; a **≥ 10 s hold = force-off**.

> Confirm exact J401 control-header pins against the schematic before wiring — do not guess pin numbers. Use a spare Mega GPIO (verify against `firmware/arduino/board_pins.h`, same as the Task 2 LED pin selection).

**Force-off rule (4i):** if the MCU has commanded shutdown and the Orin neither sends `SHUTDOWN_ACK` nor stops telemetry within **60 s**, the MCU hard-cuts the Orin — supply switch off (option 1) or ≥ 10 s PWR_BTN hold (option 2). Build whichever of (1)/(2) is feasible; (1) is preferred. Document what was wired.

## 5. Power-state messages (4b)

Define in `firmware/interfaces/` alongside `joint_telemetry.py`, each with a leading `schema_version` byte:

| Message | Direction | Reason codes |
|---|---|---|
| `POWERING_DOWN` | Mega → Orin | `under_voltage_soft`, `over_voltage`, `manual` |
| `SHUTDOWN_ACK` | Orin → Mega | (ack) |
| `RESUMING` | Mega → Orin | `voltage_recovered` |
| `OVER_VOLTAGE_SHUTDOWN` | Mega → Orin | (informational) |

These travel on the same serial link as telemetry. The leader emits them; the Orin reader must recognize them (§6).

## 6. Orin-side power daemon (4j) — part of this milestone

Today the production process is `python -m hal.server.jetson.main` (`hal/server/jetson/main.py`), launched by `krabby run` (`krabby/__main__.py` → `krabby/run.py` → `krabby/_docker.py run_cmd`, which `docker run`s the locomotion image `--privileged` with `-v /dev:/dev`). That one process already runs **HAL server + HAL client + the model inproc**, and spawns optional threads the same way we'll add the daemon: the data collector (`start_collector_thread`, main.py ~L276) and teleop signaling (`start_hal_teleop_signaling_thread`, ~L209).

Add the power daemon as another such component started from `hal/server/jetson/main.py` so a normal boot (and an auto-power-on cold boot) brings the whole stack up together:

1. **Parse the messages.** The MCU serial is read by `firmware/krabby_mcu.py` `KrabbyMCUSDK._reader_loop` (~L111–155); its dispatch (~L135–141) currently handles telemetry-prefixed lines and `VER ` lines. Add recognition of `POWERING_DOWN` / `RESUMING` / `OVER_VOLTAGE_SHUTDOWN` and a registered callback. The actuator SDK on the Orin (`hal/server/jetson/krabby_mcusdk.py`) wraps this same `FirmwareKrabbyMCUSDK` (its `self._mcu`, ~L67), so the daemon can hang off the existing serial reader rather than opening a second port (two readers on one serial device is a bug to avoid).
2. **The daemon.** On `POWERING_DOWN`: send `SHUTDOWN_ACK` back over serial, then perform a clean OS poweroff. **Implementation note / open item:** the process runs *inside* the locomotion container; a plain `systemctl poweroff` won't reach the host without host systemd/D-Bus access. Solve this explicitly — host-systemd passthrough, a small host-side helper the container signals, or rely on the MCU's supply cut (§4) after the ack/flush. Document the chosen mechanism.
3. **Start it.** Add a thread/handle in `main.py`'s setup block (next to the collector/teleop threads) and tear it down in the `finally` block (~L336). Wire any new flag through `krabby run` if needed.

## 7. Appendix C — voltage thresholds (validate against the real pack) (4h)

Pack = 2× 12 V LiFePO4 in series (each internally 4S → **8S**). Validate against the **M12 pair (2× 12 V / 100 Ah LiFePO4 drop-ins)**: each BMS charges to ~14.6 V (3.65 V/cell), rests fully charged ~13.4 V, nominal 12.8 V, and hard-disconnects near ~10 V (2.5 V/cell). LiFePO4's discharge curve is very flat, so thresholds cluster near the knee. **Resting vs. loaded differs by pack internal-resistance × current — measure both.**

| Threshold | Pack V | Per-cell | Notes |
|---|---|---|---|
| Full charge (charging) | 29.2 V | 3.65 V | charger connected |
| Resting full | ~26.8 V | ~3.35 V | no load/charger |
| Nominal | 25.6 V | 3.20 V | mid-SoC resting |
| **WARN** | 24.8 V | 3.10 V | ~20–30% SoC; telemetry only |
| **SOFT_CUT** | 24.0 V | 3.00 V | begin graceful shutdown (~10% SoC) |
| **RECOVERY** | 26.4 V | 3.30 V | ≥ hysteresis above SOFT_CUT; auto-resume |
| **HARD_CUT** | 22.4 V | 2.80 V | margin above the ~20 V internal-BMS cutoff |
| **OVER_VOLT** | 29.6 V | 3.70 V | one-way; charger/BMS fault |

Change vs. the earlier draft: WARN lowered 25.6 V → **24.8 V** (the old value sat at nominal and would warn almost constantly on a flat LiFePO4 curve); added explicit resting-full and nominal rows; noted HARD_CUT margin above the internal BMS cutoff. **Confirm on the bench against the real batteries under load before relying on these.**

## 8. Bench test (4k)
On an adjustable PSU (current-limited): ramp voltage down through WARN → SOFT_CUT (observe park + `POWERING_DOWN` + 60 s ack window + sleep), then up through RECOVERY (observe `RESUMING` + Orin power-on via the §4 hardware). Separately drive above OVER_VOLT (observe one-way cutout). Verify the ~30 s recovery poll and ~10 s LED/OLED indicator cadence, and the 60 s force-off. Record observed timings.

## 9. Deliverable checklist
- [ ] Five thresholds in `sensors_config.h`; messages defined in `firmware/interfaces/` with `schema_version`.
- [ ] SOFT_CUT (park + ack/timeout + sleep), HARD_CUT (immediate), OVER_VOLT (one-way) implemented.
- [ ] Low-power mode polls only Pack INA228; 30 s recovery poll w/ hysteresis; 10 s LED + OLED dead-battery splash, stops below HARD_CUT.
- [ ] Orin power-control hardware built (supply switch and/or PWR_BTN optocoupler); 60 s no-response force-off; wiring documented.
- [ ] Orin power daemon implemented (ack + clean poweroff) and started by `hal/server/jetson/main.py` alongside HAL server + client + model; container→host poweroff mechanism documented.
- [ ] Appendix-C thresholds validated against the real pack and updated; resting-vs-loaded noted.
- [ ] PSU bench test run end-to-end; timings recorded.
