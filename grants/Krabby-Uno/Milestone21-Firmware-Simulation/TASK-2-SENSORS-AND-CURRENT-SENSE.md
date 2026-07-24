# Task 2 — digitalRead / analogRead: sensors and current-sense

Goal: Build the **input/read side** of the Arduino API on top of Task 1's drive side — `analogRead` (linear-pot position variance + current-sense), `digitalRead`, and the system-level calls the firmware reads through (`EEPROM`, the hall-encoder edge counter, `millis()`/timing) — each unit-tested. The headline deliverable is a **current-sense signal derived from IsaacSim physics**: when a foot loads the ground, `analogRead(pinIS)` rises in proportion to the force the leg is carrying, tuned low and sensible. Validate here with unit tests and a manual wrench-tuning viewer (both read the sim directly); the full `python -m firmware.gui` end-to-end demonstration comes once serial is wired in **Task 3**. Physical accuracy is explicitly **not** the goal — sensible, tunable outputs are.

Outputs
- Native implementations/shims for `digitalRead`, `EEPROM` (`put`/`get`/`read`/`update`), the hall edge counter, and `millis()`/system timing in the firmware host build, with unit tests.
- A physics-derived current-sense mapper: joint wrench (or actuator effort) from IsaacSim → `analogRead(pinIS)` 0–1023, tuned to the target scale.
- Hall edge-count synthesis from simulated joint motion, **scaled so a full min→max travel matches the real robot's encoder triggers** (count from James / a real robot via the fleet manager) so `saf` telemetry is realistic.
- Per-motor **linear-pot calibration variance**: each motor's `analogRead(pinPot)` min/max-stop values are randomized at `SimulatedRobotConfig` instantiation (min-stop ~80–220, max-stop ~800–1000), fixed per motor, so auto-calibration discovers realistic per-motor limits.
- EEPROM as an **in-memory** byte array (calibration `CalData` at byte 0, board role at bytes 32–33); `saveCalibration`/`loadCalibration` and `saveRole`/`loadRole` round-trip within a run (no cross-reboot persistence needed at this stage).
- A manual wrench-tuning viewer harness (move a leg, watch the joint wrench) plus unit tests validating the current-sense mapping. (The `python -m firmware.gui` end-to-end demo is Task 3, once serial carries the value to the SDK/GUI.)

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-2---digitalread--analogread-sensors-and-current-sense).)

---

**NOTE:** Line numbers are pointers to the right function and may drift.

---

## 1. `digitalRead`, `millis`, and system timing

- **`digitalRead(pin)`** is used in exactly one place that matters: `LinearActuator::printTelemetry` reads `pinEn` (`actuator_manager.h` ~L206) and prints it as both `enL` and `enR`. The sim's pin model already records `digitalWrite` state (Task 1); `digitalRead` returns it.
- **`millis()`** drives ramp timing (`update()` ~L148–150), stall detection (`isStalled` ~L169–190), the telemetry cadence (`arduino.ino` ~L426–433), and the role-election timeout (~L171). Back it with a monotonic sim clock advanced by `RobotCore`'s loop (tie it to accumulated `env.step_dt` so sim time and firmware time agree). Consistency matters: if `millis()` and physics time diverge, ramp/telemetry cadence will look wrong.

## 2. EEPROM

The firmware uses EEPROM for two things (`arduino.ino` ~L25–27; `actuator_manager.h` `CalData` ~L356–362):

| Bytes | Contents | Accessed by |
|---|---|---|
| 0–25 | `CalData` = `int minVals[6]; int maxVals[6]; int magic;` (magic `0xDEADBEEF`) | `saveCalibration` (`EEPROM.put(0, data)`, ~L527), `loadCalibration` (`EEPROM.get(0, data)`, ~L534) |
| 32 | role magic `0xAB` | `saveRole`/`loadRole` (`arduino.ino` ~L30–44, `EEPROM.update`/`read`) |
| 33 | `BoardRole` value | same |

Back these with an **in-memory** byte array (per emulator instance). Task 2 only needs the `EEPROM.put`/`get`/`read`/`update` round-trip to work within a run: `saveCalibration`→`loadCalibration` returns the `CalData` you wrote (magic `0xDEADBEEF` valid), and `saveRole`→`loadRole` returns the role byte + magic `0xAB`. **Cross-reboot persistence, the `ROLE_HINT:` serial print, and role election are Task 3** (they need serial) — keep them out of Task 2's tests, which exercise the byte round-trip directly, not a power-cycle.

## 3. Position sensors — pot calibration variance and hall edge counts

Both position sensors should behave like real hardware, because the firmware's auto-calibration (`ActuatorManager::updateCalibration`, `actuator_manager.h` ~L371–516) discovers each motor's limits by driving to stalls and recording `minStop`/`maxStop`. If every motor reads a clean ideal 0–1023, calibration is exercised trivially; presenting *per-motor* variance is what makes it realistic.

