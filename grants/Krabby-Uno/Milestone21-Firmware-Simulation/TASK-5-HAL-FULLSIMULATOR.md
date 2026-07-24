# Task 5 — HAL `FullSimulator` backend and fleet integration

Goal: Rewire the HAL to add a **third implementation** alongside the existing Jetson ("Krabby") and Isaac backends: a **`FullSimulator`** server that loads and executes the *entire firmware layer* in the call path — krabby-SDK → simulated firmware → simulated pins → IsaacSim — replacing today's Isaac backend that bypasses the firmware and krabby-SDK entirely. Then demonstrate, via an integration test, that the image can be spun up, the simulator launched through the new interface, and the simulated krab registered with the fleet manager and teleoperated. Because this is an untested path (even though it is meant to reuse the same interfaces), the task expects to surface and fix a few missing pieces.

Outputs
- A new `hal/server/fullsim/` backend package: `FullSimulatorHalServer` (reusing the Isaac observation path, overriding the command path to route through the firmware), a firmware-driven command SDK, a `main.py` entry point, and a console script.
- The command path wired so a `JointCommand` flows krabby-SDK → simulated firmware control loop → simulated pins → the Isaac env action tensor.
- Backend selection so the new server can be launched (console script + a `--backend fullsim` selection or launch branch).
- An integration test: launch the image/simulator via the new interface, register with the fleet manager, and teleoperate the simulated robot.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-5---hal-fullsimulator-backend-and-fleet-integration).)

---

**NOTE:** Line numbers are pointers to the right function and may drift. Heavy fleet wiring (publishing a sim image, auto-detecting "no krab attached") is **M23**; Task 5 proves the path works, it does not productize it.

---

## 1. How the HAL is structured today

`HalServerBase` (`hal/server/server.py` ~L26) is **not** an ABC — it is a concrete ZMQ transport (PUB observations, PULL commands) with `set_observation(hw_obs)` (~L140) and `get_joint_command(timeout_ms)` (~L184). There is **no abstract contract and no runtime factory**: each backend subclasses it, overrides `set_observation(self)` to build a `HardwareObservations` and call `super().set_observation(...)`, and provides its own command path. Selection is at the **packaging / entry-point layer**, not a config switch:

- **Jetson ("Krabby") backend** — `hal/server/jetson/hal_server.py` (`JetsonHalServer` ~L120). `apply_command(self, command)` (~L661) pushes to the MCU immediately and returns None. Wires the firmware serial SDK via `hal/server/jetson/krabby_mcusdk.py` (which imports `firmware.krabby_mcu.KrabbyMCUSDK`, ~L11, and converts radians→firmware in `apply_command` ~L105). Console script `krabby-hal-server-jetson` (`hal/server/jetson/pyproject.toml`), depends on `krabby-firmware`.
- **Isaac backend** — `hal/server/isaac/hal_server.py` (`IsaacSimHalServer` ~L190). `apply_command(self) -> torch.Tensor` (~L662) **polls the transport itself**, calls `IsaacSimMCUSDK.apply_command` (which just thresholds and returns the command dict — **no firmware, no pins**, `isaacsim_mcusdk.py` ~L36), converts dict → ordered array via `robot_definition.get_joint_names()` (~L754), and returns a tensor the main loop feeds to `env.step` (`isaac/main.py` ~L523–556). Console script `krabby-hal-server-isaac`.

The two `apply_command` shapes differ (Jetson takes a command + returns None; Isaac takes none + returns a tensor). **`FullSimulator` should follow the Isaac shape**, since it drives `env.step`.

## 2. The `FullSimulator` seam

Today's Isaac command path:
```
isaac/main.py:  action = hal_server.apply_command()
  └─ IsaacSimHalServer.apply_command()          # polls transport
       └─ get_joint_command()                    # ZMQ PULL → JointCommand
       └─ IsaacSimMCUSDK.apply_command(command)  # threshold → dict[name→rad]   ← NO firmware
       └─ dict → ordered np array → torch tensor → RETURN
isaac/main.py:  env.step(action)                 # action term → joint position targets
```

The **insertion point is the `IsaacSimMCUSDK.apply_command` step**. `FullSimulator` keeps everything else and replaces that step so the command routes through the firmware:
```
FullSimulatorHalServer.apply_command()
  └─ get_joint_command()                          # JointCommand (rad)
  └─ radians → firmware normalized/PWM            # as hal/server/jetson/krabby_mcusdk.py:32-38,105
  └─ hand to the (simulated) firmware SDK/loop    # T/B command → ActuatorManager.updateAll()
  └─ firmware writes simulated pins (Task 1)      # driveActuator → EN/PWM_R/PWM_L
  └─ read simulated pins → per-joint drive → torch action tensor  ← same shape as isaac today
env.step(action)
```

