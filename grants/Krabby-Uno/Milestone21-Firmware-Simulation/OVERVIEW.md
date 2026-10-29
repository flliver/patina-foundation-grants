# Patina Foundation Grant — Krabby-Uno Milestone 21: Firmware Simulation

## Grant Overview

Make the entire Krabby Arduino firmware runnable in simulation, with no physical robot required, so that an engineer or an AI agent can exercise, test, and iterate on the firmware the same way they run any other part of the stack.

Today the firmware runs only on the Arduino Mega. Seeing it do anything means flashing three boards, wiring six actuators each, and watching a physical robot move, which is slow for a human and effectively out of reach for an AI agent.

This milestone closes that gap with a second implementation of the Arduino hardware API. `digitalWrite`, `analogWrite`, `digitalRead`, `analogRead`, `millis`, `EEPROM`, `Serial`, and the hall-encoder interrupt path each gain a simulated backend that feeds pin-level electrical values into IsaacSim and reads simulated physics back out.

The core abstraction is a `SimulatedRobotConfig`, which composes a robot by binding every pin from `firmware/arduino/board_pins.h` to a defined actuator device. It attaches to a `RobotCore` game loop that watches pin transitions, finds the device on each pin, and executes the mapped Isaac command, so the `EN` and `PWM_R`/`PWM_L` writes made by `ActuatorManager::driveActuator()` become a linear actuator extending or retracting at the commanded speed.

From there the milestone builds up in stages. The read side adds `digitalRead`, `analogRead`, EEPROM, the hall encoder, `millis`, and a physics-derived current-sense signal that rises as a foot loads the ground. The serial command and telemetry protocol lets the existing krabby-SDK drive the simulated firmware and read telemetry back, and the inter-board serial links put three simulated Arduinos in control of all 18 legs in a single IsaacSim scene. Simulators for the Milestone 16 I2C sensors cover the IMU, OLED health screen, and battery monitors. Finally, a third HAL implementation called `FullSimulator` puts krabby-SDK and the firmware in the call path for a normal image launch, so a simulated krab can register with the fleet manager and be teleoperated.

The output is a firmware you can run, drive, and observe entirely in software.

## Why is this Important?

- The firmware is currently untestable without hardware. Every change requires flashing physical boards and watching a real robot, which keeps the firmware outside CI and out of reach for an AI agent. This milestone turns it into a normal, runnable, testable software component.
- It unblocks AI-driven firmware development. An agent can define a simulated robot, send commands, and see each leg move through unit tests, which the flash-and-watch loop makes impossible today.
- It closes the last un-simulated layer of the stack. The HAL already has an Isaac backend, but that backend bypasses the firmware and krabby-SDK entirely. After M21 the real firmware control loop and serial protocol run in the simulated call path, so what you test in sim is what ships on the robot.
- It gives current-sense and the health UI something real to show. A physics-backed current signal that responds to ground contact makes the GUI and telemetry meaningful in sim, and provides a tunable foundation for contact-based behaviors in later milestones.
- It is the foundation for fleet-level simulation in M23. Once the firmware runs in sim behind the HAL, a simulated krab can be pulled, launched, and teleoperated through the fleet manager on a desktop GPU box, with M21 having proven the base emulator works reliably.

## Tasks

Total: ~27 working days, roughly 4–5 weeks part-time at 20–30 hours per week. Implementation detail and exact code locations live in the per-task files linked below; the acceptance criteria are here.

### Task 1 - RobotCore loop + digitalWrite: drive the actuators → [TASK-1-PIN-SIMULATION-INTERFACE.md](TASK-1-PIN-SIMULATION-INTERFACE.md)

#### Narrative

Task 1 builds the `RobotCore` game loop together with the Arduino output calls: `pinMode`, `digitalWrite`, `analogWrite`, and `millis` for the loop clock. When the firmware's `driveActuator()` sets `EN` high and a `PWM_R`/`PWM_L` duty, the simulator records those writes, resolves the actuator device bound to those pins, and drives the linked IsaacSim joint at a signed proportional speed of `(EN==HIGH) × (PWM_R − PWM_L)` across the range −255…255. Dynamics are assumed linear, so −255 is a full retract and +122 is roughly a 48% extend.

