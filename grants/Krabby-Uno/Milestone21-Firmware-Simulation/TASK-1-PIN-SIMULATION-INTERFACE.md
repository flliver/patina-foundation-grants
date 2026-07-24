# Task 1 — RobotCore loop + digitalWrite: drive the actuators

Goal: Build the **output/drive side only** — the `RobotCore` game loop plus the Arduino output calls (`pinMode`/`digitalWrite`/`analogWrite`, and `millis` for the loop clock) — so the firmware's `driveActuator()` pin writes move IsaacSim joints. Jog each of the 6 legs open-loop and watch them move in sim. **Nothing on the read side is in Task 1:** `analogRead` (pot position, current-sense) and `digitalRead` are stubbed here and built in Task 2, so there is deliberately no position feedback, no closed-loop P-controller, and no calibration yet — those all depend on reads and belong to Task 2. Scope: `SimulatedRobotConfig` + `SimulatedActuator` + `RobotCore`, one board, 6 motors, `KRABBY_PIN_REV` 3.

Outputs
- A minimal host Arduino shim for the **output** calls (`pinMode`/`digitalWrite`/`analogWrite` + `millis`); `analogRead`/`digitalRead` stubbed (real in Task 2) so the drive path compiles.
- A `SimulatedActuator` linking a slot's **drive pins** (PWM_R/PWM_L/EN) to one IsaacSim joint, with a linear PWM→speed model.
- A `SimulatedRobotConfig` binding the 6 REV-3 slots' drive pins to actuator devices, and a `RobotCore` game loop that reads recorded pin state, drives joints, and steps physics.
- Unit tests: define a 6-motor config, attach to `RobotCore`, jog each leg extend/retract/hold, and assert the linked Isaac joint moves in the correct direction at the correct proportional speed (verified by inspecting the sim joint directly).

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-1---robotcore-loop--digitalwrite-drive-the-actuators).)

---

**NOTE:** Exact line numbers below are from the repo at the time of writing and may drift; treat them as pointers to the right function, not literal addresses.

---

## 1. The output seam — the pin writes `driveActuator` makes

Everything the drive path does to hardware goes through a handful of Arduino **output** calls in `firmware/arduino/actuator_manager.h`. The one that matters is `LinearActuator::driveActuator(int pwm, int pwmDeadband)` (~L224–244):

```cpp
if (abs(pwm) < deadband) { digitalWrite(pinEn, LOW);  analogWrite(pinPwmR, 0);        analogWrite(pinPwmL, 0); }        // stop/coast
else if (pwm < 0)        { digitalWrite(pinEn, HIGH); analogWrite(pinPwmR, 0);        analogWrite(pinPwmL, abs(pwm)); } // retract (Left)
else                     { digitalWrite(pinEn, HIGH); analogWrite(pinPwmR, pwm);      analogWrite(pinPwmL, 0); }        // extend (Right)
```

