# Task 4 — I2C sensor simulators (IMU, OLED, battery)

Goal: add simulators for the Milestone 16 I2C sensors — an IMU, the OLED health screen, and the battery-sense chips — and build the simulated `Wire`/I2C layer they sit on, which this task owns and adds to the pin and `millis` shim from Task 1. All three devices use I2C with standard call and response patterns, so the simulator mimics the data already present in M16's unit and integration tests instead of modeling physics. This is a short task requiring little simulation integration or dynamics.

Two dependencies need resolving first. The M16 sensors await implementation in the real firmware: `firmware/arduino/` currently has no `sensors_config.h`, no `#include <Wire.h>`, and no BMI270, SSD1306, or INA228 code, because M16 is spec-complete and code-incomplete. Separately, John's OLED simulator is absent from `krabby-research`, and a repo-wide search finds it nowhere. The real prerequisite for this task is therefore that the M16 firmware exists to be simulated and that John's OLED simulator is vendored in or pointed to. Should the M16 firmware still be incomplete when this task starts, the sim work and the M16 firmware work proceed against the same agreed I2C contract — the addresses and telemetry schemas below — which the M16 grant already specifies.

Outputs
- A simulated `Wire`/I2C layer in the firmware host build supporting the M16 device addresses with standard call and response.
- A simulated BMI270 IMU returning plausible accel, gyro, and temperature, optionally derived from the sim body state, producing the M16 `IMU` telemetry segment.
- The OLED health screen simulator, John's, hooked through the new pin and `Wire` interface and rendering the krab status screen from simulated firmware state.
- A simulated INA228 battery monitor returning plausible pack and per-battery voltage and current, producing the M16 `BATT` telemetry frame.
- Unit and integration tests mirroring the M16 sensor tests, driven through the simulated firmware.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-4---i2c-sensor-simulators-imu-oled-battery).)

---

Note: this task depends on the M16 firmware and John's OLED simulator existing. The addresses and schemas below come from the M16 grant docs, which are the controlling technical reference.

---

## 1. The M16 I2C contract to mimic

From the M16 grant (`patina-foundation-grants/grants/Krabby-Uno/Milestone16-I2C-Sensors/`), the leader FRONT Mega carries a Qwiic chain on `SDA=D20` and `SCL=D21` at 100 kHz:

| Device | I2C addr | Notes |
|---|---|---|
| BMI270 IMU | `0x68` (`0x69` jumper) | SparkFun 6DoF, 3.3 V |
| Qwiic OLED 1.3" (SSD1306, class `Qwiic1in3OLED`) | `0x3D` (`0x3C` jumper) | 128×64, 1-bit mono |
| Pack INA228 | `0x40` | across a 200 A / 75 mV shunt |
| Midpoint INA228 | `0x41` | per-battery voltage |

The simulated firmware brings up `Wire` on the leader alone, and followers skip I2C entirely. The sim `Wire` layer services these addresses with the same call and response the SparkFun and Adafruit libraries expect: register reads for the IMU and INA228, and the SSD1306 command and data stream for the OLED. Because the simulator runs the real driver code, compiled in Task 1's host build, the simulated `Wire` has only to return believable register values while the driver libraries do the parsing.

## 2. Telemetry schemas (from M16)

Both schemas are append-only additions to the leader's existing telemetry line, and `firmware/interfaces/joint_telemetry.py` gains parsers for them under M16.

The IMU segment (M16 Task 1), leader-only:

```
;IMU <accel_x> <accel_y> <accel_z> <gyro_x> <gyro_y> <gyro_z> <temp_c> <valid>
```

The BATT frame (M16 Task 3), leader-only:

```
;BATT <pack_v> <pack_i> <pack_w> <pack_charge> <batt_a_v> <batt_b_v> <divergence_flag> <power_state>
```

These append after the 6 fixed 9-token joint segments, and parsers requiring exactly 9 tokens ignore them, which keeps the format backward compatible. The simulated firmware produces them as soon as the simulated sensors return data and the M16 firmware code path runs.

## 3. The three simulators

The IMU at `0x68` returns accel, gyro, and temperature register values. The cheapest believable source derives accel from the sim base orientation, taking the gravity vector in the body frame, and gyro from base angular velocity; the sim already computes base state for the HAL (`hal/server/isaac/primary_zed_base_state.py`, `isaac_primary_rgbd_base_state`). A static stationary reading with gravity pointing down also satisfies the tests. Honor the M16 calibration and `valid` flag path.

The OLED at `0x3D` runs John's simulator, hooked through the pin and `Wire` interface so it renders from the simulated firmware's live state: controller liveness from role election and telemetry freshness, per-actuator glyphs from `currentPwm` and `isConnected`, and battery bars from the simulated INA228. The requirement that matters is that the simulated firmware drives it — the firmware writes SSD1306 command and data over the sim `Wire`, and the OLED simulator renders what arrives. Locating John's simulator and vendoring it in is the first step.

The battery INA228s at `0x40` and `0x41` return pack and midpoint voltage and current registers. Mirror the values in M16's INA228 tests, with a nominal 25 to 27 V pack and plausible current. These also feed the OLED battery bars, and the thresholds in M16 Task 4's power state machine where that code is present in the firmware. Believable readings are what M21 needs here, and the shutdown behavior belongs to M16.

## 4. Verify

- I2C layer: read each device's ID and registers through the sim `Wire` and assert the expected responses; address an absent device and assert graceful failure.
- IMU: run the simulated firmware and assert the `IMU` telemetry segment appears and parses; tilt the sim body, where the values derive from base state, and assert accel changes.
- OLED: drive the simulated firmware and assert John's simulator renders the health screen reflecting live actuator, controller, and battery state, so that a jogging leg shows the extend glyph and a downed follower shows its third missing.
- Battery: assert the `BATT` frame appears and parses with plausible pack and per-battery values.
- Non-fatal init: remove a simulated sensor and assert the firmware continues running with the invalid flag set.