The deliverable is a `RobotCore` you can jog: drive each of the 6 legs open-loop from test code and watch the joint move in sim. Verification inspects the sim joint directly, since the read path arrives in Task 2, which is also where position feedback, the closed-loop P-controller, and calibration come from.

This task also stands up the minimal host build that does not exist today, covering enough to compile the drive path (`driveActuator` and `manualDrive`) against a pin-output and `millis` shim. It covers `SimulatedRobotConfig`, `SimulatedActuator`, and `RobotCore` for one board of 6 motors on `KRABBY_PIN_REV` 3.

#### Acceptance Criteria

- 1a — The firmware's actuator drive path (`driveActuator`/`manualDrive`) compiles and runs on the host against a minimal output shim (`pinMode`/`digitalWrite`/`analogWrite`/`millis`), with `analogRead`, `digitalRead`, hall, EEPROM, and serial stubbed for later tasks. The existing `arduino-cli` AVR build (`firmware/Makefile`) still works unchanged.
- 1b — A `SimulatedRobotConfig` binds each of the 6 slots' drive pins (PWM_R/PWM_L/EN, per `board_pins.h` for `KRABBY_PIN_REV` 3) to a `SimulatedActuator` bound to a named IsaacSim joint. POT and IS pins are declared in the config and read in Task 2.
- 1c — A `RobotCore` accepts a `SimulatedRobotConfig`, runs a game loop, and on each tick reads the recorded pin state, resolves the device owning each pin, drives the linked joint, and steps physics.
- 1d — `driveActuator()`'s pin writes map to a signed motor command `cmd = (EN==HIGH) ? (pwm_R − pwm_L) : 0` taken from the recorded `digitalWrite`/`analogWrite` state, over the range −255…255: −255 retracts at full speed, +255 extends at full speed, +122 extends at roughly 48%, and 0 or EN-low holds.
- 1e — The 6 motors are independently addressable: driving one leg moves only that leg's joint, and the pin→device→joint resolution is correct for all six slots.
- 1f — Integration tests jog each of the 6 legs through extend, retract, hold, and partial speed, asserting the linked Isaac joint moves in the right direction at the right proportional speed, verified by inspecting the sim joint directly.

#### Time estimate (~6 days)

| Days | Sub-task title |
|------|----------------|
| 1 | Minimal output shim (`pinMode`/`digitalWrite`/`analogWrite`/`millis`) and compile the actuator drive path |
| 1 | `SimulatedActuator` (drive pins → Isaac joint) and signed PWM→speed model |
| 0.5 | `SimulatedRobotConfig` binds the 6 REV-3 drive-pin sets to actuators and joints |
| 2 | `RobotCore` game loop and IsaacSim integration: advance, read pin writes, drive joints, step physics |
| 0.5 | Integration tests (6 legs jog: extend, retract, hold, partial) |

### Task 2 - digitalRead / analogRead: sensors and current-sense → [TASK-2-SENSORS-AND-CURRENT-SENSE.md](TASK-2-SENSORS-AND-CURRENT-SENSE.md)

#### Narrative

Task 2 builds the read side, `analogRead` and `digitalRead`, on top of Task 1's drive loop. The first read is `analogRead(pinPot)`, which returns each joint's position on the ADC scale and closes the firmware's control loop, bringing `getPos()`, the `update()` P-controller driven by `setTarget`..

Each motor's pot also carries per-motor calibration variance, deterministic but drawn at `SimulatedRobotConfig` instantiation from a min-stop range of roughly 80–220 and a max-stop range of roughly 800–1000, mimicking manufacturer POT variance so auto-calibration discovers different limits per motor. Alongside it, `digitalRead(pinEn)` gives the EN read-back, the hall edge counter is a native shim that emulates `saf` from joint motion scaled so a full min-to-max travel matches the real robot's triggers-per-travel, `EEPROM` is an in-memory byte array holding `CalData` and role, and `millis()` is a monotonic sim clock.

