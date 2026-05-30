# Task 3 — Whole-robot calibration sequence (auto-squat starting pose, all 18 joints)

**Time estimate: ~2.5 dev days (range 2–3.5).** Sub-table:

| Days | Sub-task |
|------|----------|
| 1 | Auto-squat pre-cal sequence: drive each leg to retract-hip / extend-knee / sweep-yaw, recording hip-min / knee-max / yaw-min / yaw-max for all 6 legs |
| 1 | Standard 7-step per-leg cal sequence (hip retract → knee retract → knee extend → yaw left → yaw right → yaw center → hip extend); composed entirely of Task 2's directional `calibrateJoint` |
| 0.5 | SDK + CLI (`calibrate-all`); failure-path testing |

Goal: Calibrate the entire robot end-to-end using Task 2's per-joint function, with a **deterministic auto-driven starting pose** that the firmware can reach from any state. No operator-set pre-pose required. The sequence drives every joint to one of its end stops as the first action, which doubles as recording half of the calibration data and parks the robot in a known geometry before the per-leg sweep begins. (Serial-number assignment is **out of scope** for this task — operators set serials via Task 1's `SET serial …`. The two operations are independent.)

Outputs
- `calibrateAll()` on the MCU side that runs the auto-squat → per-leg sweep → finalize sequence on all 18 joints.
- Auto-squat pre-cal sequence that doubles as cal-data capture for hip-min, knee-max, yaw-min, yaw-max on all 6 legs.
- Per-leg sub-sweep capturing the remaining stops (knee-min, hip-max) without operator intervention.
- SDK + CLI (`krabby-firmware calibrate-all`) that drives the sequence end-to-end. While the cal runs, the only firmware output is the existing joint telemetry stream plus any `ERR <joint> <code>` lines that fire — no separate progress format.

Acceptance Criteria
- **3a** — `calibrateAll()` runs the auto-squat + per-leg cal end-to-end and saves the per-joint cal data (from Task 2) to EEPROM for all 18 joints.
- **3b** — The implementation is composed entirely of Task 2's `calibrateJoint(name, direction=…)` calls plus pose-transition primitives (`moveJointTo`, `holdAll`). The standard 7-step per-leg sequence is the same for every leg (§3); there's no special-case logic per leg type — step 7 is a plain `calibrateJoint(extend)` that records hip max like any other end-stop.
- **3c** — Auto-squat starting pose: cal begins by driving every joint to a deterministic end-stop, with the **yaw direction chosen per leg to maximally splay the legs** so they stay out of each other's way during the per-leg sweeps that follow. Concretely:
  - Every hip retracts to its retract stop.
  - Every knee extends to its extend stop.
  - **Front yaws (FLHY, FRHY) rotate the leg fully forward** — for the left-side front leg, this is rotating to the right (top-down view); for the right-side front leg, rotating to the left. Both front legs end pointing out and forward.
  - **Rear yaws (RLHY, RRHY) rotate the leg fully aft** — the mirror direction of the fronts on each side. Both rear legs end pointing out and rearward.
  - **Middle yaws (MLHY, MRHY) rotate to center** (or stay where they are if already roughly centered) — the front legs splayed forward and rear legs splayed rearward give the middle legs maximum clearance without the middles having to move.

  Yaw min/max for every leg are still recorded as a side effect during the sweep (the firmware drives each yaw through both extremes to log them, then parks at the splayed-away extreme for the splay pose). The result: a maximally-splayed squat geometry, reached from any prior state, with hip-min / knee-max / yaw-min / yaw-max for every leg in EEPROM. No operator pre-pose required.