Only one of R/L is ever nonzero, and `EN` gates whether the H-bridge is live. **The sim records these writes and interprets `(EN==HIGH) × (PWM_R − PWM_L)` as a signed motor drive** for the linked joint. (The `EN==HIGH` test uses the state the sim recorded from `digitalWrite` — it does *not* call the firmware's `digitalRead`, which is Task 2.) The calls Task 1 must service:

| Arduino API | Called at (`actuator_manager.h`) | Sim responsibility |
|---|---|---|
| `pinMode(pin, mode)` | ~L57–61 | record direction; no-op OK |
| `analogWrite(pwmPin, 0–255)` | ~L66–67, 229–242 | capture per-pin PWM duty |
| `digitalWrite(enPin, HIGH/LOW)` | ~L68, 228, 234, 240 | record EN state; gate the slot's motor |
| `millis() → ulong` | ~L148–150 (ramp), loop | monotonic sim clock (ms) |

**Stubbed in Task 1 (implemented in Task 2):** `analogRead(pinPot)` / `analogRead(pinIS)` (pot position + current-sense) and `digitalRead(pinEn)`. Because they are stubbed, drive the actuators via the **open-loop jog** path — `ActuatorManager::handleJog()` → `LinearActuator::manualDrive()` (~L115–128) → `driveActuator()` — which sets `currentPwm` directly and never calls `updateSensors()`/`analogRead`. The closed-loop `update()` path (P-controller reading `analogRead(pinPot)`) is Task 2.

## 2. The pin map (`board_pins.h`) — REV 3 only

Pins are selected at compile time by `KRABBY_PIN_REV`; the emulator only needs **Rev 3** (the current Uno v0.2 board). PWM pins are identical across revisions; EN pins are Rev-3 values below. The POT/IS analog pins are listed here too but are **only read in Task 2** — Task 1 binds them in the config but does not read them.

| Slot | PWM_R | PWM_L | EN (Rev 3) | POT (Task 2) | IS (Task 2) |
|---|---|---|---|---|---|
| S0 | D2 | D3 | D22 | A0 | A6 |
| S1 | D4 | D5 | D24 | A1 | A7 |
| S2 | D6 | D7 | D26 | A2 | A8 |
| S3 | D8 | D9 | D23 | A3 | A9 |
| S4 | D10 | D11 | D25 | A4 | A10 |
| S5 | D12 | D13 | D27 | A5 | A11 |

The `SimulatedRobotConfig` binds these pins per slot, sourced from `board_pins.h` rather than re-hardcoded — so a future pin revision is just a `board_pins.h` change plus the matching binding update in the config, with no emulator-core change.

## 3. The minimal host build (output side, new — does not exist today)

The firmware currently builds **only** for AVR via `arduino-cli` (`firmware/Makefile`). There is **no** host compilation and **no** C++ test harness anywhere in the repo (no ArduinoFake / EpoxyDuino / arduino_ci / PlatformIO / simavr). Task 1 stands up **just enough to run the actuator drive path** — nothing more.

What Task 1 needs:
1. **A minimal `Arduino.h` shim** implementing only the output calls `driveActuator`/`manualDrive` use: `pinMode`, `digitalWrite`, `analogWrite`, `millis` (plus the `constrain`/`abs` macros). These route to the simulator's pin/clock model.
2. **Stubs** for the reads and system calls the actuator translation unit references but that Task 1 never exercises — `analogRead`, `digitalRead`, `EEPROM.h`, `hallHwGetEdgeCount`. Return-constant / no-op stubs resolve the symbols so the drive path compiles; **do not implement them here** (Task 2 implements the reads; serial is Task 3).

`actuator_manager.h`/`command.h` and the drive math are portable C++ and compile unchanged against the shim. Prefer to compile the actuator layer into a host library and drive it from Python (the sim/IsaacSim side is Python) — a C-ABI or pybind11 boundary lets Python own the `SimulatedRobotConfig`/`RobotCore`/Isaac side while the real C++ runs `driveActuator`. Running the *actual shipping firmware* drive path (not a Python reimplementation that would drift) is the point of the milestone.

## 4. `SimulatedRobotConfig`, `RobotCore`, `SimulatedActuator`

- **`SimulatedActuator`** — one device linking a slot's **drive pins** `{pinPwmR, pinPwmL, pinEn}` to one IsaacSim joint (name + resolved index). Given the recorded pin state, it computes the signed command (§1) and applies it to the joint. (Its read side — mapping joint position back to `analogRead(pinPot)` and current-sense to `analogRead(pinIS)` — is added in Task 2.)
- **`SimulatedRobotConfig`** — composes a robot by binding all 6 slots' pins (from `board_pins.h`) to `SimulatedActuator`s, each mapped to a named crab joint (§5). This is the declarative "assemble a robot" object the acceptance test builds.
- **`RobotCore`** — owns the game loop. Each tick: (1) advance the firmware drive (e.g. `ActuatorManager::handleJog(...)` from the test's jog commands), producing fresh pin writes; (2) for each `SimulatedActuator`, read its recorded pins, compute the signed command, and set the Isaac joint target; (3) step IsaacSim physics. In Task 1 the loop is **drive-only** — reading joint positions back to feed pots is the Task 2 read side.

### The PWM → motion model (linear)
`cmd ∈ [−255, 255]`. Map to a joint target using one of the IsaacLab Articulation setters (§5) with `speed = (cmd / 255) × max_joint_speed`. `−255` → full-speed retract, `+255` → full-speed extend, `+122` → ~48% extend, `0` or `EN` low → hold. Assume linear dynamics (real actuators are not linear; that is a later milestone).

## 5. Driving the joint in IsaacSim

The crab robot is an Articulation defined by `assets/crab_simple.usda` and configured in `parkour/parkour_tasks/parkour_tasks/crab_hexapod_task/config/crab_hex/crab_hex_scene_cfg.py` (~L67–147). Each leg has 3 revolute joints; the 18 joint names follow `{LEG}_{Segment}_RevoluteJoint` with **LEG ∈ {FL, FR, ML, MR, RL, RR}**:

| Joint | USD name pattern | Axis |
|---|---|---|
| Hip yaw | `{LEG}_Body_Hip_RevoluteJoint` | Z |
| Hip pitch | `{LEG}_Hip_Femur_RevoluteJoint` | X |
| Knee | `{LEG}_Femur_Tibia_RevoluteJoint` | X |

**Resolve joint indices via regex — do not assume ordering:** `indices, names = robot.find_joints("FL_.*_RevoluteJoint")` (or a specific name, or `".*_Body_Hip_RevoluteJoint"` for all yaws). `find_joints` lives in `IsaacLab/source/isaaclab/isaaclab/assets/articulation/articulation.py` (~L244).

**Setters** (same file), each `(target, joint_ids=..., env_ids=...)`, buffered and flushed on the env's `write_data_to_sim()` (inside `env.step`):
- `set_joint_velocity_target(...)` (~L1081) — the natural PWM→velocity map for this task.
- `set_joint_position_target(...)` (~L1057) — what the existing policy action term uses.
- `set_joint_effort_target(...)` (~L1105) — direct torque injection (optional).

Targets set into buffers only apply on the next physics step, so `RobotCore`'s loop must call `env.step()` (or `sim.step()` + `robot.write_data_to_sim()`). For a 6-motor single-board test you can run a single-env scene (`num_envs=1`) using the crab task's PLAY config (`crab_hex_env_cfg.py` ~L310–376) or a minimal standalone loader; the sim launch scaffolding to reuse is `hal/server/isaac/main.py` (`AppLauncher` ~L235, control loop ~L523) or `parkour/scripts/rsl_rl/play.py` (~L51, L179–210).

## 6. Verify
- **Host build:** the actuator drive path compiles and links on the host against the minimal output shim; the AVR `arduino-cli` build still succeeds (`make -C firmware compile-firmware`).
- **Single-leg jog:** command one actuator to extend (`manualDrive(+pwm)`); assert the linked Isaac joint's position increases; retract (`−pwm`) and assert the reverse; command `+122` and assert ~48% of full speed; drop `EN` (pwm 0) and assert it holds. Verified by reading `robot.data.joint_pos` directly — there is no `analogRead` yet.
- **Isolation:** command leg 3 only; assert legs 0–2, 4–5 do not move.
- **All six:** parametrized test over the 6 slots, each jogged extend/retract/hold, asserting direction + proportional speed, headless, no hardware.