The headline deliverable is a current-sense signal derived from IsaacSim physics. `analogRead(pinIS)` is mapped from the joint reaction wrench (`robot.data.body_incoming_joint_wrench_b`) at the appropriate leg link, resolved via `robot.find_bodies()`, with the per-joint actuator `applied_effort` as a cheaper cross-check. The scale is tuned low and by eye: 0 for no movement, around 2 for moving without ground contact, 4 for basic ground contact, rising to roughly 30 of 1023 when a leg lifts significant force and offloads the other legs. Validation is unit tests plus a manual tuning harness, a windowed single-env viewer where you perturb a leg and watch the wrench, both reading the sim directly. 

The overall goal is that we're able to put move the limb in simulation, see appropriate EN, POT, and current sense values, as well as a store calibration data in a simulated EEPROM.

#### Acceptance Criteria

- 2a — `digitalRead(pin)` returns the last state written to that pin, so EN lines read back correctly for the telemetry `enL`/`enR` fields Task 3 emits.
- 2b — `analogRead(pinPot)` returns each joint's position on the ADC scale, closing the control loop so `getPos()`, the `update()` P-controller via `setTarget`, and auto-calibration all work. Each motor's readout carries deterministic per-motor calibration variance assigned at `SimulatedRobotConfig` instantiation, with min-stop drawn from roughly [80,220] and max-stop from roughly [800,1000], fixed per motor thereafter and seeded from the config for reproducibility, so auto-calibration discovers a different `minStop`/`maxStop` per motor. `RobotCore`'s loop gains its read path here, reading joint positions back to feed the pots.
- 2c — `EEPROM.put`/`get`/`read`/`update` are backed by a byte array, and the calibration `CalData` (byte 0, magic `0xDEADBEEF`) and board role (byte 32 magic `0xAB`, byte 33 role) round-trip properly. In-memory storage is sufficient at this stage; persistence between reboots is out of scope.
- 2d — The hall edge counter (`hallHwInit`/`hallHwGetEdgeCount`) is a native shim that emulates edge counts from simulated joint velocity, with no AVR register or ISR emulation. `saf` advances monotonically as a joint moves and holds when it stops, and the total count across a full min-stop to max-stop travel matches the real robot. Obtain the average encoder triggers between stops from James, or read them off a real robot via the fleet manager.
- 2e — Task 2 completes the read and timing Arduino surface the actuator/sensor layer needs on top of Task 1's output shim. Sweep `actuator_manager.h` and `hall_hw.cpp` for any remaining Arduino symbol (`analogRead`, `digitalRead`, `EEPROM`, the hall counter; `millis` already comes from Task 1's shim) and give each a sim implementation, so the simulated actuator and sensor layer imports no real Arduino. `Serial` is Task 3 and `Wire`/I2C is Task 4, the only Arduino APIs remaining after this task.
- 2f — `analogRead(pinIS)` returns a physics-derived current value, hooking `robot.find_bodies()` and the joint reaction wrench (`body_incoming_joint_wrench_b`) at the appropriate leg link (knee or hip), mapped onto the 0–1023 ADC scale.
- 2g — The current-sense scale is tuned to 0 for no movement, around 2 for moving without ground contact, 4 for basic ground contact, and roughly 30 of 1023 when a leg lifts significant force and begins to pick up the body. Behavior is changeable by modifying the output mapping function. These should be settable via a config function, so that we can have per-motor config definitions.
- 2h — Current-sense, POT, and hall values are validated via integration tests: move a leg via direct PWM pin commands and show the POT, hall, and current values change as expected by round-tripping through IsaacSim.

#### Time estimate (~6 days)

| Days | Sub-task title |
|------|----------------|
| 1 | `analogRead(pinPot)` position read (closes the control loop) and `digitalRead` EN read-back |
| 0.5 | Per-motor pot calibration variance and EEPROM (`CalData`/role, in-memory) |
| 1 | Hall edge synthesis scaled to James's real-robot triggers-per-travel; confirm `millis`/timing tracks sim time |
| 2 | Physics current-sense: wrench (`body_incoming_joint_wrench_b`) → `pinIS`; manual tuning harness; tune 0 / ~2 / 4 / ~30 |
| 0.5 | Integration tests (pot read and variance, digitalRead, EEPROM, hall and travel count, millis, current-sense breakpoints) |