- **3d** — Per-leg standard 7-step cal sequence (see §3) executed in **middles-first** order: **middle-left, middle-right, then rear-left, front-left, rear-right, front-right**. The middles calibrate first because the front/rear legs can't move safely until the middle legs are calibrated and centered — and the auto-squat (3c) already splayed the fronts forward and rears rearward so the middles have clearance to sweep without leg-vs-leg interference. Once the middles are done, the front/rear legs return from their splayed positions and calibrate in any order.
- **3e** — Between legs the just-calibrated leg returns to the squat pose so the next leg starts from a known geometry.
- **3f** — On any per-joint failure during cal, the firmware emits a single `ERR <joint> <errorcode>` line on serial (same wire format as Task 2 §2f), halts the sequence, and calls `ActuatorManager::holdAll` to park every motor. No JSON, no `ok`/`stage` envelope — just the bare ERR line. The SDK aggregates ERR lines and translates them via Task 4's `explain_failures()` for operator-facing display.
- **3g** — On success, the robot returns to the auto-squat pose (or to a documented neutral pose) and `calibrateAll` returns success with the full per-joint summary.
- **3h** — `KrabbyMCUSDK.calibrate_all()` and `krabby-firmware calibrate-all` drive the sequence end-to-end. No firmware-side progress lines — the host watches the existing joint telemetry stream and any `ERR <joint> <code>` lines (Task 1 §5) to know what's happening; after the command completes, the SDK reads back the result via `GET calibration` (Task 4).

---

## 1. Joint naming and the hex layout

From `firmware/arduino/arduino.ino` (~L60–80) and `firmware/krabby_mcu.py` `JOINT_GROUP_NAMES` (~L56), the 18 joints are:

| Leg | Hip-yaw | Hip-pitch (linear) | Knee (linear) |
|---|---|---|---|
| Front-left | FLHY | FLHL | FLKL |
| Front-right | FRHY | FRHL | FRKL |
| Rear-left (back-left) | RLHY | RLHL | RLKL |
| Middle-left | MLHY | MLHL | MLKL |
| Rear-right (back-right) | RRHY | RRHL | RRKL |
| Middle-right | MRHY | MRHL | MRKL |

Hip-yaw on V0.2/V0.3 is a Xingli planetary-gear motor with Hall encoder; hip-pitch and knee are linear actuators with **either** a potentiometer **or** a Hall encoder — never both. Each motor has exactly one position sensor; `JointCal.sensor_type` (POT or HALL, auto-detected during cal — see Task 2) tells the firmware which.

## 2. The auto-squat starting pose (3c)

**Replaces the operator-set passive 90° pose from earlier drafts.** Insight: drive every joint to a fixed-direction end-stop, and you've simultaneously (a) reached a deterministic pose from any starting state and (b) recorded half the calibration data. The "squat" pose is:

- Every hip **fully retracted** (Hip-pitch driven to its retract end-stop).
- Every knee **fully extended** (Knee driven to its extend end-stop).
- Every hip-yaw **swept to each end-stop in turn** (records both yaw end-stops), parked at the **splay direction** that's right for that leg per 3c — front legs forward, rear legs aft, middle legs explicitly moved to center afterward (since the cal sweep alone doesn't land them centered).

This is exactly the leg orientation Fletcher's dictation described: hip retracted lifts the leg; knee extended unfolds the tibia outward; hip-yaw sweep captures the rotational range. After the sequence, the robot is in a wide, low pose with all 18 joints at known positions.

The pre-cal sequence is a one-pass loop over all 6 legs (order doesn't matter for this phase — each leg's hip / knee / yaw motions are mechanically independent). Note that each `calibrateJoint(joint, direction=X)` call **parks the joint at the X end-stop** (no automatic return-to-center) per the Task 2 §2 API; we just need to order the yaw calls so the leg ends at the right splay extreme:

```
for each leg in [RL, FL, ML, RR, FR, MR]:
    # Linear joints: park at their retract (hip) / extend (knee) stops naturally — that's the squat pose.
    calibrateJoint(legHL, direction=retract)             → records HL.sensor_min; HL parked at retract stop
    calibrateJoint(legKL, direction=extend)              → records KL.sensor_max; KL parked at extend stop

    # Yaw: pick the order so the leg ends up at the right splay extreme for its position.
    if leg in [FL, FR]:
        calibrateJoint(legHY, direction=aft)             → records aft end-stop
        calibrateJoint(legHY, direction=forward)         → records forward end-stop; HY parked at forward (splayed forward, 3c)
    elif leg in [RL, RR]:
        calibrateJoint(legHY, direction=forward)         → records forward end-stop
        calibrateJoint(legHY, direction=aft)             → records aft end-stop; HY parked at aft (splayed rearward, 3c)
    else:  # middle legs
        calibrateJoint(legHY, direction=forward)         → records forward end-stop
        calibrateJoint(legHY, direction=aft)             → records aft end-stop
        moveJointTo(legHY, 0.5)                          → middle legs centered (3c) — explicit, since cal alone parks at aft
```

