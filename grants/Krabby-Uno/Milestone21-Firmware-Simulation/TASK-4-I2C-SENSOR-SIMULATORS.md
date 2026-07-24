# Task 4 — I2C sensor simulators (IMU, OLED, battery)

Goal: Add simulators for the **Milestone 16** I2C sensors — a simulated **IMU**, the **OLED health screen**, and the **battery-sense chip** — by building the **simulated `Wire`/I2C layer** (this task owns it; Task 1 built only the pin/`millis` shim, and there is no `Wire` before this task). All three are I2C with standard call/response patterns, so the sim mimics the data already present in M16's unit/integration tests rather than modeling any physics. This is a short task that requires little simulation integration or dynamics.

> **Two dependencies to resolve first (flagged, important).** As of this writing: (1) the **M16 sensors are not yet implemented in the real firmware** — there is no `sensors_config.h`, no `#include <Wire.h>`, and no BMI270/SSD1306/INA228 code in `firmware/arduino/`; M16 is spec-complete but code-incomplete. (2) **John's OLED simulator is not committed to `krabby-research`** (a repo-wide search finds it nowhere). So this task's real prerequisite is that the M16 firmware exists to be simulated, and that John's OLED simulator is vendored in or pointed to. If M16 firmware is incomplete when this task starts, the sim work and the M16 firmware work proceed against the same agreed I2C contract (addresses + telemetry schemas below), which the M16 grant already specifies.

Outputs
- A simulated `Wire`/I2C layer in the firmware host build supporting the M16 device addresses with standard call/response.
- A simulated **BMI270 IMU** returning plausible accel/gyro/temp (optionally derived from the sim body state), producing the M16 `IMU` telemetry segment.
- The **OLED health screen** simulator (John's, hooked through the new pin/`Wire` interface) rendering the krab status screen from simulated firmware state.
- A simulated **INA228 battery monitor** returning plausible pack/per-battery voltage/current, producing the M16 `BATT` telemetry frame.
- Unit/integration tests mirroring the M16 sensor tests, driven through the simulated firmware.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-4---i2c-sensor-simulators-imu-oled-battery).)

---

**NOTE:** This task depends on the M16 firmware and John's OLED simulator existing; addresses and schemas below come from the M16 grant docs (the controlling technical reference).

---

## 1. The M16 I2C contract to mimic

From the M16 grant (`patina-foundation-grants/grants/Krabby-Uno/Milestone16-I2C-Sensors/`), the leader (FRONT) Mega carries a Qwiic chain on `SDA=D20 / SCL=D21` at **100 kHz**:

| Device | I2C addr | Notes |
|---|---|---|
| BMI270 IMU | `0x68` (0x69 jumper) | SparkFun 6DoF, 3.3 V |
| Qwiic OLED 1.3" (SSD1306, class `Qwiic1in3OLED`) | `0x3D` (0x3C jumper) | 128×64, 1-bit mono |
| Pack INA228 | `0x40` | across a 200 A/75 mV shunt |
| Midpoint INA228 | `0x41` | per-battery voltage |

The simulated firmware brings up `Wire` only on the leader; followers skip all I2C. The sim `Wire` layer must service these addresses with the same call/response the SparkFun/Adafruit libraries expect (register reads for the IMU/INA228; the SSD1306 command/data stream for the OLED). Because the sim runs the *real* driver code (Task 1's host build compiles the firmware), the simulated `Wire` only has to return believable register values — the driver libraries do the parsing.

## 2. Telemetry schemas (from M16)

Both are append-only additions to the leader's existing telemetry line; `firmware/interfaces/joint_telemetry.py` gains parsers for them under M16.

**IMU segment** (M16 Task 1), leader-only:
```
;IMU <accel_x> <accel_y> <accel_z> <gyro_x> <gyro_y> <gyro_z> <temp_c> <valid>
```

**BATT frame** (M16 Task 3), leader-only:
```
;BATT <pack_v> <pack_i> <pack_w> <pack_charge> <batt_a_v> <batt_b_v> <divergence_flag> <power_state>
```

These are appended after the 6 fixed 9-token joint segments and are ignored by parsers that require exactly 9 tokens (append-only, backward compatible). The sim just needs to make the simulated firmware produce them, which happens automatically once the simulated sensors return data and the M16 firmware code path runs.

## 3. The three simulators

- **IMU (`0x68`).** Return accel/gyro/temp register values. Cheapest believable source: derive accel from the sim base orientation (gravity vector in body frame) and gyro from base angular velocity — the sim already computes base state for the HAL (`hal/server/isaac/primary_zed_base_state.py`, `isaac_primary_rgbd_base_state`). A static "stationary + gravity down" reading also satisfies the tests. Honor the M16 calibration/`valid` flag path.
- **OLED (`0x3D`).** Hook **John's OLED simulator** through the pin/`Wire` interface so it renders from the simulated firmware's live state (controller liveness from role election + telemetry freshness, per-actuator glyphs from `currentPwm`/`isConnected`, battery bars from the simulated INA228). The key requirement is that it is driven by the *simulated firmware*, not a standalone display mock — the firmware writes SSD1306 command/data over the sim `Wire`, and the OLED simulator renders that. Locate John's simulator and vendor it in (or reference its repo) as the first step.
- **Battery INA228 (`0x40`/`0x41`).** Return pack and midpoint voltage/current registers. Mirror the values in M16's INA228 tests (a nominal ~25–27 V pack, plausible current). This also feeds the OLED battery bars and, if M16 Task 4's power state machine is in the firmware, its thresholds — but M21 only needs believable readings, not the shutdown behavior.

## 4. Verify
- **I2C layer:** read each device's ID/register through the sim `Wire`; assert expected responses; address an absent device and assert graceful failure.
- **IMU:** run the simulated firmware; assert the `IMU` telemetry segment appears and parses; tilt the sim body (if deriving from base state) and assert accel changes.
- **OLED:** drive the simulated firmware; assert John's simulator renders the health screen reflecting live actuator/controller/battery state (e.g. a jogging leg shows the extend glyph; a downed follower shows its third missing).
- **Battery:** assert the `BATT` frame appears and parses with plausible pack/per-battery values.
- **Non-fatal:** remove a simulated sensor; assert the firmware continues (invalid flag set, no crash).