### Task 3 - Serial emulation and three-board firmware → [TASK-3-SERIAL-EMULATION.md](TASK-3-SERIAL-EMULATION.md)

#### Narrative

Task 3 extends the simulated firmware to speak the serial protocol end to end, so the existing krabby-SDK (`firmware/krabby_mcu.py`) can connect to the simulated MCU, send commands, and receive telemetry back, closing the full command-to-motion-to-telemetry loop against IsaacSim. Because the SDK opens a port by path and honors `KRABBY_MCU_PORT`, the simulated MCU attaches through a virtual serial device — a PTY, or a virtual COM pair on Windows — leaving the SDK unchanged.

The simulated firmware parses the full host-to-MCU command set (`T` targets, `B` batch jog, `J` single jog, `C` calibrate, `H` hold, `V` version) and emits role-prefixed telemetry lines in the exact wire format on the roughly 50 ms tick, exactly like it does on the real HW today. Since Tasks 1 and 2 already produce real drive and sensor values, wiring serial makes them flow to the SDK, so `python -m firmware.gui` works as-is: move the robot and watch the "Cur" and "Pot" columns respond.

The task then emulates the inter-board serial links so three simulated Arduinos cooperate in one IsaacSim scene, driving all 18 legs. That means wiring each emulator instance's `Serial1` and `Serial2` to its peers so the real `determineRole()` election runs with one FRONT leader and LEFT/RIGHT followers, having the leader forward each follower's telemetry line verbatim and relay commands, and binding the three boards to three disjoint sets of six joints of the shared crab Articulation. 

A single-leader shortcut, where one emulator owns all 18 joints and emits all three role-prefixed lines, is a valid stepping stone before the true three-instance election exercises the multi-board code paths, if necessary, but the output goal of three boards communicating over simulated serial lines lets us test the whole serial communication across boards e2e.

#### Acceptance Criteria

- 3a — The simulated firmware exposes a serial endpoint that `serial.Serial(port)` can open by path, and `KrabbyMCUSDK(port=...)` or `KRABBY_MCU_PORT` connects with no SDK code changes.
- 3b — The simulated firmware parses the full host-to-MCU command set: `T` (position targets, val 0–1), `B` (batch jog, pwm −255…255), `J` (single jog), `C` (calibrate), `H` (hold/de-energize), and `V` (version), matching `arduino.ino`'s `loop()` dispatch.
- 3c — The simulated firmware emits telemetry lines in the exact wire format every ~50 ms: `<ROLE>; <name> <pos> <pot> <current> <enL> <enR> <pwmL> <pwmR> <saf>; …`, six joints per line joined by `;`, parseable by `firmware/interfaces/joint_telemetry.py` unchanged.
- 3d — A full round trip is demonstrated: the SDK sends a jog or target command, the linked Isaac joint moves, telemetry reflects the new `pos`/`pwm`/`en`, and `KrabbyMCUSDK.joints[name]` updates, all in a unit or integration test without hardware.
- 3e — `V` returns a well-formed `VER …` reply that `parse_ver_reply` accepts, `H` de-energizes all actuators, and `C` starts the calibration state machine and completes against simulated joint limits.
- 3f — Three simulated Arduinos elect roles over emulated inter-board serial, one FRONT leader with LEFT and RIGHT followers, matching `determineRole()` semantics, and the leader ends up owning the USB/host link.
- 3g — The leader forwards each follower's full telemetry line verbatim to the host, so the SDK sees three role-prefixed lines (`FRONT;…`, `LEFT ;…`, `RIGHT;…`) per cycle, and relays `T`/`B`/`J`/`C`/`H`/`V` to followers.
- 3h — The three MCU instances are each bound to a disjoint set of 6 joints of the same 18-joint crab in one IsaacSim scene, and commanding any of the 18 legs through the SDK moves exactly that leg.
- 3i — End-to-end current-sense in the GUI: `python -m firmware.gui` against the simulated MCU shows the "Cur" column responding as legs load the ground, rising on contact, reaching roughly 30 when lifting the body, and resting near 0, with serial telemetry staying nominal and all fields parsing via `joint_telemetry.py`.

