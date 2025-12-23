# Patina Foundation Grant - Krabby-Uno Milestone 4: MCU Firmware

## Grant Overview
Build, document, and validate firmware for the Krabby-Uno robot so that all 18 joints are driven by three Arduino Megas (one per leg pair) under a single USB link to the Jetson Orin. The leader Mega receives normalized joint targets from the KrabbyMCU Python package (to be created as part of grant), converts them to per-motor PWM based on calibrated limits and gearbox multipliers, actuates its legs, forwards commands to the two follower Megas, aggregates joint feedback from all legs, and streams positions back to the Orin at ~50 Hz. Hardware layouts live in `krabby-research/hardware/diagrams/` for wiring and pin mapping reference.

## Why is this Important?
- Establishes a reproducible firmware stack that exercises the full electromechanical chain (Orin -> USB -> Mega -> H-Bridge -> motor) before higher-level gait code lands.
- Forces clear normalization and scaling rules (-1..1 yaw, 0..1 linear) so controls and simulation stay aligned.
- Provides a tested template for per-leg configuration (gear ratios, angle limits, PWM mapping) that scales from one motor to the full 18-joint rig.
- Documents a self-serve path (SETUP.md) so contributors can bring up hardware without hidden steps.
- Advances the open, community-owned robotics stack: by shipping transparent firmware and SDKs for a complex legged platform, we lower the barrier for students, hobbyists, and researchers to experiment with embodied AI without proprietary lock-in.
- Contributes to reproducible robotics research: clear configs, wiring, and open-source firmware let others replicate results, build on them, and validate safety and performance claims in the open.
- Connects Krabby's broader goals: this milestone bridges simulation work to real hardware, turning Krabby into a tangible, hackable platform where community members can test locomotion, control, and embodied intelligence ideas on affordable parts.

## Tasks
### Task 1 - Single-motor USB-to-PWM chain
#### Narrative
Bring up the end-to-end path from Jetson Orin (Ubuntu) through USB to one Arduino Mega controlling a single hip yaw motor via an H-bridge. Accept a normalized yaw command in [-1, 1], apply a configurable gearbox multiplier (e.g., 12x), and map to PWM that sweeps from -30 deg to +30 deg at the output shaft (e.g. one full motor revolution if gear box multiplier is set to 12 --> 30 * 12 = 360). Echo the measured/estimated joint angle back to the Orin at ~50 Hz by fetching normalized quadrature encoder and/or slide potentiometer values (again matching [-1, 1]) so the SDK callback can report live position.
#### Acceptance Criteria
- Commit to `krabby-research/firmware/` with the Arduino sketch for one hip yaw motor and a step-by-step `SETUP.md` (Ubuntu 22.04 + Arduino toolchain + wiring + run instructions). No hidden steps; call out prerequisites (packages, udev rules, required libraries).
- From the Orin, a KrabbieMCUSDK Python snippet can send -1, 0, and +1 yaw targets and see the motor sweep left 30 deg, center, right 30 deg (or scaled by the configured gearbox multiplier) with matching returned telemetry.
- Gearbox multiplier, min/max angles, PWM bounds, and USB port selection are configurable without code edits (e.g., constants or a small config header).
- Telemetry loop returns joint position at ~50 Hz without dropped samples for a 30-second run.

### Task 2 - Full leg pair on one Mega
#### Narrative
Expand the single-motor sketch to drive all six joints for one leg pair on a single Arduino Mega (hip yaw x2, hip linear x2, knee linear x2). Normalize inputs to [-1, 1] for yaw and [0, 1] for linear axes, then map to PWM per motor using per-joint calibration. Return all six joint positions in the response frame to the Orin via a new KrabbyMCU python package. The hip/knee linear motors (4x total) will be connected to the RobotPower Multimoto, and a linear potentiometer for feedback, while the yaw motors will each be connected to a BTN7960 h-bridge + DFRobot 12V motor w/ quadrature encoder (so two h-bridge, one per motor). 
#### Acceptance Criteria
- Commit to `krabby-research/firmware/` updates the Arduino firmware to control and report all six joints on one Mega, plus an updated `SETUP.md` with wiring pinouts, per-joint calibration instructions, and a one-command run example from the Orin.
- KrabbyMCU python package can issue a full joint vector for one leg pair and observe all six motors moving through their full configured ranges with matching telemetry at ~50 Hz.
- A testRange function is available that moves all six motor joints one at a time to its configured min, home, and max positions. Running this script is how we'll validate the whole thing. This should be run on setup() to move to the forward position slowly w/ pulsing until feedback on R_IS/L_IS is reached (depending on if it is left or right side motor), then it should internally store the end stop positions reached for each limb (and print to log).
- Hip motor will self-calibrate by moving upward until a stop is reached (leg fully vertical), then will use the current linear potentiometer value + a config value (i.e 85% of max potentiometer value) as the end stop (since it will not be possible to move the hip downward w/o hitting the floor)
- Knee motor will move down/outward (i.e. clockwise) until physical end stop is reached (when tibia and fibia are touching), then similar to hip, will use some configured value for the leg max movement (i.e. 90% of linear potentiometer value). Note, knee pot will be at smallest value when leg is extended, while hip will be at largest value when leg is extended. If you need a diagram for it let me know and I can do so.
- All pin outs must be recorded in a wiring diagram (similar to what was provided in task 1). Pin outs for liner pots + h-bridges should not block multimoto pins, and multimoto must be used in simulation or real world if sim is not available. 
- Config supports per-joint gearbox multipliers, soft limits, and PWM bounds without code rewrites; defaults documented in `SETUP.md`.
- Fix all TODOs in `krabby-research/firmware/SETUP.md`
- MCU Config files should be human readable in a single well organized config.h w/ clear documentation that can be uploaded separate from main firmware code.
- Error handling: invalid or out-of-range commands are clamped and logged as warnings, and the Mega continues streaming telemetry.
- Error handling: basic miswirings such as non-responding encoder, motor not moving, not getting power feedback from IS pins, unexpected movement behavior (i.e motor moving left when given command ot go right) should be immediately logged/warned/errored w/ clear human readable error information. 
- Testing: Python script w/ test behavior, including common miswiring w/ mocked outputs should be included in committed code
- `KrabbyMCU` Python package is added to `krabby-research/firmware/` with its own `pyproject.toml` (patterned after the HAL packages: `package-dir` and `packages` entries set so `pip install -e firmware` exposes the `krabby_mcu` module). Package includes: a Python API to send normalized joint vectors (yaw [-1,1], linear [0,1]) to the Mega, receive the six-joint telemetry callback/observer for that board, and a minimal CLI or example script demonstrating one-leg control end-to-end.
- All python code should be unit tested with 80%+ code coverage

