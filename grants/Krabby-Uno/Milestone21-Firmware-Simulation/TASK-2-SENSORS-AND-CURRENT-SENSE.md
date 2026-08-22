# Task 2 — digitalRead / analogRead: sensors and current-sense

Goal: build the read side of the Arduino API on top of Task 1's drive side — `analogRead` for pot position and current-sense, `digitalRead`, and the system-level calls the firmware reads through (`EEPROM`, the hall-encoder edge counter, and `millis()` timing). The headline deliverable is a current-sense signal derived from IsaacSim physics: when a foot loads the ground, `analogRead(pinIS)` rises in proportion to the force the leg carries. The goal is a sensible, tunable output, and fidelity belongs to a later milestone.

Outputs
- Native implementations for `digitalRead`, `EEPROM` (`put`, `get`, `read`, `update`), the hall edge counter, and `millis()` timing in the firmware host build, with unit tests.
- A physics-derived current-sense mapper taking the joint wrench, or actuator effort, from IsaacSim onto the `analogRead(pinIS)` 0-1023 scale at the tuned breakpoints.
- Hall edge-count synthesis from simulated joint motion, scaled so a full min-to-max travel matches the real robot's encoder triggers, making `saf` telemetry realistic.
- Per-motor linear-pot calibration variance, with each motor's min and max stop values assigned at `SimulatedRobotConfig` instantiation and fixed thereafter, so auto-calibration discovers realistic per-motor limits.
- EEPROM as an in-memory byte array holding calibration `CalData` at byte 0 and board role at bytes 32-33, with `saveCalibration`/`loadCalibration` and `saveRole`/`loadRole` round-tripping within a run.
- A manual wrench-tuning viewer harness and unit tests validating the current-sense mapping.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-2---digitalread--analogread-sensors-and-current-sense).)

---

Note: line numbers are pointers to the right function and may drift. The end-to-end `python -m firmware.gui` demonstration needs serial and arrives in Task 3.

---

## 1. `digitalRead`, `millis`, and system timing

`digitalRead(pin)` matters in one place: `LinearActuator::printTelemetry` reads `pinEn` (`actuator_manager.h` ~L206) and prints it as both `enL` and `enR`. The simulator's pin model already records `digitalWrite` state from Task 1, so `digitalRead` returns it.

`millis()` drives ramp timing (`update()` ~L148-150), stall detection (`isStalled` ~L169-190), the telemetry cadence (`arduino.ino` ~L426-433), and the role-election timeout (~L171). Back it with a monotonic sim clock advanced by `RobotCore`'s loop, tied to accumulated `env.step_dt` so sim time and firmware time agree. Consistency matters here: ramp behavior and telemetry cadence both read wrong when `millis()` and physics time diverge.

## 2. EEPROM

The firmware uses EEPROM for two things (`arduino.ino` ~L25-27; `actuator_manager.h` `CalData` ~L356-362):

| Bytes | Contents | Accessed by |
|---|---|---|
| 0-25 | `CalData` = `int minVals[6]; int maxVals[6]; int magic;` (magic `0xDEADBEEF`) | `saveCalibration` (`EEPROM.put(0, data)`, ~L527), `loadCalibration` (`EEPROM.get(0, data)`, ~L534) |
| 32 | role magic `0xAB` | `saveRole`/`loadRole` (`arduino.ino` ~L30-44, `EEPROM.update`/`read`) |
| 33 | `BoardRole` value | same |

Back these with an in-memory byte array per emulator instance. Task 2 needs the `EEPROM.put`/`get`/`read`/`update` round-trip to work within a run: `saveCalibration` followed by `loadCalibration` returns the `CalData` written, with magic `0xDEADBEEF` valid, and `saveRole` followed by `loadRole` returns the role byte and magic `0xAB`. Cross-reboot persistence, the `ROLE_HINT:` serial print, and role election all need serial and belong to Task 3, so this task's tests exercise the byte round-trip directly.

## 3. Position sensors — pot calibration variance and hall edge counts

Both position sensors should behave like real hardware, because the firmware's auto-calibration (`ActuatorManager::updateCalibration`, `actuator_manager.h` ~L371-516) discovers each motor's limits by driving to stalls and recording `minStop` and `maxStop`. Per-motor variance is what makes that path realistic; a clean ideal 0-1023 on every motor exercises it trivially.

### Linear pots — per-motor calibration variance

