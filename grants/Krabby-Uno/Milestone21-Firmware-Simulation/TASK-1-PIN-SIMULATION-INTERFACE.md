# Task 1 — RobotCore loop + digitalWrite: drive the actuators

Goal: build the `RobotCore` game loop and the Arduino output calls (`pinMode`, `digitalWrite`, `analogWrite`, and `millis` for the loop clock) so the firmware's `driveActuator()` pin writes move IsaacSim joints. The deliverable is a loop you can jog: drive each of the 6 legs open-loop from test code and watch the joint move in sim.

Outputs
- A minimal host Arduino shim for the output calls, with the read calls stubbed so the drive path compiles.
- A `SimulatedActuator` linking a slot's drive pins (PWM_R, PWM_L, EN) to one IsaacSim joint, with a linear PWM-to-speed model.
- A `SimulatedRobotConfig` binding the 6 REV-3 slots' drive pins to actuator devices, and a `RobotCore` game loop that reads recorded pin state, drives joints, and steps physics.
- Tests that define a 6-motor config, jog each leg through extend, retract, and hold, and assert the linked Isaac joint moves in the correct direction at the correct proportional speed.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-1---robotcore-loop--digitalwrite-drive-the-actuators).)

---

Note: line numbers are pointers to the right function and may drift. Position feedback, the closed-loop P-controller, and calibration all depend on the read path and arrive in Task 2.

---

## 1. The output seam — the pin writes `driveActuator` makes

Everything the drive path does to hardware goes through a handful of Arduino output calls in `firmware/arduino/actuator_manager.h`. The one that matters is `LinearActuator::driveActuator(int pwm, int pwmDeadband)` (~L224-244):

```cpp
if (abs(pwm) < deadband) { digitalWrite(pinEn, LOW);  analogWrite(pinPwmR, 0);   analogWrite(pinPwmL, 0); }        // stop/coast
else if (pwm < 0)        { digitalWrite(pinEn, HIGH); analogWrite(pinPwmR, 0);   analogWrite(pinPwmL, abs(pwm)); } // retract (Left)
else                     { digitalWrite(pinEn, HIGH); analogWrite(pinPwmR, pwm); analogWrite(pinPwmL, 0); }        // extend (Right)
```

Only one of R and L is ever nonzero, and `EN` gates whether the H-bridge is live. The simulator records these writes and interprets `(EN==HIGH) × (PWM_R − PWM_L)` as a signed motor drive for the linked joint. The `EN==HIGH` test reads the state the simulator recorded from `digitalWrite`, so it stays independent of the firmware's own `digitalRead`, which Task 2 implements.

The calls Task 1 services:

| Arduino API | Called at (`actuator_manager.h`) | Sim responsibility |
|---|---|---|
| `pinMode(pin, mode)` | ~L57-61 | record direction; a no-op is fine |
| `analogWrite(pwmPin, 0-255)` | ~L66-67, 229-242 | capture per-pin PWM duty |
| `digitalWrite(enPin, HIGH/LOW)` | ~L68, 228, 234, 240 | record EN state; gate the slot's motor |
| `millis() -> ulong` | ~L148-150 (ramp), loop | monotonic sim clock in ms |

Because the reads are stubbed, drive the actuators through the open-loop jog path: `ActuatorManager::handleJog(...)` to `LinearActuator::manualDrive()` (~L115-128) to `driveActuator()`. That path sets `currentPwm` directly and leaves `updateSensors()` and `analogRead` out of it. The closed-loop `update()` path, where the P-controller reads `analogRead(pinPot)`, belongs to Task 2.

## 2. The pin map (`board_pins.h`) — REV 3

Pins are selected at compile time by `KRABBY_PIN_REV`, and the emulator supports Rev 3, the current Uno v0.2 board. PWM pins are identical across revisions, and the EN pins below are the Rev 3 values. The POT and IS analog pins appear here because the config binds them, and Task 2 reads them.

| Slot | PWM_R | PWM_L | EN (Rev 3) | POT (Task 2) | IS (Task 2) |
|---|---|---|---|---|---|
| S0 | D2 | D3 | D22 | A0 | A6 |
| S1 | D4 | D5 | D24 | A1 | A7 |
| S2 | D6 | D7 | D26 | A2 | A8 |
| S3 | D8 | D9 | D23 | A3 | A9 |
| S4 | D10 | D11 | D25 | A4 | A10 |
| S5 | D12 | D13 | D27 | A5 | A11 |

The `SimulatedRobotConfig` binds these per slot, sourced from `board_pins.h` rather than re-hardcoded, so a future pin revision is a `board_pins.h` change plus the matching binding update in the config.

## 3. The minimal host build

The firmware builds for AVR via `arduino-cli` (`firmware/Makefile`). Host compilation and a C++ test harness would both be new: the repo has neither, and no ArduinoFake, EpoxyDuino, arduino_ci, PlatformIO, or simavr to build on. Task 1 stands up enough to run the actuator drive path.

