# Task 5 — Swap the Isaac command path to the simulated firmware

Goal: replace the one step in the Isaac backend that turns a `JointCommand` straight into joint position targets, so commands instead travel through krabby-SDK, the simulated firmware control loop, and the simulated pins before becoming the env action. The rest of the Isaac backend stays as it is.

Outputs
- A firmware-backed command SDK that replaces `IsaacSimMCUSDK` in the Isaac server's command path and returns an action tensor of the same shape.
- The Task 3 firmware instances constructed against the env the Isaac server already owns, bound to its 18 joints.
- A selection flag so a launch chooses the direct path or the firmware path, leaving Jetson and Isaac launches untouched.
- An integration test driving a teleop command end to end, plus a fleet registration and teleop check against the existing isaacsim image.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-5---swap-the-isaac-command-path-to-the-simulated-firmware).)

---

Note: line numbers are pointers to the right function and may drift. Publishing a sim image and auto-detecting an absent krab remain M23.

---

## 1. What the Isaac backend does today

`HalServerBase` (`hal/server/server.py` ~L26) is a concrete ZMQ transport rather than an abstract contract: PUB for observations, PULL for commands, with `set_observation(hw_obs)` ~L140 and `get_joint_command(timeout_ms)` ~L184. Each backend subclasses it and supplies its own command path, and selection happens at the packaging and entry-point layer.

The Isaac server's command path runs:

```
isaac/main.py:531   action = hal_server.apply_command()
  |- IsaacSimHalServer.apply_command()          # hal_server.py ~L662, polls the transport
       |- get_joint_command()                    # ZMQ PULL -> JointCommand, radians
       |- IsaacSimMCUSDK.apply_command(command)  # isaacsim_mcusdk.py ~L36, thresholds, returns dict
       |- dict -> ordered array -> torch tensor  # hal_server.py ~L749-767
isaac/main.py:556   obs_dict, ... = env.step(action)
```

`IsaacSimMCUSDK.apply_command` applies a small-angle neutral threshold and returns the command dict unchanged. No firmware and no pins take part, and the env action term converts the values into position targets.

The two backends have different `apply_command` shapes: the Jetson one (`jetson/hal_server.py` ~L661) takes a command and returns None, while the Isaac one takes no argument and returns a tensor. This task keeps the Isaac shape, since the return value feeds `env.step`.

## 2. The swap

The insertion point is the single `IsaacSimMCUSDK.apply_command` call at `hal/server/isaac/hal_server.py` ~L750. The replacement leaves the surrounding code alone and changes what happens between receiving the command and building the tensor:

```
apply_command()
  |- get_joint_command()                     # JointCommand, radians
  |- radians -> firmware normalized or PWM   # as jetson/krabby_mcusdk.py ~L32-38, ~L105
  |- hand to the simulated firmware          # T or B command -> ActuatorManager::updateAll()
  |- firmware writes simulated pins          # driveActuator -> EN, PWM_R, PWM_L (Task 1)
  |- read pin state -> per-joint drive -> tensor
env.step(action)
```

There are two ways to deliver the command to the firmware, and both keep it in the loop.

In-process calls the Task 1 host build directly, feeding the command in, ticking the control loop, and reading the pin writes. It has the fewest moving parts.

Through the serial SDK drives the simulated MCU via `firmware.krabby_mcu.KrabbyMCUSDK` over the virtual serial device from Task 3, exercising the exact krabby-SDK and wire protocol that ship on the robot.

The serial route is the one that proves the production call path, and Task 3 has already built the virtual device it needs. The in-process route is a reasonable first step if serial timing proves awkward inside the step loop.

## 3. Attaching the firmware to the existing env

Task 3 already produces three firmware instances wired to each other over emulated inter-board serial and bound to three disjoint sets of six joints of one crab Articulation. Task 5 constructs them against the env the Isaac server already holds (`IsaacSimHalServer.env`, assigned in `__init__` ~L222) rather than creating a scene.

Joint order comes from `robot_definition.get_joint_names()` and the action width from `get_total_joint_count()`, the same calls the current path makes at ~L749-756, so tensor shape and index mapping stay as they are.

Observations need no work. `set_observation` ~L351, `_cache_references` ~L262, and `get_sensor_interface` ~L254 all read the env and are untouched by this change. If contact or current data is routed through the contact sensor, fix the `self.contacvt_sensor` typo at ~L317 first.

## 4. Selection

`IsaacSimHalServer` chooses its SDK in `_initialize_mcusdk()`, so the selection belongs there: construct either `IsaacSimMCUSDK` or the firmware-backed SDK. Surface the choice as a `--backend fullsim` flag in `hal/server/isaac/main.py`, which is where the server is constructed.

A new sibling package under `hal/server/` is unnecessary here, since the observation half, the transport, and the launch path are all the Isaac ones. Keep the new SDK alongside `isaacsim_mcusdk.py`.

`krabby run` selects a launch entry point rather than a backend (`krabby/run.py` ~L20), and `krabby/_docker.py` ~L125 hard-codes `krabby-hal-server-jetson` for gamepad mode, so a sim launch adds a branch there.

## 5. Fleet check and the M23 boundary

The fleet path already runs for the real robot: the agent (`krabby/agent.py`, IoT Core MQTT shadow reporting, secure tunneling, teleop signaling), enrollment (`krabby/enroll.py`), and the operator-side CLI (`fleet/cli/krabby_fleet_cli/`). The presence signal M23 will use already exists, with `collect_health()` reporting `mcu_present` (`krabby/telemetry.py` ~L101-103) and `reported_image` ~L173.

Task 5 confirms that a simulated krab running the firmware path registers, appears in the UI, connects, and teleoperates. This is the first time the fleet and teleop path runs with firmware in the loop, so expect gaps: fix the small ones and hand the rest to M23.

Deferred to M23: publishing a sim image to ECR or S3 (`images/isaacsim/` builds locally via `make build-isaacsim-image` and has no `publish-isaacsim` workflow, unlike `publish-locomotion.yml`), a `--sim` install flag and absent-krab auto-detect in `krabby/install.py` and `__main__.py`, GPU capability validation, and IoT-based resume. Stub or hand-stand whatever is needed to demonstrate registration and teleop, and record it as M23 work.

## 6. Verify

- Construct the server with an Isaac env and the firmware SDK selected, and assert `set_observation` publishes a valid `HardwareObservations` matching the env.
- Send a `JointCommand` and assert the firmware control loop shaped the result, for example by observing the ramp or deadband in the resulting motion, and that the action came from simulated pin state.
- Launch with the flag set and unset, and assert Jetson and Isaac launches behave as before.
- Drive a teleop command end to end and assert the commanded joint moves and observations reflect it.
- Bring the simulated krab up and assert it registers, appears in the fleet UI, connects, and teleoperates; stop it and assert it shows offline.
- Enumerate anything missing on the untested path, fix the small items, and document the M23 hand-offs.