Task 2 implements `analogRead(pinPot)`, starting with the basic position map from joint position onto 0-1023. That map closes the control loop: Task 1 drove open-loop with no pot read, so `getPos()`, the `update()` P-controller, and auto-calibration all begin working here.

Each motor then gets its own fixed calibration band, assigned once at `SimulatedRobotConfig` instantiation and constant thereafter, mimicking the POT calibration each unit gets from the manufacturer:

- Min stop maps to an `analogRead` value drawn from a distribution centered low, roughly 80 to 220 for that motor.
- Max stop maps to a value drawn from a distribution centered high, roughly 800 to 1000.
- Between the stops, `analogRead(pinPot)` interpolates the joint's travel fraction across that motor's `[min_adc, max_adc]` band, and outside the stops it saturates at the endpoint.
- The reading is deterministic per motor, so the same joint position always yields the same ADC value, while varying across motors at config instantiation. Seed the randomization from the `SimulatedRobotConfig` so a given config reproduces run to run.

When the firmware runs auto-calibration in sim it should therefore discover a different `minStop` and `maxStop` per motor, as on the real robot, and persist those values to EEPROM as described in section 2.

### Hall encoder — synthesize edge counts, calibrated to the real robot

On hardware, `hall_hw.cpp` counts pin-change interrupts on `PINB`, `PINK`, and `PINC` via `PCMSK*`, `PCICR`, and `ISR(PCINTn_vect)`, exposing two functions in `hall_hw.h`: `void hallHwInit()` and `uint32_t hallHwGetEdgeCount(uint8_t hallSlot)`. Those two are the only external callers (`arduino.ino` ~L246 init; `actuator_manager.h` ~L216 in telemetry), so replace the whole file with a native shim. A compile-time return-0 stub may already exist from Task 1.

The shim synthesizes an edge count from simulated joint motion by accumulating the absolute change in joint angle per slot, scaled so a full min-stop to max-stop travel produces the same total number of edges as the real robot. Obtain that number, the average encoder triggers between the stops, from James, or read it off a real robot via the fleet manager by watching the `saf` field in live telemetry while driving a joint stop to stop. Set the per-radian edge rate so the sim's `saf` matches, advancing monotonically as the joint moves and holding when it stops. Modeling AVR interrupt registers stays out of the picture entirely. Until James's figure is available, use a documented placeholder constant so the hall path stays testable, and swapping in the real number is a one-line tune.

## 4. Physics current-sense — the core of this task

On hardware, `analogRead(pinIS)` reads the motor-driver current-sense output on the 0-1023 ADC scale, EMA-smoothed into `avgIS` (`updateSensors` ~L77-85, `alphaIS = 0.10`) and printed as the `current` field. Motor current is proportional to torque, so the sim equivalent is a joint load signal.

### Source: joint reaction wrench (primary)

`robot.data.body_incoming_joint_wrench_b` gives the spatial reaction wrench transmitted through each link's inbound joint, shape `(num_envs, num_bodies, 6)` as `[Fx, Fy, Fz, Tx, Ty, Tz]` in the parent-body frame (`IsaacLab/source/isaaclab/isaaclab/assets/articulation/articulation_data.py` ~L719-733, backed by `get_link_incoming_joint_force()`). Index the child link of the joint in question, resolved via `robot.find_bodies(...)`.

For how much force the leg is carrying, the hip link (`{LEG}_Hip`) carries the whole leg's load into the chassis, while the knee (`{LEG}_Tibia` or `{LEG}_Femur`) reflects load past that joint. Take the wrench magnitude, or the axial force or torque component, as the raw load. This is the signal that rises when a foot presses into the ground, and rises further when a leg starts lifting the body.

### Source: actuator effort (cheaper secondary proxy)

`ParkourDCMotor` computes a per-joint `applied_effort` (`parkour/parkour_isaaclab/actuators/parkour_actuator_pd.py` ~L48-64), the commanded and clipped motor torque. It is the direct analog of how hard a motor is working and is cheaper to read than the wrench, so it serves as a cross-check or fallback.

### Also available: foot contact force