#### Time estimate (~4 days)

| Days | Sub-task title |
|------|----------------|
| 1 | Simulated `Serial`/`HardwareSerial` backed by a virtual serial device; SDK connects via `KRABBY_MCU_PORT`/PTY |
| 0.5 | Host-to-MCU command parse (`T`/`B`/`J`/`C`/`H`/`V`); telemetry emitted in exact wire format on the 50 ms tick |
| 0.5 | Full round-trip test: command → Isaac motion → telemetry → `mcu.joints[...]`; `V`/`H`/`C` behaviors |
| 0.5 | Three-instance role election over emulated inter-board serial; leader telemetry forwarding and command relay |
| 1 | Bind three boards to 18 joints in one scene; per-leg test across all 18 using standard krabby-cli commands |
| 0.5 | GUI current-sense end-to-end (`python -m firmware.gui`) and telemetry-nominal check |

### Task 4 - I2C sensor simulators (IMU, OLED, battery) → [TASK-4-I2C-SENSOR-SIMULATORS.md](TASK-4-I2C-SENSOR-SIMULATORS.md)

#### Narrative

Task 4 adds simulators for the Milestone 16 I2C sensors: an IMU (BMI270, `0x68`), the OLED health screen (SSD1306, `0x3D`), and the battery-sense chips (two INA228, `0x40` and `0x41`). It also adds the simulated `Wire`/I2C layer on top of the Task 1 host build, which so far provides only the pin and `millis` shim.

All three devices use I2C with standard call and response patterns, so the simulator mimics the data already present in M16's unit and integration tests rather than modeling physics. The simulated `Wire` layer returns believable register values and the real driver code, compiled in the host build, does the parsing. The IMU derives accel and gyro cheaply from the sim base state, the INA228s return nominal pack and per-battery readings, and John's OLED simulator is hooked through the simulated `Wire` interface so it renders the krab status screen from live simulated-firmware state. This is a short task requiring little simulation integration or dynamics.

Two dependencies must be resolved first, and the task flags them explicitly. The M16 sensors are not yet implemented in the real firmware — `firmware/arduino/` today has no `sensors_config.h`, no `#include <Wire.h>`, and no BMI270, SSD1306, or INA228 code, since M16 is spec-complete but code-incomplete. Separately, John's OLED simulator is not committed to `krabby-research` and must be located and vendored in, or pointed to. If the M16 firmware is still incomplete when this task starts, the sim work proceeds against the same agreed I2C contract — addresses plus `IMU` and `BATT` telemetry schemas — that the M16 grant already specifies.

#### Acceptance Criteria

- 4a — A simulated I2C/`Wire` layer supports the four M16 device addresses (BMI270 `0x68`, OLED `0x3D`, Pack INA228 `0x40`, Midpoint INA228 `0x41`) with standard read and write call/response, and addressing an absent device fails gracefully, matching the firmware's init-failure handling.
- 4b — The simulated IMU returns three-axis accel, three-axis gyro, and temperature; the simulated firmware appends the M16 `IMU` telemetry segment and `joint_telemetry.py`'s IMU parser from M16 accepts it.
- 4c — The OLED simulator is driven through the new pin and `Wire` interface and renders the health screen — controller-thirds liveness, per-actuator glyphs, battery bars — from the simulated firmware's live state rather than as a standalone mock.
- 4d — The simulated INA228s return pack and per-battery voltage and current, the simulated firmware emits the M16 `BATT` frame, and the M16 battery parser accepts it.
- 4e — Sensor values mirror the data already present in M16's unit and integration tests. Where a value can be derived cheaply from the sim, such as IMU from base orientation and velocity, it may be, though fidelity is not required.
- 4f — Tests exercise each simulated sensor end-to-end through the firmware, from I2C call to firmware read to telemetry or OLED output, without hardware.
- 4g — Absent or failed sensors leave the simulated firmware running: init failure sets the invalid flag and control continues, matching M16's non-fatal-init requirement.

