# Task 6 — HAL → model integration and flail test

**Time estimate: ~2 dev days (range 1.5–3).** Sub-table:

| Days | Sub-task |
|------|----------|
| 0.5 | Deploy the default locomotion image on the Orin with `--control-source inference`; smoke-test that the model issues commands over HAL |
| 0.5 | Audit the HAL → model mapper against the calibrated MCU outputs (joint positions, joint velocities, previous action ranges); document |
| 0.5 | Current-sense → `contact_forces` mapping (5 slots, 6 legs) implementation and scaling decision |
| 0.5 | Enumerate Isaac-only observation fields missing on real; capture the flail-test clip + HAL log |

Goal: With Tasks 1–5 complete (boards bring-up'd, joints calibrated, current-sense validated, ZED IMU plumbed), deploy the default locomotion image on the Orin so the parkour model — not the controller — drives the joints, and audit every value the model consumes against the calibrated robot. The success criterion is the robot **flailing under model command with all telemetry live**, exactly the state M15 Task 1 picks up from. Gait quality is M15's bar, not this milestone's.

Outputs
- `krabby run --checkpoint …` end-to-end with the parkour model driving joints over HAL.
- A written audit of the HAL → model mapper showing which values land where, with measured ranges from the calibrated robot.
- A documented current-sense → `contact_forces` mapping (5 slots, 6 legs) with scaling constants chosen against Task 4's loaded/unloaded readings.
- A table of observation fields populated in Isaac but empty on real (the gap M15's randomization will cover).
- A flail-test clip + telemetry log capturing ~30 s of model-driven joint motion, committed alongside the milestone deliverable.

Acceptance Criteria
- **6a** — `krabby run --checkpoint <path>` (per `hal/server/jetson/main.py` `--control-source inference` at ~L91–99) brings up HAL server + inference client + parkour model in one process, with the model issuing joint commands over `inproc://hal_commands` and the loop running at the configured 100 Hz target (`CONTROL_RATE_HZ` ~L40).
- **6b** — Joint positions reaching the model's proprioceptive vector (`compute/parkour/mappers/hardware_to_model.py` `_extract_proprioceptive` offset 12, ~L172–173) are float in `[0.0, 1.0]` per Task 2. Confirmed via a logged sample.
- **6c** — Joint velocities (offset 24, ~L177) are computed from successive joint positions (Python-side, in the HAL server) and their range is documented. (MCU-side velocity is a follow-up; not required for M17.)
- **6d** — Previous action (offset 36, ~L181) tracks the last commanded joint position via the mapper's existing `set_previous_action` (~L43); verified by jogging a joint and observing the value follow.
- **6e** — Current-sense mapped into `contact_forces` (5 slots; hex has 6 legs — see §3 for the mapping decision). Loaded and unloaded raw ranges from Task 4 drive the scaling constants; mapper converts raw `avgIS` to the model's expected `contact_forces` range (`[-0.5, 0.5]` per the `HardwareObservations` docstring at ~L86).
- **6f** — Mapper consumes ZED IMU via Task 5's `base_ang_vel_b` and `base_quat_w` plumbing; the existing roll/pitch extraction (~L131–141) lights up with live data; tilt-the-robot test produces the expected proprioceptive changes.
- **6g** — Observation fields populated by Isaac but empty on real hardware are enumerated in a deliverable table (`privileged_latent`, `delta_yaw`, `delta_next_yaw`, `terrain_type_flag`, `flat_terrain_flag`, `scan_features` until M11/M15 Task 3 provides it). This is the M15 randomization target list.
- **6h** — Flail test: `krabby run --checkpoint <m2-student-weights>` runs for ≥ 30 s with the robot on a stand or otherwise constrained; the robot moves on all 18 joints under model command (no specific gait expected); a clip + a HAL observation log (.mcap from the existing collector path) are captured.

---

## 1. Running the default image

`hal/server/jetson/main.py` is the production entry (~L45). Relevant args (~L51–106):

- `--control-source inference` — selects the parkour inference client as the joint-command source instead of `portal` (WebRTC operator commands).
- `--checkpoint <path>` — required when `inference`; points at an M2 student `.pt`.
- `--robot hex` — selects `KRABBY_HEX_DEFINITION` (18-joint Krabby), the default.

`krabby run` (`krabby/__main__.py` ~L28–33, `krabby/run.py` ~L16–37) wraps `docker run` of the locomotion image and forwards args after `--`:

```bash
krabby run --checkpoint /workspace/checkpoints/m2-student.pt -- --control-source inference --robot hex
```

If the M2 weights don't load cleanly on hex (joint-count or observation-dim mismatch), use the closest known-loadable checkpoint and document the mismatch — M17's success is the data path, not the policy quality (M15 owns retraining against the prismatic action space).

## 2. HAL → model mapper audit (6b–6f)

The mapper lives at `compute/parkour/mappers/hardware_to_model.py`. The proprioceptive layout (~L99–190):

| Offset | Field | Source | M17 verification |
|---|---|---|---|
| 0:3 | base angular velocity × 0.25 | `hw_obs.base_ang_vel_b` | Task 5 (ZED IMU); confirm units rad/s after camera→body rotation |
| 3:5 | IMU roll, pitch (from quat) | `hw_obs.base_quat_w` → `euler_xyz_from_quat` | Task 5; confirm tilt produces expected sign |
| 5 | zero placeholder | — | leave 0 |
| 6 | delta_yaw | `hw_obs.delta_yaw` | **Isaac-only; document missing on real** |
| 7 | delta_next_yaw | `hw_obs.delta_next_yaw` | **Isaac-only; document missing on real** |
| 8 | zero placeholder (vy) | — | leave 0 |
| 9 | vx command | `nav_cmd.vx` | from controller; available |
| 10 | terrain_type_flag | `hw_obs.terrain_type_flag` | Isaac-only; default 1.0 |
| 11 | flat_terrain_flag | `hw_obs.flat_terrain_flag` | Isaac-only; default 0.0 |
| 12:12+n | joint positions (rel default) | `hw_obs.joint_positions[:n]` | **From Tasks 2/3, normalized [0,1]** |
| 24:24+n | joint velocities × 0.05 | `hw_obs.joint_velocities[:n]` | **Computed Python-side in HAL (§4)** |
| 36:36+n | previous action | `hw_obs.previous_action[:n]` | mapper's existing `set_previous_action` |
| 12+3n:… | contact forces | `hw_obs.contact_forces[:k]` | **From current sense via §3 mapping** |

For each row: log the value with the model running, confirm range, and note any scaling the mapper applies (×0.25 on `base_ang_vel`, ×0.05 on joint velocity). Write the audit into `docs/m17-hal-model-audit.md` (new) or append it to the milestone deliverable.

## 3. Current-sense → contact_forces (5 slots, 6 legs) (6e)

The model's `contact_forces` is 5-wide (`hal/client/data_structures/hardware.py` ~L86; the M2 student trained against this 5-slot vector, originally for a quadruped's 4 feet + something). The hex has 6 legs. Pick a mapping and document it.

**Option A — five legs, drop one (recommended for first pass).** Map five of the six legs into the five slots; document which leg is dropped (e.g. drop one of the two middle legs since they're geometrically redundant for forward gait). Simple, no folding.

**Option B — sum-and-clip pairs.** `slot[i] = clip(sum(per_leg_currents_in_group), [-0.5, 0.5])` with groups chosen to give the model a coherent geometric signal.

**Option C — per-leg surrogate.** `slot[i] = scale * (avgIS_hip + avgIS_knee) / 2` per leg, with the middle pair averaged into one slot. Most physically motivated but most parameter-tuned.

Whichever path, use Task 4's loaded vs unloaded current ranges to set the `scale` constant so the model sees values in the trained `[-0.5, 0.5]` range when a foot is on the ground. Document the choice + the scale in `hal/server/jetson/krabby_mcusdk.py` (or wherever the current → contact mapping lands) and label it explicitly as a "first-pass mapping, expected to be refined in M15."

## 4. Joint velocities (6c)

`HardwareObservations.joint_velocities` is required (~L85). The MCU firmware doesn't emit velocity today — `JointTelemetry` carries `pos` but no velocity. For M17, compute on the Python side:

- In `hal/server/jetson/krabby_mcusdk.py` (or where observations are assembled), keep a per-joint last-position + last-timestamp.
- On each observation tick, compute `vel = (pos_t - pos_{t-1}) / (t - t_{t-1})` and write it to `joint_velocities`.
- Smooth lightly (single-pole EMA, α ~0.2) to suppress serial-jitter spikes.

Firmware-side velocity is a follow-up; M17 ships the Python path. Note this in the audit doc.

## 5. Missing-on-real observations (6g)

Enumerate fields that Isaac populates but the real path can't:

| Field | Source in sim | Real-path status |
|---|---|---|
| `delta_yaw` / `delta_next_yaw` | Isaac `parkour_event` | None — leave 0.0 on real |
| `terrain_type_flag` / `flat_terrain_flag` | Isaac terrain config | None on real; default to 1.0 / 0.0 (treat as non-flat) |
| `scan_features` (132) | Isaac `measured_heights` raycaster | None until M11 + M15 Task 3 |
| `privileged_latent` (29) | Isaac (body mass, CoM, friction, gains) | None on real (correctly — that's what the estimator network fills) |
| `contact_forces` (5) | Isaac contact sensor | **Approximate from current sense (§3); first pass** |

This table is the explicit distribution-mismatch list M15's domain randomization needs to cover. Commit it under `docs/m17-isaac-vs-real-gaps.md` or in the milestone deliverable.

## 6. Flail test (6h)

With everything wired:

1. Robot on a stand or otherwise constrained from running away. E-stop within reach.
2. `krabby run --checkpoint /workspace/checkpoints/m2-student.pt -- --control-source inference --robot hex`.
3. Watch for ≥ 30 s. The robot should move on all 18 joints (random-looking motion is fine; the M2 student trained against rotational sim joints produces a poor gait on a real prismatic hex — that's exactly what M15 fixes).
4. Capture:
   - Clip (phone is fine).
   - HAL observation log (use `--data-collector-output-dir`, see `hal/server/jetson/main.py` ~L66, to record an .mcap of observations + commands).
5. Confirm in the log: joint positions are `[0,1]` floats moving; current sense varies; `base_ang_vel_b` / `base_quat_w` change when the robot is tilted; commands flow.

This is the bar M15 Task 1 picks up from — same image, same model, same data path.

## 7. Bench-without-chassis

The flail test works **without** the V0.2 chassis. Mount the boards + motors on the Task 0 base board, plug in the ZED, run the image. The robot won't walk anywhere (no chassis), but the data path validates: joints move under model command, telemetry flows back, ZED IMU updates when the bench is tilted. The actual chassis arrives later; the M15 hand-off doesn't require the chassis being ready for M17 to finish.

## 8. Deliverable checklist
- [ ] `krabby run --checkpoint … -- --control-source inference --robot hex` brings the model up driving joints over HAL.
- [ ] HAL → model mapper audit doc committed; per-row source / range / scaling documented.
- [ ] Current-sense → `contact_forces` mapping chosen (Option A / B / C from §3), implemented, scaling constants tuned against Task 4 ranges.
- [ ] Joint velocities computed Python-side and verified.
- [ ] Task 5's ZED IMU plumbing verified in the mapper (roll/pitch react to tilt).
- [ ] Missing-on-real observation fields table committed.
- [ ] Flail test clip + .mcap log captured (≥ 30 s, all 18 joints moving).
- [ ] `images/locomotion/README.md` (and the krabby CLI README) updated with the inference invocation.
