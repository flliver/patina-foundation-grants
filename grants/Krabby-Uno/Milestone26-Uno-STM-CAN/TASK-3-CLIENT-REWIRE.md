# Task 3 — GUI, controller, and model image on --stm

Goal: Control the v0.3 robot from the GUI, the gamepad, and the AI model image using an `--stm` option, and confirm that each still works on a Mega robot without the option.

Outputs
- `python -m firmware.gui --stm`.
- `hal/server/jetson/krabby_stm_mcusdk.py`, the CAN version of the HAL's motor class.
- An `--stm` option on the Jetson HAL server, with a matching setting in `krabby run`.
- A release image that includes the CAN packages and the STM CLI.
- A task report covering one session each with the GUI, gamepad, and AI model.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-3---gui-controller-and-model-image-on---stm).)

---

Note: line numbers below point at the right function at the time of writing and may drift.

---

## 1. What changes

The gamepad and AI model send `JointCommand` messages to the HAL server over ZMQ, and only the HAL server talks to the firmware. They need no code changes. The chain is described in `controller/scripts/jetson/E2E_GAMEPAD_KRABBY.md`.

Two places need the `--stm` choice: the GUI, which creates the SDK at `firmware/gui/app.py` ~L295, and the HAL server, which creates it at `hal/server/jetson/hal_server.py` ~L195. The keyboard jog menu in `firmware/__main__.py` stays on serial, and `krabby-firmware-stm duty` replaces it for v0.3 robots.

## 2. GUI

The GUI is built around the Mega SDK's interface: joint names like `FLHY`, a `joints` map of `JointTelemetry`, and methods like `send_command_jog`. To keep GUI changes small, add `firmware/gui/stm_adapter.py`, a thin class with that same interface on top of the CAN SDK. All the name translation lives in this one file.

- `send_command_jog(name, pwm)` converts `FLHY` to leg 1, motor 0, and `pwm` to a duty by dividing by 255. Because SetDuty carries all three of a leg's motors, the adapter remembers the leg's other two duties and resends them.
- `send_command_joints_hold()` sends zeros to all six legs.
- `joints` returns `JointTelemetry` values with `pot` and `saf` (the hall count) filled in, `pwm` converted back from duty, `pos` set to `pot / 3590`, and `current` set to 0.
- `imu` and `battery` are `None`. `role_last_seen` is empty, and the GUI's per-board freshness display stays blank.
- `last_feedback_ts`, `last_error`, `target`, `connect()`, and `close()` behave as they do in the Mega SDK.

In `firmware/gui/__main__.py`, add `--stm`, which makes `app.py` create the adapter. Hide the position target and calibrate controls in this mode, because the firmware has neither command.

## 3. HAL

Add `KrabbyStmMCUSDK` in `hal/server/jetson/krabby_stm_mcusdk.py`, with the same methods as the serial class: `connect`, `is_connected`, `apply_command`, and `close`. `is_connected` returns true once at least one leg has sent telemetry. Base `apply_command` on the logic in `_jog_all_joints` (`krabby_mcusdk.py` ~L101), which is what production runs today, with two changes:

```python
LEG_ID = {"FL": 1, "FR": 2, "ML": 3, "MR": 4, "RL": 5, "RR": 6}
MOTOR_ID = {"hip_yaw": 0, "hip_pitch": 1, "knee": 2}

def _rad_to_duty(rad: float) -> float:
    if abs(rad) <= NEUTRAL_RAD_THRESHOLD:
        return 0.0
    return max(-1.0, min(1.0, rad / JOINT_LIMIT_RAD))
```

This produces the same output as the serial path, since `_rad_to_pwm` reaches ±255 at the same ±0.5 rad. Each `apply_command` groups the 18 values by leg and calls `set_duty()` six times.

Add an `stm` argument to `JetsonHalServer` (~L139) and to `create_hal_server()` (`runtime.py` ~L85), and create the matching class at `hal_server.py` ~L195. Importing the new class only when `stm` is true means Mega robots do not need the CAN packages at import time. Add a `--stm` flag to both `main_gamepad.py` and `main_model.py`. `main.py` already passes unknown arguments through to them.

## 4. Image and launcher

Add `python-can`, `cantools`, and the `krabby-firmware` version containing `firmware.stm` to `images/locomotion/requirements.release.txt`. Leave `avrdude` and `arduino-cli` in `images/locomotion/Dockerfile.release`, because Mega robots use the same image.

The container needs no new access to the CAN ports. On the Jetson, `krabby/_docker.py` already runs with `--network host` and `--privileged`, so `can0` and `can1` are visible inside it. The host configures the bit rate, as set up in Task 2.

Add an `stm` setting to `krabby/_locomotion_config.py` and have `build_hal_argv()` (~L132) add `--stm` when it is true, so a v0.3 robot always starts in this mode. Check that the default gamepad launch in `krabby/run.py` passes the flag as well.

## 5. Tests

`tests/unit/hal/server/jetson/test_krabby_stm_mcusdk.py` checks `_rad_to_duty` at 0, within the neutral threshold, ±0.25 rad, and beyond ±0.5 rad. It also checks that a full `JointCommand` produces six `set_duty()` calls with each joint in the right leg and position, using `robot_definition_krabby_hex.py`'s `mcu_joints`.

`tests/unit/firmware/test_gui_stm_adapter.py` checks the jog and hold translation, and that jogging one motor preserves the leg's other two duties.

On the v0.3 robot, run the GUI, gamepad, and AI model in turn, and confirm the robot responds correctly to each. Then run the GUI and the same image without `--stm` on a Mega robot to confirm they still work. Record the results in the task report.