#### Time estimate (~2 days)

| Days | Sub-task title |
|------|----------------|
| 1.0 | Simulated `Wire`/I2C layer for `0x68`/`0x3D`/`0x40`/`0x41`; graceful absent-device handling |
| 0.5 | Simulated BMI270 → `IMU` segment; simulated INA228 ×2 → `BATT` frame; hook OLED simulator to render live state |
| 0.5 | End-to-end tests per sensor through the firmware; non-fatal-on-missing check |

### Task 5 - HAL `FullSimulator` backend and fleet integration → [TASK-5-HAL-FULLSIMULATOR.md](TASK-5-HAL-FULLSIMULATOR.md)

#### Narrative

Task 5 rewires the HAL to add a third implementation alongside the existing Jetson ("Krabby") and Isaac backends: a `FullSimulator` server that loads and executes the entire firmware layer in the call path, running krabby-SDK to simulated firmware to simulated pins to IsaacSim. This replaces the behavior of today's Isaac backend, which converts a `JointCommand` straight into joint position targets and never touches the firmware.

`FullSimulator` reuses the Isaac observation half by subclassing `IsaacSimHalServer`, leaving `set_observation`, `_cache_references`, and the sensor interface unchanged, and overrides only the command path. A `JointCommand` is converted to firmware commands (radians to normalized, as the Jetson adapter does), the simulated firmware's `ActuatorManager` control loop runs, its simulated pins produce per-joint drive, and that becomes the `env.step` action tensor with the same shape today's Isaac `apply_command` returns. The backend is selected the way the others are, through a `krabby-hal-server-fullsim` console script or a `--backend fullsim` flag in the sim launcher.

The task then demonstrates, via an integration test, that the image can be spun up, the simulator launched through the new interface, and the simulated krab registered with the fleet manager and teleoperated, reusing the existing teleop signaling. Because the fleet and teleop stack has never run against a firmware-in-the-loop simulator, the task expects to surface and fix a few missing pieces, or document them clearly as M23 work. Heavy fleet productization — publishing a sim image to ECR/S3, auto-detecting "no krab attached" to pull the SIM image, GPU validation, and IoT resume — is deferred to M23. Task 5 proves the path works and stops there.

#### Acceptance Criteria

- 5a — A `FullSimulatorHalServer(HalServerBase)` exists in `hal/server/fullsim/` and implements the practical HAL contract: `set_observation(self)` building a valid `HardwareObservations` and calling `super().set_observation(...)`, a command path that consumes a `JointCommand`, `get_sensor_interface()`, and `close()`.
- 5b — The command path puts both krabby-SDK and the firmware in the loop: a `JointCommand` is converted to firmware commands (radians to normalized, as the Jetson adapter does), the simulated firmware's `ActuatorManager` control loop runs, its simulated pins produce per-joint drive, and that becomes the `env.step` action tensor, replacing the direct `JointCommand`-to-position-target passthrough of `IsaacSimMCUSDK`.
- 5c — Observations still come from the Isaac env (joint state, base state, cameras, contacts), reusing `IsaacSimHalServer.set_observation` and `_cache_references` so the observation half is unchanged.
- 5d — The backend can be selected and launched: a `krabby-hal-server-fullsim` console script, or a `--backend fullsim` flag in the sim `main.py`, constructs and runs the new server, and existing Jetson and Isaac launches are unaffected.
- 5e — An integration test spins up the simulator through the new interface and drives the full path end to end (command in, firmware loop, sim pins, `env.step`, observation out), asserting a teleop command moves the simulated robot and observations reflect it.
- 5f — The simulated krab registers with the fleet manager, appears in the UI, can be connected to, and is teleoperable. When the sim is stopped it shows offline, with manual restart via the krabby CLI.
- 5g — Any missing pieces on this untested path are identified and fixed, or clearly documented as deferred to M23, since the fleet and teleop path has not been exercised against the firmware-in-the-loop simulator before.

#### Time estimate (~5 days)

