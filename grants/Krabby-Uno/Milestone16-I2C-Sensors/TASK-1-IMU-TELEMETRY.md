# Task 1 — IMU on the primary MCU, in the telemetry stream

Goal: Add a SparkFun BMI270 6-axis IMU to the **primary MCU only**, read it every telemetry tick, and append accel/gyro/temperature to the **existing** Arduino→Orin telemetry that already flows today. The locomotion model needs body-frame inertial data delivered on the same low-latency serial path as joint state — not fused on the Orin from the ZED. This is firmware + SDK-parse work; no new transport and no new CLI command.

Outputs
- BMI270 wired to the leader Mega's I2C bus via a Qwiic→Dupont adapter and read in `firmware/arduino/arduino.ino`.
- A new `firmware/arduino/sensors_config.h` holding all I2C addresses, pin notes, the IMU axis→body transform, and poll/telemetry cadence.
- An `IMU` segment appended to the leader's telemetry line; `firmware/interfaces/joint_telemetry.py` extended to parse it; the existing telemetry print/GUI updated to show it.
- Gyro/accel calibration captured at boot and persisted in EEPROM.
- Short doc (in the firmware README) covering BMI270 address, axis convention, and the sensor→body transform.

Acceptance Criteria
- **1a** — BMI270 connected to the **primary (leader) Mega only**, powered from **3.3 V**, on `SDA=D20 / SCL=D21`, via a Qwiic→Dupont adapter. Followers are untouched.
- **1b** — Firmware brings up I2C with `Wire.begin()` at **100 kHz** and initializes the BMI270; init failure is logged, sets `imu_valid=0`, and does **not** crash or stall the gait loop.
- **1c** — Accel (3-axis), gyro (3-axis), and die temperature are read once per telemetry tick with no measurable change to loop timing (verify against the timing test, §6).
- **1d** — A new `IMU` segment is appended to the leader's telemetry line (schema in §4); old parsers ignore it (append-only, backward compatible).
- **1e** — `joint_telemetry.py` parses the `IMU` segment into a dedicated structure; `firmware/krabby_mcu.py` stores it; `format_compact`/the GUI show it.
- **1f** — No `controller_role` field is added to telemetry (Milestone 14 / James owns role-in-telemetry); the existing `FRONT;`/`LEFT;`/`RIGHT;` prefix is unchanged.
- **1g** — Gyro zero-rate bias (and optional accel offset) captured at boot while stationary, stored in EEPROM in a non-colliding region with a magic sentinel, and reused across reboots.
- **1h** — No `krabby-cli telemetry --imu` command is introduced; the IMU values appear in the telemetry that is already printed/streamed.
- **1i** — Firmware README documents the BMI270 I2C address, axis convention, and sensor→body transform.

---

**NOTE:** Exact line numbers below are from the repo at the time of writing and may drift; treat them as pointers to the right function, not literal addresses.

---

## 1. Hardware and wiring

The primary MCU is the **Arduino Mega 2560 + Krabby-Uno shield** that wins role election as `ROLE_FRONT` (the leader, on USB to the Orin — see `firmware/arduino/arduino.ino` `determineRole()`, ~L149–234). Only this board gets the IMU; the LEFT/RIGHT followers are unchanged. We do not duplicate the sensor cluster on every Arduino.

The Mega has **no Qwiic connector**. Cut/adapt the Qwiic cable to a 4-pin Dupont header and land it on the shield:

| Qwiic wire | Mega pin | Note |
|---|---|---|
| VCC (red) | **3.3V** | BMI270 is a 3.3 V part — do **not** use 5 V |
| GND (black) | GND | |
| SDA (blue) | **D20** | Mega hardware I2C SDA |
| SCL (yellow) | **D21** | Mega hardware I2C SCL |

D20/D21 are otherwise unused in `firmware/arduino/board_pins.h` (PWM is D2–D13, EN is D22–D28, the follower UARTs are Serial1 on D18/19 and Serial2 on D16/17). The BMI270 daisy-chains to the OLED (Task 2) and the two INA228s (Task 3) on the same bus.