### Task 3 - Three-Mega network with leader auto-detect and KrabbyMCU python controller package
#### Narrative
Wire three Arduino Megas (one per leg pair) so that one is the leader on the USB link to Orin and the other two are followers connected over serial. The leader distributes the 18-joint command vector, each Mega drives its local six joints, and all followers return joint positions to the leader for aggregation back to Orin at ~50 Hz. Leader detection is done via wiring (e.g., auto-detection of host serial to leader board, or if that's impossible, a pulled-up ID pin or jumper) so any board can be the leader without reflashing, and all MCUs can be flashed identically. Leader, if not properly connected to followers, should auto-identify and throw clear errors back to Orin via KrabbyMCU package. The KrabbyMCU package should be well tested, have good error handling, and have a mock controller mode for testing.
#### Acceptance Criteria
- Commit to `krabby-research/firmware/` contains the three-Mega firmware set and docs covering wiring for leader ID, serial connections, baud rates, and addressing, plus a refreshed `SETUP.md` that walks through detection, flashing, and bring-up from a clean Ubuntu/Jetson setup.
- Orin can send an 18-joint command vector via KrabbyMCU; the leader fans out commands to followers, all three Megas actuate their joints, and aggregated telemetry for all 18 joints returns at ~50 Hz for at least 60 seconds without desync.
- Leader detection works by wiring (e.g., reading a dedicated pin) and is documented with a test step to verify correct leader selection before running.
- Failure handling: if a follower is absent or silent, the leader reports which leg pair is missing, continues streaming available telemetry, and clamps or zeros commands to the missing pair (documented behavior). If it cannot communicate with an entire board, it should throw appropriate error information back to Orin via KrabbyMCU, and continue sending other commands/joint positions as appropriate.
- All code should be unit tested with 80%+ code coverage 
- All errors should be properly caught and thrown to KrabbyMCU w/ clear human readable values
- KrabbyMCU package should have a mock controller that instead of communicating over serial, maintains internal state, moves the 'motors' at roughly the same speed they would move in real life, and reports back a fake motor state. This makes the Orin SW unit testable w/o connecting to an actual MCU.




## Information
- Repositories: firmware and docs live in the `krabby-research/mcu/` github repository; grant doc lives in this repo under `grants/Krabby-Uno/Milestone4-MCU-Firmware/`.
- Hardware reference: see wiring and actuator layouts in `krabby-research/hardware/diagrams/`.
- Control contract: yaw commands are normalized to [-1, 1]; linear (hip/knee) commands normalized to [0, 1]. Firmware maps normalized values to PWM using per-joint limits, gearbox multipliers, and calibration constants. Review hardware wiring diagrams in the `hardware/diagrams` directory for more detail on specific motor information.
- Telemetry: joint positions streamed back to Orin at ~50 Hz; SDK exposes a callback/observer to read the latest joint states.
- Required platform: Ubuntu 22.04 with access to Jetson Orin (USB), Arduino Mega x3, matching H-bridges/drivers, and the Arduino toolchain (CLI or IDE). All prerequisites and dev steps must be in `SETUP.md`. All steps should be scriptable/executable via command line from VSCode.

## FAQ
- How is the leader chosen?  
  Ideally by auto-detecting that the board is connected to another device via serial, and is upstream of two other arduinos. If that's impossible, then by wiring: a dedicated ID pin or jumper is read on startup; high selects leader mode, low selects follower. The docs should include a quick test to confirm which board is leader before running.
- How do I change limits or gear ratios?  
  Edit the documented config block for gearbox multiplier, angle/linear limits, and PWM bounds per joint; no code changes required. `SETUP.md` lists defaults and examples.
- Can I bench test without motors?  
  Yes. A mock/sim mode should be provided (documented in `SETUP.md`) that responds with canned or ramped telemetry so the SDK path can be validated without hardware load.
- What happens if a follower drops?  
  Leader logs which leg pair is missing, continues sending to remaining boards, clamps commands for the missing pair, and still returns telemetry for connected legs so higher layers can react.