This relies on Task 2's directional `calibrateJoint`: `direction=retract`/`extend` for linear joints and `direction=forward`/`aft` (the yaw equivalents of left/right per Task 2 §2) for yaw. Each directional call drives to that one stop, records it, and leaves the joint parked there — no implicit return-to-center.

Geometrically, after this phase the robot is resting low with **front legs splayed forward, rear legs splayed rearward, middle legs centered** — maximum clearance for the middles to do their per-leg sweep first.

## 3. Per-leg standard cal sequence (3d) — middles first

Each leg's per-leg calibration runs the **same standard 7-step sequence**, regardless of which leg. Anything §2's auto-squat happened to capture as a side effect is just re-recorded by this sequence — the standard sequence is the authoritative recording path.

```
def standardCalibrate(leg):
    1. calibrateJoint(legHL, direction=retract)   → records HL.sensor_min   (hip min stop)
    2. calibrateJoint(legKL, direction=retract)   → records KL.sensor_min   (knee min stop)
    3. calibrateJoint(legKL, direction=extend)    → records KL.sensor_max   (knee max stop)
    4. calibrateJoint(legHY, direction=left)      → records HY.sensor_min   (yaw left stop)
    5. calibrateJoint(legHY, direction=right)     → records HY.sensor_max   (yaw right stop)
    6. moveJointTo(legHY, 0.5)                    → return yaw to center
    7. calibrateJoint(legHL, direction=extend)    → records HL.sensor_max   (hip max stop)
```

The whole-robot sequence runs this for every leg in **middles-first** order (3d):

```
# Phase 1: middles first — front/rear are splayed out of the way from §2
for each leg in [ML, MR]:
    standardCalibrate(leg)
    moveJointTo(legHL, 0.5); moveJointTo(legKL, 0.5)   → return leg to neutral mid-travel

# Phase 2: now that middles are calibrated and centered, front/rear can move safely
for each leg in [RL, FL, RR, FR]:
    moveJointTo(legHY, 0.5)                            → un-splay yaw from §2's parked stop
    standardCalibrate(leg)
    moveJointTo(legHL, 0.5); moveJointTo(legKL, 0.5)   → return leg to neutral mid-travel
```

The phase split matters: the front/rear legs can't safely move from their splayed positions until the middles have been calibrated and are holding at a known centered geometry. Within phase 2 the front/rear order is flexible.

Step 7 is just a regular `calibrateJoint(direction=extend)` — the hip motor easily handles the weight of half the robot under extension, so no special body-load handling is needed. Hip max is recorded once, at step 7, like any other end-stop.

## 5. Pose-transition primitives

`moveJointTo(name, normPos)`: a P-controlled drive to a target normalized position via `LinearActuator::setTarget(val)` (`actuator_manager.h` ~L99–104). Wait for settle (`|getPos() - target| < 0.02` for 250 ms) or time out.

`returnLegToSquat(leg)`: `moveJointTo(legHL, 0.0)` (full retract) → `moveJointTo(legKL, 1.0)` (full extend) → `moveJointTo(legHY, 0.5)` (yaw center). Used between legs in §3.

`parkInSquat()` / `parkInNeutral()`: final-pose primitives called after the full sweep completes successfully.

## 6. Failure handling (3f)

The firmware never emits JSON. When a joint fails during `calibrateAll`, the firmware streams a single line on the leader's USB serial in the same shape as every other firmware error (Task 2 §2f):

```
ERR <joint> <errorcode>
```