### Linear pots — per-motor calibration variance
Task 2 implements `analogRead(pinPot)`: first the basic position map (joint position → 0–1023) that **closes the control loop** — Task 1 drove open-loop with no pot read, so `getPos()`, the `update()` P-controller, and auto-calibration only start working now — then makes it realistic. Each motor gets its own fixed calibration band, **assigned once at `SimulatedRobotConfig` instantiation** and constant thereafter, mimicking the different POT calibration each unit gets from the manufacturer:
- **Min stop** → an `analogRead` value drawn from a distribution centered low, roughly **80–220** for that motor.
- **Max stop** → a value drawn from a distribution centered high, roughly **800–1000** for that motor.
- Between the stops, `analogRead(pinPot)` interpolates the joint's travel fraction across that motor's `[min_adc, max_adc]` band; outside the stops it saturates at the endpoint.
- The reading is **deterministic per motor** — the same joint position always yields the same ADC value for that motor — but **randomized across motors** at config instantiation. Seed the randomization from the `SimulatedRobotConfig` so a given config is reproducible run-to-run.

The point: when the firmware runs auto-calibration in sim it should discover a *different* `minStop`/`maxStop` per motor (as on the real robot), and those values persist to EEPROM (§2). This exercises the calibration + EEPROM path realistically instead of every motor landing on a clean 0–1023.

### Hall encoder — synthesize edge counts, calibrated to the real robot
On hardware, `hall_hw.cpp` counts pin-change interrupts on `PINB`/`PINK`/`PINC` via `PCMSK*`/`PCICR` and `ISR(PCINTn_vect)`, exposing only two functions (`hall_hw.h`): `void hallHwInit()` and `uint32_t hallHwGetEdgeCount(uint8_t hallSlot)`. Only these two are called externally (`arduino.ino` ~L246 init; `actuator_manager.h` ~L216 in telemetry). **Replace the whole file with a native shim** (a compile-time return-0 stub may already exist from Task 1) that synthesizes an edge count from simulated joint motion — accumulate `|Δjoint_angle|` per slot — **scaled so that a full min-stop→max-stop travel produces the same total number of edges as the real robot.** Get that number (the average encoder triggers between the min and max stops) **from James**, or read it directly off a real robot via the fleet manager (watch the `saf` field in live telemetry while driving a joint stop-to-stop). Set the per-radian edge rate so the sim's `saf` matches. `saf` advances monotonically as the joint moves and holds when it stops. This avoids modeling AVR interrupt registers entirely and gives realistic, real-robot-calibrated edge counts. Until James's number is available, use a documented placeholder constant so the hall path stays testable — swapping in the real figure is a one-line tune, not a blocker.

## 4. Physics current-sense — the core of this task

On hardware, `analogRead(pinIS)` reads the motor-driver current-sense output (0–1023 ADC), EMA-smoothed into `avgIS` (`updateSensors` ~L77–85, `alphaIS = 0.10`), and printed as the `current` field. Motor current is proportional to torque, so the sim equivalent is a **joint load** signal.

### Source: joint reaction wrench (primary)
`robot.data.body_incoming_joint_wrench_b` — the spatial reaction wrench transmitted through each link's inbound joint, shape `(num_envs, num_bodies, 6)` = `[Fx, Fy, Fz, Tx, Ty, Tz]` in the parent-body frame (`IsaacLab/source/isaaclab/isaaclab/assets/articulation/articulation_data.py` ~L719–733; backed by `get_link_incoming_joint_force()`). Index the **child link** of the joint you care about, resolved via `robot.find_bodies(...)`:
- For "how much force is the leg carrying," the **hip** link (`{LEG}_Hip`) carries the whole leg's load into the chassis; the **knee** (`{LEG}_Tibia`/`{LEG}_Femur`) reflects load past that joint.
- Take the wrench magnitude (or the axial force/torque component) as the raw load. This is the signal that rises when a foot presses into the ground and especially when a leg starts lifting the body.

### Source: actuator effort (cheaper secondary proxy)
`ParkourDCMotor` computes a per-joint `applied_effort` (`parkour/parkour_isaaclab/actuators/parkour_actuator_pd.py` ~L48–64) — the commanded/clipped motor torque. It is directly the "how hard is this motor working" analog to current and is cheaper than the wrench read; use it as a cross-check or fallback.