Two pieces are needed. First, a minimal `Arduino.h` shim implementing the output calls `driveActuator` and `manualDrive` use — `pinMode`, `digitalWrite`, `analogWrite`, `millis`, plus the `constrain` and `abs` macros — routed to the simulator's pin and clock model. Second, stubs for the reads and system calls the actuator translation unit references but this task never exercises: `analogRead`, `digitalRead`, `EEPROM.h`, and `hallHwGetEdgeCount`. Return-constant and no-op stubs resolve the symbols so the drive path links, and Task 2 replaces them with real implementations.

`actuator_manager.h`, `command.h`, and the drive math are portable C++ and compile unchanged against the shim. Compile the actuator layer into a host library and drive it from Python, since the sim and IsaacSim side is Python: a C-ABI or pybind11 boundary lets Python own `SimulatedRobotConfig`, `RobotCore`, and the Isaac side while the real C++ runs `driveActuator`. Running the shipping firmware drive path is the point of the milestone, and a Python reimplementation would drift from it.

## 4. `SimulatedRobotConfig`, `RobotCore`, `SimulatedActuator`

`SimulatedActuator` is one device linking a slot's drive pins `{pinPwmR, pinPwmL, pinEn}` to one IsaacSim joint by name and resolved index. Given the recorded pin state it computes the signed command from section 1 and applies it to the joint. Its read side, mapping joint position back to `analogRead(pinPot)` and current-sense to `analogRead(pinIS)`, arrives in Task 2.

`SimulatedRobotConfig` composes a robot by binding all 6 slots' pins from `board_pins.h` to `SimulatedActuator`s, each mapped to a named crab joint from section 5. This is the declarative assemble-a-robot object the acceptance test builds.

`RobotCore` owns the game loop. Each tick it advances the firmware drive, for example `ActuatorManager::handleJog(...)` from the test's jog commands, producing fresh pin writes; then for each `SimulatedActuator` reads its recorded pins, computes the signed command, and sets the Isaac joint target; then steps IsaacSim physics. Task 1's loop drives only, and Task 2 adds reading joint positions back to feed the pots.

The PWM to motion model is linear. With `cmd` in [−255, 255], map to a joint target using one of the IsaacLab Articulation setters from section 5 with `speed = (cmd / 255) × max_joint_speed`, so −255 is a full-speed retract, +255 a full-speed extend, +122 roughly 48% extend, and 0 or EN low a hold. Real actuators behave nonlinearly, and modeling that is a later milestone.

## 5. Driving the joint in IsaacSim

The crab robot is an Articulation defined by `assets/crab_simple.usda` and configured in `parkour/parkour_tasks/parkour_tasks/crab_hexapod_task/config/crab_hex/crab_hex_scene_cfg.py` (~L67-147). Each leg has 3 revolute joints, and the 18 joint names follow `{LEG}_{Segment}_RevoluteJoint` with LEG in {FL, FR, ML, MR, RL, RR}:

| Joint | USD name pattern | Axis |
|---|---|---|
| Hip yaw | `{LEG}_Body_Hip_RevoluteJoint` | Z |
| Hip pitch | `{LEG}_Hip_Femur_RevoluteJoint` | X |
| Knee | `{LEG}_Femur_Tibia_RevoluteJoint` | X |

Resolve joint indices via regex rather than assuming ordering: `indices, names = robot.find_joints("FL_.*_RevoluteJoint")`, or a specific name, or `".*_Body_Hip_RevoluteJoint"` for all yaws. `find_joints` lives in `IsaacLab/source/isaaclab/isaaclab/assets/articulation/articulation.py` (~L244).

The setters in that same file each take `(target, joint_ids=..., env_ids=...)`, buffer the value, and flush on the env's `write_data_to_sim()` inside `env.step`:

- `set_joint_velocity_target(...)` (~L1081) is the natural PWM-to-velocity map for this task.
- `set_joint_position_target(...)` (~L1057) is what the existing policy action term uses.
- `set_joint_effort_target(...)` (~L1105) injects torque directly, if that suits better.

Targets apply on the next physics step, so `RobotCore`'s loop calls `env.step()`, or `sim.step()` followed by `robot.write_data_to_sim()`. A 6-motor single-board test can run a single-env scene (`num_envs=1`) using the crab task's PLAY config (`crab_hex_env_cfg.py` ~L310-376) or a minimal standalone loader. The sim launch scaffolding to reuse is `hal/server/isaac/main.py` (`AppLauncher` ~L235, control loop ~L523) or `parkour/scripts/rsl_rl/play.py` (~L51, L179-210).

## 6. Verify

- Host build: the actuator drive path compiles and links on the host against the minimal output shim, and the AVR `arduino-cli` build still succeeds (`make -C firmware compile-firmware`).
- Single-leg jog: command one actuator to extend with `manualDrive(+pwm)` and assert the linked Isaac joint's position increases; retract with `−pwm` and assert the reverse; command `+122` and assert roughly 48% of full speed; drop `EN` with pwm 0 and assert it holds. Read `robot.data.joint_pos` directly, since the pot read arrives in Task 2.
- Isolation: command leg 3 only, and assert legs 0-2 and 4-5 stay put.
- All six: a parametrized test over the 6 slots, each jogged extend, retract, and hold, asserting direction and proportional speed, headless and without hardware.
