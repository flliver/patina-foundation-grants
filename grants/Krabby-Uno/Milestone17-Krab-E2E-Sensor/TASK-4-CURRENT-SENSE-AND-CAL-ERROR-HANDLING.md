# Task 4 — Current-sense validation (final phase of `calibrate-all`)

**Time estimate: ~2 dev days (range 1.5–3).** Sub-table:

| Days | Sub-task |
|------|----------|
| 0.5 | Calibration reason-code vocabulary as a single source of truth (firmware + Python) |
| 0.5 | Neutral-pose transition step + `validateCurrentSense()` per-leg lift sequence (chassis-dependent) |
| 0.5 | SDK reason-code → user-instruction translation; CLI prints actionable directions on failure |
| 0.5 | End-to-end run on real chassis (or documented deferred run if chassis isn't ready) |

Goal: **Current-sense validation is the *final phase* of `calibrate-all`** — it runs **after** Task 3's per-joint cals complete on all 18 joints, because the validation needs the robot to drive itself to a known neutral pose (every joint at position 0.5), and that only works once every joint is individually calibrated and accepts position targets. So the flow is:

```
calibrate-all
  ├── (Task 3) per-joint cal sequence for all 18 joints (middles first, then front/rear)
  ├── neutral-pose transition  ←  every joint → position 0.5 via send_command_joints
  └── (Task 4) per-leg body-lift current-sense validation × 6
```

Without the neutral-pose step the validation can't position itself; without the per-joint cal the neutral-pose command can't be issued (calibration_state would block any `T` target — see Task 2 §6.5 "Position commands on PARTIALLY_CALIBRATED joints are rejected"). So Task 4 is structurally downstream of Task 3 and explicitly chained in the same operator command.

Wire format for error reporting is **`ERR <joint> <errorcode>` per [Task 1 §5](TASK-1-V03-BRINGUP-AND-COMMS.md)** — this task does not redefine it. Task 4's scope is (a) the calibration-specific reason-code vocabulary all of Tasks 2/3/4 emit through that channel, and (b) the SDK-side translation that turns codes into operator-facing fix instructions.

Outputs
- Error codes Tasks 2/3/4 emit are added to Task 1 §5's canonical vocabulary table (single source of truth). No separate errors module; firmware emits the strings inline.
- A new **neutral-pose** step in `calibrateAll` (between Task 3's per-joint phase and the validation) that issues a `T` target with every joint at position 0.5 and waits for settle.
- `validateCurrentSense()` on the MCU that runs the per-leg body lift × 6 and emits `ERR <joint> <code>` lines (per Task 1 §5) for any joint that fails. Runs **only after** the neutral-pose step lands.
- `KrabbyMCUSDK.explain_failures(...)` that turns reason codes (parsed from the ERR stream by Task 1's `_reader_loop`) into operator-facing fix instructions.
- CLI: `krabby-firmware calibrate-all` runs the full chain by default (per-joint cal → neutral pose → current-sense validation). `--skip-validation` shortcuts past the chassis-required validation step for bench-only runs.

Acceptance Criteria
- **4a** — A fixed calibration reason-code vocabulary is defined once and shared by Tasks 2, 3, and 4. Codes include at minimum: `current_sense_no_signal`, `current_sense_no_spike`, `motor_did_not_move`, `pot_value_invalid`, `hall_no_edges`, `hall_drift`, `not_in_starting_pose`, `not_calibrated`. Codes are strings (not numeric IDs); the wire format that carries them is Task 1 §5's `ERR <joint> <errorcode>` telemetry line. (`pot_hall_disagree` is dropped — Task 2 now treats each motor as having exactly one sensor.)
- **4b** — Neutral-pose transition: after Task 3's per-joint cal completes, `calibrateAll` issues a single `T` target with every joint at position 0.5 and waits for settle (e.g. `|getPos() − 0.5| < 0.02` for 250 ms on every joint, or a documented timeout) before the per-leg lifts start.
- **4c** — `validateCurrentSense()` runs after the neutral-pose step (chained inside `calibrateAll`, or standalone via `validate-current-sense`) and lifts the body-half on each of the 6 legs in turn.
- **4d** — For each lift: unloaded current (hip lifted, leg unloaded) `avgIS` reads near zero (e.g. ≲ 50 raw ADC counts); loaded current (leg pushing body up ~1–2") `avgIS` spikes above a small threshold (e.g. ≳ 100 raw counts) and within a sanity ceiling (e.g. ≲ 800). Thresholds are inline numeric literals in the firmware where the check is done, not named constants — tune on the real robot.
- **4e** — Current sense is **not normalized** in this validation — the absolute raw ADC value is what the model consumes downstream (via Task 6's mapping to `contact_forces`). Spikes under load will vary with pose, so this validation only checks the signal is alive.
- **4f** — `KrabbyMCUSDK.explain_failures(errors)` takes a list of `(joint, code)` tuples (as parsed from `ERR <joint> <code>` telemetry lines by Task 1's `_reader_loop`) and returns a list of human-readable fix instructions:
  - `current_sense_no_signal` → "Check current-sense wiring on <joint> (shunt + analog IS line back to the shield)."
  - `current_sense_no_spike` → "Current sense on <joint> reads but does not spike under load. Verify the motor is actually bearing weight; if so, check that the shunt resistor isn't damaged."
  - `motor_did_not_move` → "Check motor power wiring on <joint> (H-bridge output + power harness)."
  - `pot_value_invalid` → "Check potentiometer wiring on <joint> (3-wire harness: VCC, signal, GND)."
  - `hall_no_edges` → "Check Hall encoder wiring on <joint> (A/B channels and power)."
  - `hall_drift` → "Hall encoder count drifts between repeat sweeps on <joint>. Check signal integrity (cable shielding, ground loops) and the encoder coupling to the motor shaft."
  - `not_in_starting_pose` → "Robot is not in the auto-squat starting pose. Re-run calibration from any state — the cal routine drives there automatically."
  - `not_calibrated` → "<joint> is PARTIALLY_CALIBRATED — a position target was rejected. Jog the joint to either end-stop or run `calibrate-joint <joint>`."
- **4g** — CLI: `krabby-firmware calibrate-all` runs Task 3 + neutral-pose + Task 4 by default. `--skip-validation` shortcuts past validation for bench-only runs. `krabby-firmware validate-current-sense` runs the validation standalone (assumes Task 3 cal already done). On failure, the CLI prints `explain_failures()` output (not just raw codes). To inspect cal state after the fact, the operator uses `krabby-firmware get calibration` / `get-left calibration` / `get-right calibration` (per §4). **Nothing is written to disk and no JSON is generated** — `ERR` lines are the live indication; `GET calibration` is the after-the-fact inspection path.
- **4h** — Tasks 2 and 3 emit error codes as bare string literals matching the canonical vocabulary in Task 1 §5. No shared module to import, no `Enum` class to keep in sync — just well-known strings on the wire. If a task starts emitting a new code, the code is added to Task 1 §5's table and to the SDK's `_FIX_INSTRUCTIONS` dict (§5 of this task) in the same PR.

---

## 1. Why current sense + error handling are in one task

Current sense is the last sensor signal that hasn't been validated end-to-end before the model touches the robot. It's also the only validation step that strictly needs the chassis — the lift maneuver requires body weight. Pairing it with the calibration error-handling system means:

- The reason-code vocabulary is finalized before Task 5 / 6 / M15 lean on it.
- The "explain failures" SDK surface is built once for all calibration-adjacent commands.
- The chassis-blocked piece (the actual lift) lives in the same task as a lot of chassis-independent work (the schema, the translation), so progress isn't stalled while waiting for the chassis.

If the chassis isn't ready yet (see [OVERVIEW FAQ](OVERVIEW.md)), Task 4's chassis-independent work (4a, 4b, 4f, 4g without the live run, 4h) ships first, then the per-leg lift validation lands when the chassis arrives. This is the cleanest place in the milestone to absorb chassis-delivery slip.

## 2. Why current sense ≈ foot touch for the model

The locomotion model's `HardwareObservations.contact_forces` (5 values, `hal/client/data_structures/hardware.py` ~L86, docstring "5 values from environment, normalized to [-0.5, 0.5]") is the foot-ground contact signal. On the trained quadruped that came from a force/torque-style sensor; on the hex we proxy from per-leg current draw — when a leg bears weight, its hip/knee actuators draw more current.

**If the current-sense channel is dead or noisy, the model has no contact signal**, and M15's sim-to-real has nothing to anchor against. Task 6 picks the actual current→contact_forces mapping; Task 4 just confirms the per-actuator signal is alive and reads sanely under known load.

## 3. The neutral-pose transition and per-leg lift sequence

This runs as the final phase of `calibrateAll`, immediately after Task 3's per-joint cal completes on all 18 joints.

**Step 0 (new): neutral-pose transition.** The firmware issues a single `T` command (`send_command_joints`) with every joint targeted at position **0.5** — mid-travel on the calibrated `[0.0, 1.0]` scale. It waits for every joint to settle within tolerance (e.g. `|getPos() − 0.5| < 0.02` for 250 ms) before continuing. This is only possible *after* per-joint cal, because Task 2 §6.5 rejects position targets on `PARTIALLY_CALIBRATED` joints; once Task 3 has run, every joint is `FULLY_CALIBRATED` and will accept the target. After settle, the robot is in its canonical standing pose with all feet on the ground — the starting state every per-leg lift expects.

**Per-leg lifts.** For each leg (rear-left → … → middle-right):

1. **Set baseline.** Drive the leg's hip-pitch to a position that lifts the leg off the ground (use the calibrated lifted end-stop, scaled back by 10% to avoid driving into hardstop). Hold 250 ms for current to settle. Record `unloaded_IS[legHL]`, `unloaded_IS[legKL]`, `unloaded_IS[legHY]` from `LinearActuator.avgIS` (~L46 in `actuator_manager.h`).
2. **Lift the body half.** Drive the knee to extend, raising the body half by ~1–2" on this single leg. Hold 250 ms. Record `loaded_IS[…]`.
3. **Lower.** Drive hip and knee back to neutral (mid-travel ≈ 0.5). Wait for settling.
4. **Evaluate** per joint:
   - `loaded_IS - unloaded_IS` < a small delta (e.g. 20 raw counts) → emit `ERR <joint> current_sense_no_signal`.
   - `loaded_IS` below the spike threshold (e.g. < 100) → emit `ERR <joint> current_sense_no_spike`.
   - `loaded_IS` above the sanity ceiling (e.g. > 800) → emit `ERR <joint> current_sense_no_spike` (anomalously high; probably noise/short, not load).
   - Otherwise pass; SDK aggregates the unloaded/loaded readings for the operator-facing summary.

The "~1–2 inches" body lift is approximate — geometry sets the actual height. The point is enough load to produce a measurable current spike, not a precise height.

## 4. Retrieving the full calibration state — `GET calibration`

There is **no JSON report** anywhere in the firmware-side flow. While a calibration is running, the firmware emits `ERR <joint> <code>` lines as faults occur (per Task 1 §5) and that's it — no summary, no envelope, no `ok` field. The SDK aggregates whatever ERR lines it sees and translates them via `explain_failures` (§5).

If the operator wants to inspect the **full calibration state** after the fact, they issue a new `GET calibration` command. The reply is **one line per joint, tagged like telemetry**, in the same shape as the existing per-joint telemetry segment:

```
> GET calibration
< CAL FLHY HALL -512 511 0
< CAL FLHL POT 200 820 1
< CAL FLKL POT 180 805 0
< CAL FRHY HALL -498 503 0
< CAL FRHL POT 195 815 1
< CAL FRKL POT 178 808 0
```

Each `CAL` line is `CAL <joint_name> <sensor_type> <sensor_min> <sensor_max> <sensorReversed>` — the per-joint tuple from Task 2's `JointCal` struct, in the order the firmware stores it. The host's `_reader_loop` recognizes the `CAL ` prefix the same way it recognizes telemetry / `VER` / `ERR` prefixes, collects the 6 lines per board, and presents them as a dict to the caller.

For follower boards, the primary forwards `GET_LEFT calibration` / `GET_RIGHT calibration` to Serial1/Serial2 (per Task 1 §4) and rewrites each follower's `CAL …` reply line to `CAL_LEFT …` / `CAL_RIGHT …` on USB so the host can attribute it.

The operator-facing CLI exposes this as `krabby-firmware get calibration` (and `get-left calibration` / `get-right calibration`), which pretty-prints the 6 joints from each board. That's the entire reporting path: streaming ERR lines while it runs, `GET calibration` to inspect after — no JSON to assemble, no SDK-side schema to maintain.

## 5. SDK error-code → fix-instruction translation (4f)

A simple Python dict in the SDK, keyed by the error-code strings from Task 1 §5's canonical vocabulary:

```python
_FIX_INSTRUCTIONS = {
    "motor_did_not_move":       "Check motor power wiring on {joint} (H-bridge output + power harness).",
    "motor_jammed":             "Motor on {joint} is drawing current but not moving — mechanical jam, bind, or obstruction. Investigate before re-driving.",
    "pot_value_invalid":        "Check potentiometer wiring on {joint} (3-wire harness: VCC, signal, GND).",
    "hall_no_edges":            "Check Hall encoder wiring on {joint} (A/B channels and power).",
    "hall_drift":               "Hall count drifts between repeat sweeps on {joint}. Check signal integrity and encoder coupling to the motor shaft.",
    "not_calibrated":           "{joint} is PARTIALLY_CALIBRATED — a position target was rejected. Jog the joint to either end-stop or run `calibrate-joint {joint}`.",
    "current_sense_no_signal":  "Check current-sense wiring on {joint} (shunt + analog IS line back to the shield).",
    "current_sense_no_spike":   "Current sense on {joint} reads but does not spike under load. Verify the motor is actually bearing weight; if so, check the shunt resistor.",
}

def explain_failures(errors: list[tuple[str, str]]) -> list[str]:
    """errors is a list of (joint, errorcode) pairs as parsed from `ERR <joint> <code>` lines."""
    out = []
    for joint, code in errors:
        tpl = _FIX_INSTRUCTIONS.get(code, f"Unknown failure on {{joint}}: {code}")
        out.append(tpl.format(joint=joint))
    return out
```

The SDK's `_reader_loop` (Task 1 §5) collects `(joint, code)` tuples from the `ERR <joint> <code>` telemetry stream. The CLI calls `explain_failures(errors)` on whatever ERRs fired during the run and prints the instructions so the operator sees the next action.

## 6. CLI entry points (4g)

- `krabby-firmware calibrate-all` — runs Task 3's per-joint cal, then the neutral-pose step, then `validateCurrentSense()`. Default behavior for the full bring-up.
- `krabby-firmware calibrate-all --skip-validation` — bench-only mode; runs Task 3's per-joint cal + neutral-pose but skips the chassis-required lifts.
- `krabby-firmware validate-current-sense` — runs the validation standalone (assumes Task 3 cal already done).
- `krabby-firmware get calibration` / `get-left calibration` / `get-right calibration` — after-the-fact inspection of the EEPROM cal state (per §4).

The firmware emits **no progress lines**. The only outputs while a command runs are: continuous joint telemetry (always), any `ERR <joint> <code>` lines if something fails, and (for GET) the final reply lines. The CLI just listens. If a long-running operation needs progress display, the SDK polls joint positions via the existing telemetry stream — but that's an SDK concern, not a firmware contract.

## 7. Chassis-without setup

`validateCurrentSense()` is the **one step in M17 that hard-requires the chassis** — you can't fake body load on a bench. The chassis-independent pieces ship first; the actual lift validation runs when the chassis arrives. Until then, `calibrate-all --skip-validation` runs the cal and skips validation cleanly.

## 8. Safety FYI

The per-leg body lift is small and the other five legs hold the robot up. Still:
- E-stop within reach.
- Run on a stable floor, not a stand (the loaded lift needs the leg to actually bear weight against the ground).
- If a lift produces no current spike on **any** leg, stop the validation — likely a systemic wiring or sensor issue.

## 9. Verify
- **Vocab parity:** Tasks 2 and 3 emit error codes via `ERR <joint> <code>` lines (Task 1 §5) end-to-end; SDK parses them; `explain_failures` translates each.
- **Lift-test on chassis:** every leg produces a current spike on at least one of its actuators; failures fire `ERR <joint> <code>` with the right code.
- **`GET calibration` round-trip:** after `calibrate-all` completes, `get calibration` / `get-left calibration` / `get-right calibration` return 6 `CAL <joint> <sensor_type> <min> <max> <reversed>` lines per board, matching what was just written to EEPROM.
- **Operator-facing CLI:** on a forced failure (unplug a current-sense lead before running), the CLI prints the right fix-instruction from the streamed ERR.

## 10. Deliverable checklist
- [ ] All error codes Tasks 2/3/4 emit are listed in Task 1 §5's canonical vocabulary table. No `CalibrationReason` enum, no `CalErr` namespace, no `calibration_errors.py` module — just inline string literals in firmware and Task 1's table.
- [ ] Tasks 2 and 3 emit error codes via Task 1 §5's `ERR` channel — no parallel strings, no JSON, no progress lines.
- [ ] Neutral-pose step (every joint → position 0.5) chained between Task 3's per-joint cal and the validation lifts.
- [ ] `validateCurrentSense()` per-leg lift implemented; failures emit `ERR <joint> <code>`.
- [ ] `KrabbyMCUSDK.explain_failures(errors)` returns user-facing instructions from `(joint, code)` tuples.
- [ ] `GET calibration` (+ `GET_LEFT` / `GET_RIGHT` variants) returns one `CAL <joint> <sensor_type> <min> <max> <reversed>` line per joint, parsed by the SDK.
- [ ] CLI: `calibrate-all` runs the full chain by default; `--skip-validation` for bench mode; `validate-current-sense` standalone; `get calibration` exposes EEPROM state.
- [ ] End-to-end run on real chassis (or deferred to chassis arrival with the rest of M17 unblocked).