### Also available: foot contact force
`ParkourHexContactSensor.data.net_forces_w` (`parkour/parkour_tasks/parkour_tasks/crab_hexapod_task/sensors/parkour_hex_contact_sensor.py` ~L124) gives external ground-reaction force at each `{LEG}_Footpad`. This captures *ground contact* but not the internal load path (gravity/inertia of the leg), so the wrench is the better primary. (Heads-up: `IsaacSimHalServer._cache_references` currently mis-assigns the contact sensor to a typo'd `self.contacvt_sensor` at `hal/server/isaac/hal_server.py` ~L317, so `self.contact_sensor` is never set — fix if you route current-sense through the HAL server.)

### Mapping to the 0–1023 scale
Map the raw load onto the target breakpoints with named constants:

| Condition | Target `pinIS` (of 1023) |
|---|---|
| No movement (idle, EN low or holding) | 0 |
| Moving, no ground contact (leg swinging) | ~2 |
| Basic ground contact | 4 |
| Leg lifting significant force (offloading other legs) | ~30 |

This is deliberately compressed into a low band — the real ADC would swing far higher, but the tuning target here is a *sensible, readable* signal, not fidelity. Return the mapped value from `analogRead(pinIS)`; the firmware's existing `avgIS` EMA then smooths it. Keep the breakpoints as tunable constants so a later milestone can recalibrate.

### Tuning method (manual)
The notes call for tuning by eye: run a viewer, manually move a leg, and watch the force. Assemble the harness from existing pieces (there is no single-joint scrub viewer today):
1. Launch a windowed single-env sim — reuse `hal/server/isaac/main.py --joystick --robot hex --usd assets/crab_simple.usda` (`AppLauncher` ~L235, loop ~L523) or a `parkour/scripts/rsl_rl/play.py`-style loader with `--num_envs 1` and no `--headless`. The crab PLAY/viewer configs are in `crab_hex_env_cfg.py` (~L31–36 viewer, ~L310–376 PLAY variants).
2. Perturb one leg — `robot.set_joint_velocity_target(target, joint_ids=robot.find_joints("FL_.*_RevoluteJoint")[0])` or `write_joint_state_to_sim(...)` for hard scrubbing; the `carb.input` gamepad pattern in `parkour/scripts/rsl_rl/demo.py` (~L131–214) is a template for interactive control + follow-cam.
3. Read `body_incoming_joint_wrench_b[:, body_ids]` each step and watch it as you press the foot into the floor and as the leg takes body weight; set the breakpoint constants so the mapped `pinIS` lands on 0 / ~2 / 4 / ~30 at those conditions.

## 5. Where the value surfaces (and where it's demonstrated)

Within this task, validate current-sense two ways that read the sim directly — the unit tests (assert the mapping at each breakpoint) and the manual wrench viewer (§4: move a leg and watch `pinIS`). The **end-to-end `python -m firmware.gui` demonstration is Task 3**, because it needs serial. The full flow is: sim `analogRead(pinIS)` → `avgIS` → telemetry `current` field → SDK `_reader_loop` → `JointTelemetry.current` → `KrabbyMCUSDK.joints[name].current` → GUI `_poll_telemetry` (100 ms) → the **"Cur"** column (`firmware/gui/app.py` ~L63, L143–159). Everything left of "telemetry `current` field" is built here in Task 2; everything right of it is Task 3's serial + SDK loop. The GUI mirrors the field verbatim, so once serial carries the load-responsive `current`, the "Cur" column shows it — that demo is a Task 3 acceptance criterion.

## 6. Verify
- **`digitalRead`:** drive EN high/low; assert telemetry `enL`/`enR` follow.
- **EEPROM:** run calibration, assert `saveCalibration`→`loadCalibration` round-trips `CalData` (magic valid); write a role byte via `saveRole` and assert `loadRole` returns it (+ magic `0xAB`). No reboot / election / `ROLE_HINT` here — that's Task 3.
- **Hall:** move a joint; assert `saf` advances proportional to travel and freezes when stopped; drive one joint stop-to-stop and assert the total `saf` delta matches the real-robot figure from James (within tolerance).
- **Pot variance:** instantiate a config; assert each motor's min-stop and max-stop `analogRead` values are fixed, distinct across motors, and land in the ~80–220 / ~800–1000 bands; re-instantiate the same config and assert the values reproduce; run auto-calibration and assert it discovers per-motor `minStop`/`maxStop` that persist across a simulated reboot.
- **`millis`:** assert ramp cadence and 50 ms telemetry interval track sim time.
- **Current-sense (no serial):** in the viewer, exercise the four conditions and assert mapped `pinIS` ≈ 0 / ~2 / 4 / ~30; unit-test the mapping function at each breakpoint. The end-to-end `python -m firmware.gui` demo is Task 3, once serial carries the value.
- **Telemetry fields well-formed:** the sensor values this task produces (`pot`, `current`, `saf`, EN) are the fields Task 3's telemetry line emits — confirm they hold sane values so Task 3's wire format parses cleanly.