- Part: **SparkFun 6DoF IMU Breakout — BMI270 (Qwiic)**, default I2C address **0x68** (0x69 via the address jumper).
- **Bus speed: 100 kHz.** There is no need for 400 kHz — the model runs at ~50 Hz and the per-tick I2C payload is tiny; 100 kHz buys noise margin and tolerance for the Dupont run.

## 2. Firmware: bring up the bus and the driver

Library: **`SparkFun_BMI270_Arduino_Library`** (Arduino Library Manager → "SparkFun BMI270", or add to the arduino-cli lib list used by `firmware/Makefile` / CI). Repo: <https://github.com/sparkfun/SparkFun_BMI270_Arduino_Library>. Hookup guide: <https://docs.sparkfun.com/SparkFun_Qwiic_6DoF_BMI270/>.

API shape (from the library examples):

```cpp
#include <Wire.h>
#include "SparkFun_BMI270_Arduino_Library.h"

BMI270 imu;
// in setup(), only on the leader:
Wire.begin();
Wire.setClock(100000);              // 100 kHz, per sensors_config.h
if (imu.beginI2C(BMI270_ADDR) != BMI2_OK) { imuValid = false; }

// once per telemetry tick:
imu.getSensorData();                // must be called before reading
// imu.data.accelX/Y/Z  (g),  imu.data.gyroX/Y/Z  (deg/s)
```

Integration points in `firmware/arduino/arduino.ino`:

- **`setup()`** (~L236): after role election, **only if `currentRole == ROLE_FRONT`**, call `Wire.begin()`, set the clock, and `imu.beginI2C()`. Followers skip all IMU code. (Wire is not currently included anywhere — add `#include <Wire.h>`.)
- **Telemetry block in `loop()`** (~L426–433): this is where the leader prints `roleName(currentRole)` + `"; "` + `actuatorManager->printTelemetry(...)` every `TELEMETRY_INTERVAL_MS`. Read the IMU here (or just before) and append the `IMU` segment (§4) to the same line, leader-only.

Convert to SI before shipping, or document the units you ship: accel `g → m/s²` (×9.80665), gyro `deg/s → rad/s` (×π/180). Apply the sensor→body axis transform (mounting orientation) here and document it (§5, 1i).

## 3. `sensors_config.h` (new)

Create `firmware/arduino/sensors_config.h` as the single home for the wiring/contract constants. Move the existing `const int TELEMETRY_INTERVAL_MS = 50;` out of `arduino.ino` (~L105) into this header. Include:

- I2C addresses (BMI270 `0x68`; OLED, INA228×2 added in later tasks).
- SDA/SCL pin notes (D20/D21) and bus clock (100000).
- IMU axis→body transform constants.
- Poll/telemetry cadence (`TELEMETRY_INTERVAL_MS`).

> The current cadence is **50 ms (20 Hz)**. The Orin control loop runs at 100 Hz (`hal/server/jetson/main.py` `CONTROL_RATE_HZ = 100.0`) and the model at ~50 Hz. The IMU rides the existing 20 Hz telemetry tick. **If the model needs faster proprioception than 20 Hz, lower `TELEMETRY_INTERVAL_MS` here as an explicit, documented decision** — don't change it silently.

## 4. Telemetry schema — append an `IMU` segment

Today the leader prints (see the wire-format comment in `firmware/arduino/actuator_manager.h` ~L192–194, and `ActuatorManager::printTelemetry` ~L316–324):

```
FRONT; <name> <pos> <pot> <current> <enL> <enR> <pwmL> <pwmR> <saf>; <name> ...
```

Append one tagged segment to the **leader's own line only** (not to the forwarded LEFT/RIGHT lines):

```
;IMU <accel_x> <accel_y> <accel_z> <gyro_x> <gyro_y> <gyro_z> <temp_c> <valid>
```