For example: `ERR RLHL hall_drift`. The reason codes come from the shared vocabulary in Task 4. As soon as an `ERR` fires during a cal sweep, `calibrateAll` halts and calls `holdAll()` (`actuator_manager.h` ~L304–314) to park every actuator.

The SDK side handles operator-facing presentation: `_reader_loop` (`firmware/krabby_mcu.py` ~L135–141) captures `ERR ` lines and `KrabbyMCUSDK.calibrate_all()` translates them via Task 4's `explain_failures()` into human-readable fix instructions. The firmware contract is just `ERR` lines on the wire and `GET calibration` for post-hoc inspection — nothing more.

## 7. SDK and CLI (3h)

- `KrabbyMCUSDK.calibrate_all()` — sends a single `CALL` command, watches the existing joint telemetry stream and any `ERR <joint> <code>` lines that fire during the sweep, and (once the firmware is idle again) issues `GET calibration` / `GET_LEFT calibration` / `GET_RIGHT calibration` to read back the final per-joint cal data. Supports a cancel command that triggers `holdAll()`.
- `krabby-firmware calibrate-all` — operator-facing wrapper. Prints any `ERR <joint> <code>` lines as they come over serial (translated via Task 4's `explain_failures()`); on completion, prints the per-joint cal tuples from `GET calibration`. Nothing else.

## 8. Bench-without-chassis fallback

If the V0.2 chassis isn't assembled yet, **Task 3 can still be exercised** on the bench setup from Task 0. The auto-squat sequence and the standard per-leg cal sequence are driven to mechanical end-stops — they don't need gravity or body weight. Step 7 (hip extend) is the same `calibrateJoint(direction=extend)` whether the leg is on the bench or in the chassis. The cal values may differ slightly on the bench vs. assembled (different mechanical strain on the motor mounts), so plan to re-run `calibrate-all` once the chassis lands. Don't block the rest of the task on chassis availability.

## 9. Safety FYI

Before the first real-robot run:
- E-stop or DC converter power switch within reach.
- Robot on a floor that can take the foot scraping (the auto-squat's "knee extend" step drives the foot outward; if the floor is delicate, put down a mat).
- Operator should hand-verify all 18 joints respond to jog commands (Task 1) before letting `calibrate-all` run unattended.

## 10. Verify
- **Cold start to cal:** power-cycle from a random pose, run `calibrate-all`, confirm auto-squat brings the robot into the deterministic pose and the full sequence completes.
- **Repeat:** run cal twice in a session; cal values within Task 2's drift tolerance.
- **Recovery:** trigger a deliberate failure mid-cal (unplug one motor's pot lead between auto-squat and per-leg sweep); confirm the firmware emits the expected `ERR <joint> <errorcode>` line, `calibrate-all` halts, motors park, and the SDK surfaces the right fix-instruction via Task 4's `explain_failures()`.

## 11. Deliverable checklist
- [ ] `calibrateAll()` implemented as auto-squat → per-leg sweep → finalize.
- [ ] Auto-squat pre-cal captures hip-min, knee-max, yaw-min, yaw-max for all 6 legs as a side effect of reaching the deterministic pose.
- [ ] Standard 7-step per-leg cal sequence (hip retract → knee retract → knee extend → yaw left → yaw right → yaw center → hip extend) implemented and composed of Task 2's directional `calibrateJoint` calls.
- [ ] Phase ordering: middles (ML, MR) first, then front/rear (RL, FL, RR, FR) after each un-splays its yaw.
- [ ] All 18 joints' cal data saved via Task 2's `JointCal` (in the shared `EepromLayout` struct from Task 1).
- [ ] On failure: firmware emits `ERR <joint> <errorcode>` on serial (no JSON), motors parked, SDK surfaces the fix-instruction.
- [ ] `KrabbyMCUSDK.calibrate_all()` + `krabby-firmware calibrate-all` end-to-end flow tested.
- [ ] Bench-without-chassis fallback documented; the full standard sequence runs on the bench and re-runs on the chassis.