| Days | Sub-task title |
|------|----------------|
| 1 | `hal/server/fullsim/` package scaffold: `FullSimulatorHalServer` reusing Isaac observation half; console script and `pyproject.toml` |
| 1.5 | Command path: JointCommand → firmware control loop → simulated pins → `env.step` action tensor |
| 0.5 | Backend selection (`--backend fullsim` or console script); confirm Jetson and Isaac launches unaffected |
| 1 | Integration test: full path end-to-end, teleop command moves the sim robot |
| 1 | Fleet registration and teleop demo; enumerate and fix gaps or document M23 hand-offs |

## Information

- Repositories: `krabby-research` (firmware, SDK, HAL, Orin runtime, parkour/Isaac task), `patina-foundation-grants` (this grant), `krabby-contracts` (milestone ICA — the `M21/` folder is currently empty and the ICA is TBD), and `IsaacLab` (upstream Isaac Lab, vendored alongside).
- Firmware C++, the thing being simulated: `firmware/arduino/board_pins.h` (pin map, `KRABBY_PIN_REV`), `firmware/arduino/actuator_manager.h` (`LinearActuator`, `ActuatorManager`, `driveActuator`, `updateSensors`, EEPROM `CalData`), `firmware/arduino/arduino.ino` (`setup`, `loop`, role election, telemetry, serial command parse), `firmware/arduino/hall_hw.cpp` (PCINT edge counter), `firmware/arduino/command.h`, and `version.h`. There are only seven real firmware files; the `firmware/arduino/Downloads/` tree is unrelated junk.
- Python SDK and sim host side: `firmware/krabby_mcu.py` (`KrabbyMCUSDK`, serial reader, command emit), `firmware/interfaces/joint_telemetry.py` (wire format), `firmware/mcu_port.py` (`default_port`, `KRABBY_MCU_PORT`), `firmware/gui/app.py` (telemetry/jog GUI), `firmware/cli.py`, and `firmware/Makefile` (arduino-cli build).
- HAL: `hal/server/server.py` (`HalServerBase`), `hal/server/config.py`, `hal/server/robot_definition*.py`, `hal/server/isaac/{hal_server,isaacsim_mcusdk,main}.py` (Isaac backend), `hal/server/jetson/{hal_server,krabby_mcusdk,main}.py` (real-robot backend), and `hal/client/data_structures/hardware.py` (`HardwareObservations`, `JointCommand`).
- IsaacSim and robot: `assets/crab_simple.usda` (USD, 18 joints), `parkour/parkour_tasks/parkour_tasks/crab_hexapod_task/config/crab_hex/crab_hex_scene_cfg.py` (`ArticulationCfg`, actuators), `.../sensors/parkour_hex_contact_sensor.py`, `parkour/parkour_isaaclab/actuators/parkour_actuator_pd.py`, and `IsaacLab/source/isaaclab/isaaclab/assets/articulation/articulation.py` (`find_joints`, `set_joint_*_target`) with `articulation_data.py` (`body_incoming_joint_wrench_b`).
- Image and fleet, for Task 5: `images/isaacsim/` (built locally, not published), `images/locomotion/Dockerfile.release`, `krabby/{_state,install,run,_docker,telemetry,__main__}.py`, `fleet/` (`fleet/cli/`, `fleet/infra/`), and `.github/workflows/publish-locomotion.yml`.
- Dependencies: Milestone 16 (I2C sensors) for Task 4, noting the M16 firmware is spec-complete but not code-complete. A CUDA-capable desktop, RTX 5080-class, is needed for running IsaacSim, per the M23 target hardware.

## Looking ahead to M23

M21 delivers a reliable base emulator: the firmware runs in sim, krabby-SDK drives it, and a `FullSimulator` HAL backend puts it in the normal image call path.

M23 builds the fleet-manager experience on top. Install the krabby fleet client on a stock Ubuntu box, have it detect that no real krab is attached (the `mcu_present` signal already exists in `krabby/telemetry.py`), prompt to pull an IsaacSim image in place of the real image, which requires standing up a `publish-isaacsim` workflow and ECR repo since the sim image builds locally today but is never published, validate the box has the GPU to run it, launch it, and have it register with and be teleoperated through the fleet manager UI. When the sim is off, the robot shows offline until manually restarted via the krabby CLI. That work sits outside M21 scope and is captured here so the emulator work lands with the fleet direction in view.