`ParkourHexContactSensor.data.net_forces_w` (`parkour/parkour_tasks/parkour_tasks/crab_hexapod_task/sensors/parkour_hex_contact_sensor.py` ~L124) gives external ground-reaction force at each `{LEG}_Footpad`. It captures ground contact while omitting the internal load path from the leg's own gravity and inertia, which is why the wrench serves better as the primary. One heads-up: `IsaacSimHalServer._cache_references` assigns the contact sensor to a typo'd `self.contacvt_sensor` at `hal/server/isaac/hal_server.py` ~L317, leaving `self.contact_sensor` unset. Fix that if current-sense is routed through the HAL server.

### Mapping to the 0-1023 scale

Map the raw load onto the target breakpoints with named constants:

| Condition | Target `pinIS` (of 1023) |
|---|---|
| No movement (idle, EN low or holding) | 0 |
| Moving, no ground contact (leg swinging) | ~2 |
| Basic ground contact | 4 |
| Leg lifting significant force (offloading other legs) | ~30 |

The band is deliberately compressed. A real ADC swings far higher, and the tuning target here is a readable signal that tracks what the robot is doing. Return the mapped value from `analogRead(pinIS)`, and the firmware's existing `avgIS` EMA smooths it. Keep the breakpoints as tunable constants so a later milestone can recalibrate.

### Tuning method (manual)

Tuning happens by eye: run a viewer, move a leg manually, and watch the force. Assemble the harness from existing pieces.

1. Launch a windowed single-env sim, reusing `hal/server/isaac/main.py --joystick --robot hex --usd assets/crab_simple.usda` (`AppLauncher` ~L235, loop ~L523) or a `parkour/scripts/rsl_rl/play.py`-style loader with `--num_envs 1` and windowed rendering. The crab PLAY and viewer configs live in `crab_hex_env_cfg.py` (~L31-36 viewer, ~L310-376 PLAY variants).
2. Perturb one leg with `robot.set_joint_velocity_target(target, joint_ids=robot.find_joints("FL_.*_RevoluteJoint")[0])`, or `write_joint_state_to_sim(...)` for hard scrubbing. The `carb.input` gamepad pattern in `parkour/scripts/rsl_rl/demo.py` (~L131-214) is a template for interactive control with a follow-cam.
3. Read `body_incoming_joint_wrench_b[:, body_ids]` each step and watch it as you press the foot into the floor and as the leg takes body weight, setting the breakpoint constants so the mapped `pinIS` lands on 0, ~2, 4, and ~30 at those conditions.

## 5. Where the value surfaces

Within this task, validate current-sense two ways that read the sim directly: unit tests asserting the mapping at each breakpoint, and the manual wrench viewer from section 4.

The full flow to the GUI runs `analogRead(pinIS)` to `avgIS` to the telemetry `current` field, then the SDK `_reader_loop`, `JointTelemetry.current`, `KrabbyMCUSDK.joints[name].current`, the GUI `_poll_telemetry` on a 100 ms tick, and finally the "Cur" column (`firmware/gui/app.py` ~L63, L143-159). Task 2 builds everything up to the telemetry `current` field, and Task 3's serial and SDK loop carry it the rest of the way. The GUI mirrors the field verbatim, so once serial carries a load-responsive `current` the column shows it, which is a Task 3 acceptance criterion.

## 6. Verify

- `digitalRead`: drive EN high and low, and assert telemetry `enL` and `enR` follow.
- EEPROM: run calibration and assert `saveCalibration` to `loadCalibration` round-trips `CalData` with a valid magic; write a role byte via `saveRole` and assert `loadRole` returns it along with magic `0xAB`. Reboot behavior, election, and `ROLE_HINT` belong to Task 3.
- Hall: move a joint and assert `saf` advances in proportion to travel and freezes when the joint stops; drive one joint stop to stop and assert the total `saf` delta matches the real-robot figure from James within tolerance.
- Pot variance: instantiate a config and assert each motor's min-stop and max-stop `analogRead` values are fixed, distinct across motors, and land in the 80-220 and 800-1000 bands; re-instantiate the same config and assert the values reproduce; run auto-calibration and assert it discovers per-motor `minStop` and `maxStop` that persist across a simulated reboot.
- `millis`: assert ramp cadence and the 50 ms telemetry interval track sim time.
- Current-sense: in the viewer, exercise the four conditions and assert mapped `pinIS` lands near 0, 2, 4, and 30; unit-test the mapping function at each breakpoint.
- Telemetry fields well-formed: the sensor values this task produces (`pot`, `current`, `saf`, EN) are the fields Task 3's telemetry line emits, so confirm they hold sane values and Task 3's wire format parses cleanly.
