# Task 2 — GUI, HAL, gamepad, and model on --stm

Goal: Connect the GUI, the HAL, the gamepad, and the AI model to the CAN SDK with an `--stm` option, and confirm each one moves the simulated robot over `vcan0`. Without the option, every tool behaves as it does today.

Outputs
- `python -m firmware.gui --stm`, with `firmware/gui/stm_adapter.py`.
- `hal/server/jetson/krabby_stm_mcusdk.py`, which sends duty to the CAN SDK.
- An `--stm` option on the Jetson HAL server and the gamepad and model mappers, with a matching setting in `krabby run`.
- A `can` firmware transport for the Isaac HAL's `--backend fullsim`.
- A release image that includes the CAN packages and the STM CLI.
- A task report covering one session each with the GUI, gamepad, and AI model against the sim.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-2---gui-hal-gamepad-and-model-on---stm).)

---

Note: line numbers below point at the right function at the time of writing and may drift.

---

## 1. What changes

The gamepad and AI model send `JointCommand` messages to the HAL server over ZMQ, and only the HAL server talks to the firmware. The chain is described in `controller/scripts/jetson/E2E_GAMEPAD_KRABBY.md`.

Today the values in `JointCommand` are called radians, but they act as duty. The HAL turns each one into jog PWM with `PWM = rad * 255 / 0.5` (`_rad_to_pwm` in `hal/server/jetson/krabby_mcusdk.py` ~L29), and the gamepad mapper scales full stick to 0.5 for that reason (`controller/mappers/gamepad_to_krabby_hal_mapper.py` ~L45). The model mapper (`compute/parkour/mappers/model_to_hardware.py`) copies the model's raw action straight through. On the `--stm` path, `JointCommand` carries duty from -1.0 to 1.0 for each joint, and nothing converts radians.

Three places create the SDK and need the `--stm` choice: the GUI at `firmware/gui/app.py` ~L295, the Jetson HAL server at `hal/server/jetson/hal_server.py` ~L195, and the Isaac HAL's fullsim backend in `hal/server/isaac/hal_server.py` (`_initialize_mcusdk`, ~L348). The keyboard jog menu in `firmware/__main__.py` stays on serial, and `krabby-firmware-stm duty` replaces it for v0.3 boards.

Joint names such as "front left hip yaw" exist only in the GUI and the robot definition. The CAN SDK and the firmware use leg and joint IDs only.

## 2. GUI

The GUI is built around the Mega SDK's interface: joint names like `FLHY`, a `joints` map of `JointTelemetry`, and methods like `send_command_jog`. To keep GUI changes small, add `firmware/gui/stm_adapter.py`, a thin class with that same interface on top of the CAN SDK. All the name translation lives in this one file.

- `send_command_jog(name, pwm)` converts `FLHY` to leg 1, joint 1, and `pwm` to a duty by dividing by 255. The SDK keeps every other joint's last duty, so jogging one joint leaves the rest unchanged.
- `send_command_joints_hold()` sends zeros to all six legs.
- `joints` returns `JointTelemetry` values with `pot` and `saf` (the hall count) filled in, `pwm` converted back from duty, `pos` set to the calibrated position from telemetry, and `current` set to 0.
- `imu` and `battery` are `None`. `role_last_seen` is empty, and the GUI's per-board freshness display stays blank.
- `last_feedback_ts`, `last_error`, `target`, `connect()`, and `close()` behave as they do in the Mega SDK.

In `firmware/gui/__main__.py`, add `--stm`, which makes `app.py` create the adapter. Hide the position target control in this mode, because the firmware has no position command. The calibrate control calls `send_calibration()` for the selected joint.

## 3. HAL

Add `KrabbyStmMCUSDK` in `hal/server/jetson/krabby_stm_mcusdk.py`, with the same methods as the serial class: `connect`, `is_connected`, `apply_command`, and `close`. `is_connected` returns true once at least one leg has sent telemetry. At startup it builds one table from the robot definition's `mcu_joints` that gives the leg and joint ID for each `JointCommand` index. `apply_command` clamps each value to -1.0 to 1.0, and sends all 18 with one `set_duties()` call. It applies no neutral band and no scaling.

Add an `stm` argument to `JetsonHalServer` and to `create_hal_server()` (`runtime.py` ~L85), and create the matching class at `hal_server.py` ~L195. Importing the new class only when `stm` is true means Mega robots do not need the CAN packages at import time. Add a `--stm` flag to both `main_gamepad.py` and `main_model.py`. `main.py` already passes unknown arguments through to them.

## 4. Gamepad and model mappers

With `--stm`, the gamepad mapper scales full stick to a duty of 1.0 instead of 0.5. The model mapper clamps the model's action to -1.0 to 1.0 and sends it as duty. How a trained model's action becomes duty for the linear actuators is a question for the next model release, and this task only passes the action through.

## 5. Isaac fullsim over CAN

Add `can` to the `--firmware-transport` choices in `hal/server/isaac/command_backend_cli.py`. In this mode `_initialize_mcusdk` attaches the six simulated boards from Task 1 to the env and connects `KrabbyStmMCUSDK` to them over `vcan0`. The HAL server's loop still steps the env, so the boards do not step it themselves. Each step the SDK returns every joint's current duty as the normalized action, the way `IsaacSimFirmwareMCUSDK` returns drive pins today. This lets the gamepad and AI model drive the simulated robot through the same SDK the Orin uses.

## 6. Image and launcher

Add `python-can`, `cantools`, and the `krabby-firmware` version containing `firmware.stm` to `images/locomotion/requirements.release.txt`. Leave `avrdude` and `arduino-cli` in `images/locomotion/Dockerfile.release`, because Mega robots use the same image.

The container needs no new access to the CAN ports. On the Jetson, `krabby/_docker.py` already runs with `--network host` and `--privileged`, so `can0` and `can1` are visible inside it. The host configures the bit rates before the container starts.

Add an `stm` setting to `krabby/_locomotion_config.py` and have `build_hal_argv()` (~L132) add `--stm` when it is true, so a v0.3 robot always starts in this mode. Check that the default gamepad launch in `krabby/run.py` passes the flag as well.

## 7. Tests

`tests/unit/hal/server/jetson/test_krabby_stm_mcusdk.py` checks that a full `JointCommand` produces one `set_duties()` call with each value at the right leg and joint, using `robot_definition_krabby_hex.py`'s `mcu_joints`, and that values beyond ±1.0 are clamped.

`tests/unit/firmware/test_gui_stm_adapter.py` checks the jog and hold translation, and that jogging one joint preserves every other joint's duty.

New tests in `tests/unit/controller/` and `tests/unit/test_compute_model_to_hardware_stm.py` check that the gamepad and model mappers produce duty in `--stm` mode and keep today's output without it.

Against the sim, run the GUI, the gamepad, and the AI model in turn, and confirm the simulated robot responds to each. Record the results in the task report. Running the release image against the real boards happens in the support weeks.