| field | type | unit |
|---|---|---|
| accel_x/y/z | float | m/s² (or document g) |
| gyro_x/y/z | float | rad/s (or document deg/s) |
| temp_c | float | °C |
| valid | 0/1 | 1 = fresh reading, 0 = sensor not responding |

This is append-only and backward compatible: `JointTelemetry.from_tokens()` (`firmware/interfaces/joint_telemetry.py` ~L25) requires exactly 9 tokens and returns `None` otherwise, so existing parsers silently drop the `IMU` segment.

**Parser change** (`firmware/interfaces/joint_telemetry.py`):
- `parse_line()` (~L47) splits on `;`. Add recognition for a segment whose first token is `IMU` and parse it into a new `@dataclass ImuTelemetry`. Keep `ROLE_PREFIXES` (~L23) unchanged.
- `firmware/krabby_mcu.py` `_parse_joint_line()` (~L157) currently stores joints in `self.joints`; add an `self.imu` attribute populated from the parsed `IMU` segment. The reader dispatch (`_reader_loop`, ~L135) already routes any line starting with a role prefix to `_parse_joint_line`, so the leader's extended line is handled with no transport change.

**Display change (this is the "print telemetry" Fletcher means — not a new CLI):**
- `firmware/interfaces/joint_telemetry.py` `format_compact()` (~L60) is the compact debug formatter used by the SDK's DEBUG log (`krabby_mcu.py` ~L166–174). Extend the debug log to also print the IMU line.
- `firmware/gui/app.py` is the existing telemetry GUI. Add an IMU readout (roll/pitch or raw accel/gyro) near the header (`_build_ui` ~L82, `_poll_telemetry` ~L143).

## 5. Calibration (EEPROM)

The BMI270 needs a gyro zero-rate (bias) offset; an accel offset helps too. Capture at boot while the robot is known-stationary (average N samples), subtract going forward, and persist.

EEPROM map (from `arduino.ino` ~L25–27 and `actuator_manager.h` `CalData` ~L356): bytes **0–25** = joint `CalData`, bytes **32–33** = board role (magic `0xAB`). **Allocate IMU calibration at byte 40+**, with its own magic sentinel and a `schema_version` byte, following the `EEPROM.put/get` pattern in `ActuatorManager::saveCalibration`/`loadCalibration` (~L518/L531). Document the axis convention and the sensor→body transform in the firmware README (1i).

## 6. Verify

- **Timing (1c):** the firmware already warns on loop overrun on the Orin side; on the MCU, confirm the added I2C read doesn't push the loop past its budget (the existing integration timing test is `tests/integration/test_timing.py`). Capture before/after loop timing.
- **End-to-end (1d/1e):** with the leader on USB, run the GUI (`python -m firmware.gui`) or enable `KRABBY_MCU_RAW_RX=1` to dump raw lines, and confirm the `IMU` segment appears and parses; move the robot and watch accel/gyro change.
- **Calibration (1g):** power-cycle and confirm the stored bias is reloaded (gyro reads ~0 at rest without re-running the boot capture).

## 7. Deliverable checklist
- [ ] BMI270 on the leader only, 3.3 V, D20/D21 via Qwiic→Dupont; 100 kHz bus.
- [ ] `Wire`/BMI270 brought up in `setup()` leader-only; init failure handled (`imu_valid=0`, no crash).
- [ ] `sensors_config.h` created; `TELEMETRY_INTERVAL_MS` moved into it.
- [ ] `IMU` segment appended to the leader line; backward-compatible.
- [ ] `joint_telemetry.py` parses `IMU`; `krabby_mcu.py` stores it; `format_compact`/GUI show it.
- [ ] No `controller_role` field added; no new CLI command added.
- [ ] Gyro/accel calibration captured at boot and persisted to EEPROM (byte 40+, sentinel + version).
- [ ] Firmware README updated with address, axis convention, transform; loop-timing before/after captured.