## FAQ

- Why build a second `digitalWrite`/`analogWrite` instead of rewriting the firmware for sim? The point is to run the real firmware — the same `actuator_manager.h` control loop, ramp, deadbands, and serial protocol that ship on the robot. Swapping the hardware API underneath the control logic means what you test in sim is the shipping firmware rather than a reimplementation that can drift.
- How does a PWM value become motion? `driveActuator()` writes `EN` HIGH or LOW and one of `PWM_R`/`PWM_L` at 0–255. The simulator reads those pins as a signed command `(EN==HIGH) ? (pwm_R − pwm_L) : 0` in the range −255…255 and drives the linked Isaac joint at the proportional speed, assuming linear dynamics for now. Position feedback comes back as `analogRead(pinPot)` mapped to 0–1023.
- Does the firmware compile off the Arduino today? It builds only for AVR via `arduino-cli`, and the repo has no C++ unit-test harness. Task 1 stands up a minimal host shim covering pin I/O and `millis`, sufficient to compile and run the actuator control layer (`actuator_manager.h`) without emulating AVR machinery. The fuller host build — the AVR-specific `hall_hw.cpp` and EEPROM in Task 2, serial in Task 3 — is filled in as those subsystems come online, so each task emulates only what it needs.
- Which board revision does the emulator support? `KRABBY_PIN_REV` 3, the current Uno v0.2 board. Supporting a future pin revision should be cheap: update the pin numbers in `firmware/arduino/board_pins.h`, as you would for the real firmware anyway, then update the emulator's `SimulatedRobotConfig` to map the new pin to its `SimulatedActuator`. The pin-to-device-to-joint resolution is data-driven from the config, so a pin move is a one-line binding change rather than a code change.
- Do we need real serial ports for the SDK to talk to sim? `KrabbyMCUSDK` opens a port by path and honors `KRABBY_MCU_PORT`, so the simulated MCU attaches through a virtual serial device — a PTY, or a virtual COM pair on Windows — with zero SDK changes.
- Is the current-sense meant to be physically accurate? At this stage the goal is sensible, tunable outputs — 0, ~2, 4, ~30 of 1023 across no-motion through loaded-contact — that respond correctly to what the robot is doing. Fidelity is a later milestone.
- Where does the current-sense number come from? IsaacSim physics: `robot.data.body_incoming_joint_wrench_b`, the joint reaction wrench through the leg, at the hip or knee, resolved via `robot.find_bodies()` and mapped onto the `pinIS` 0–1023 ADC scale. The per-joint actuator `applied_effort` serves as a cheaper secondary proxy.
- Why is Task 4 short when M16 was a whole milestone? Task 4 simulates the I2C call and response for three sensors behind the pin interface built in Tasks 1–3, with no physics or dynamics, mirroring data already in M16's tests. It does depend on the M16 firmware and John's OLED simulator existing, which the task calls out.
- What is the difference between the existing Isaac HAL backend and the new `FullSimulator`? Today's Isaac backend converts a `JointCommand` straight into Isaac joint position targets, leaving the firmware and krabby-SDK out of the path. `FullSimulator` keeps the Isaac observation half but routes commands through krabby-SDK, the simulated firmware, and simulated pins into the env action, exercising the real firmware control loop end to end.
- What is deferred to M23? The full fleet-manager experience: publishing a sim image to ECR/S3, auto-detecting "no krab attached" to pull the SIM image, and making the whole system simple to use. M21 focuses on getting the base emulator working reliably, and Task 5 proves the path end-to-end while heavy fleet wiring waits for M23. See "Looking ahead to M23" above.
- Where are the full task details? In this folder: `TASK-1-PIN-SIMULATION-INTERFACE.md`, `TASK-2-SENSORS-AND-CURRENT-SENSE.md`, `TASK-3-SERIAL-EMULATION.md`, `TASK-4-I2C-SENSOR-SIMULATORS.md`, and `TASK-5-HAL-FULLSIMULATOR.md`.