Two integration styles, both keeping the firmware in the loop:
1. **In-process firmware** — run the Task 1 `RobotCore`/firmware host build directly inside the server: feed it the command, tick `ActuatorManager::updateAll()`, feed the env's current joint positions back as pots, read the pin writes, and form the action tensor. Tightest and simplest for a single process.
2. **Through the serial SDK** — drive the simulated MCU via `firmware.krabby_mcu.KrabbyMCUSDK` over a virtual serial device (Task 2), so the *exact* krabby-SDK + serial protocol are exercised. Closer to production, more moving parts. Prefer (2) if the goal is to prove the full production call path (krabby-SDK included); (1) is a fine first step.

Reuse `IsaacSimHalServer` for the observation half — subclass it (or compose it) so `set_observation`, `_cache_references`, and the sensor interface are inherited unchanged. The required `HardwareObservations` fields are unchanged (`hal/client/data_structures/hardware.py` ~L49): `joint_positions`, `camera_height/width`, `timestamp_ns`, `base_ang_vel_b`, `base_lin_vel_b`, `base_quat_w`, `joint_velocities`, `contact_forces`, `previous_action`. (Fix the `self.contacvt_sensor` typo at `isaac/hal_server.py` ~L317 if you route contact/current data through it.)

## 3. Packaging and selection

Mirror the existing backends: a new sibling package `hal/server/fullsim/` with `hal_server.py`, a firmware-command module, `main.py`, `__init__.py`, and a `pyproject.toml` declaring `krabby-hal-server-fullsim = "hal.server.fullsim.main:main"` and depending on **both** `krabby-firmware` and the Isaac stack. Selection options:
- **Packaging-level** (matches today's pattern): the console script picks the server, exactly as `krabby-hal-server-jetson`/`-isaac` do.
- **Runtime flag** (cleaner for a sim launcher): a `--backend {isaac,fullsim}` flag inside the sim `main.py`, since that is the only place servers are constructed. `krabby run` selects the *launch script/entrypoint* today (`krabby/run.py` ~L20; `krabby/_docker.py` ~L125 hard-codes `krabby-hal-server-jetson` for gamepad mode), so a sim launch adds a branch there rather than a factory.

## 4. Fleet registration and teleop

The fleet path already exists for the real robot and is meant to be shared: the fleet agent (`krabby/agent.py`, IoT Core MQTT: shadow reporting, secure tunneling, teleop signaling), enrollment (`krabby/enroll.py`), and the operator-side fleet CLI (`fleet/cli/krabby_fleet_cli/`). The robot-presence signal that M23 will use to auto-select the sim image already exists — `krabby/telemetry.py` `collect_health()` reports `mcu_present` (~L101–103) and `reported_image` (~L173). For Task 5, the goal is narrower: get the **simulated** krab (running the `FullSimulator` backend) to register, show in the UI, connect, and teleoperate — reusing the existing teleop signaling that the HAL sensor interface (`get_sensor_interface`) feeds. Expect to find gaps here: this is the first time the fleet/teleop path runs against a firmware-in-the-loop simulator. Document what works, fix what's small, and hand off larger fleet-image/auto-detect work to M23.

> **Out of scope for M21 (→ M23):** publishing a sim image to ECR/S3 (the `images/isaacsim/` image builds locally via `make build-isaacsim-image` but is **never published** — there is no `publish-isaacsim` workflow, unlike `publish-locomotion.yml`); a `--sim` install flag / "no krab attached → pull SIM image" auto-detect in `krabby/install.py`/`__main__.py`; GPU-capability validation; and IoT-based resume. Task 5 may stub or manually stand up whatever it needs to demonstrate registration + teleop, and note it as M23 productization work.

## 5. Verify
- **Backend contract:** construct `FullSimulatorHalServer` with an Isaac env; assert `set_observation` publishes a valid `HardwareObservations` and observations match the env.
- **Firmware in the loop:** send a `JointCommand`; assert it passes through the firmware control loop (e.g. the firmware's ramp/deadband is observable in the resulting motion) and that the simulated pins, not a direct position target, produce the action.
- **Selection:** launch via the new console script / `--backend fullsim`; assert Jetson/Isaac launches still work.
- **End-to-end:** drive a teleop command through the full path; assert the simulated robot moves and observations reflect it.
- **Fleet:** bring the simulated krab up; assert it registers, appears in the fleet UI, connects, and teleoperates; stop the sim and assert it shows offline.
- **Gaps:** enumerate anything missing on the untested path; fix small items, document M23 hand-offs.
