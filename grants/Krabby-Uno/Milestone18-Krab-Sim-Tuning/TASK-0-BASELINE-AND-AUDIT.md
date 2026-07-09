# Task 0 - Baseline eval harness and reward audit

**Time estimate: ~1.5 dev days (range 1-2).** Sub-table:

| Days | Sub-task |
|------|----------|
| 0.5 | Metrics logger wrapping the play loop (contact/joint/base/command logging; teacher and student play paths) |
| 0.5 | Derived metrics + tripod score + gait-diagram render; cross-check air time against the sensor buffers |
| 0.5 | Run the eval set, commit baselines; audit table + suspect verdicts + diagnosis |

Goal: Prerequisite-scale measurement work before any training change. The reward terms already exist; this task adds no new rewards and trains nothing. It produces the one new piece of code the milestone runs on (a metrics logger wrapped around the existing play loop) and two documents (baseline metric reports, reward/config audit). Every later task is scored against these numbers, so this lands first. With AI assistance this is days, not a week.

Outputs
- Eval harness script in `crab_hexapod_task/scripts/` that plays a checkpoint over a fixed command schedule and writes per-episode metrics files plus a gait-diagram PNG.
- Fixed, versioned eval scenario set (tasks, modes, seeds, episode counts, command schedules).
- Baseline metric reports for teacher `6300` and student `9800`.
- Audit table and diagnosis: every reward term and command range mapped to measured behavior.

Acceptance Criteria
- **0a** - Eval harness committed to `crab_hexapod_task/scripts/`; takes checkpoint + task/mode + seeds; emits per-episode metrics files (including the scalar tripod-phasing score) and a gait-diagram PNG as a debug view; invocation documented in the repo README.
- **0b** - Versioned eval scenario set committed covering `Flat-Walk-v0` and 2b2-mixed terrain.
- **0c** - Baseline metric reports for teacher `6300` and student `9800` committed; these are the before-side of every later comparison.
- **0d** - Audit table committed covering all reward stage classes in `parkour_mdp_cfg.py` and the command/DR setup in `crab_hex_env_cfg.py`: term, weight per stage, intended effect, measured effect, pathology attribution.
- **0e** - Diagnosis committed: each observed gait pathology mapped to a config cause (or marked cause-unknown), including verdicts on the three flagged suspects below.

---

**NOTE**: All paths, attribute names, and config names in this document are guidance from reading the current repo state; verify against the code and document what actually works.

---

## 1. What the harness is

A play script with logging, not a new eval framework. `crab_hexapod_task/scripts/demo_crab_hex_student.py` already does everything except the logging: builds the env (`parse_env_cfg` + `ParkourRslRlVecEnvWrapper`), loads a checkpoint through `make_on_policy_runner`, and steps the policy in a loop. The harness is that same loop with the gamepad code removed and three additions:

1. **Fixed command schedule instead of random resampling.** The demo script already shows the mechanism: override the `base_velocity` command term's `compute` so the env cannot resample, then set commands from a schedule (e.g. hold 0.4 m/s forward for 10 s, then 0.6, then 0.8). This is also how the off-axis probes for suspect 2 are injected (commanded vy or yaw the policy was never trained on).
2. **Per-step logging into buffers.** Every step, read the signals in Section 2 and append to arrays. At episode end, dump raw arrays (CSV or npz) plus a computed-metrics JSON per episode.
3. **Teacher support.** The demo script is student-only (`DistillationWithExtractor` + depth encoder). The harness needs the teacher play path too (`6300` on `Teacher-v0` 2b2 mode); the teacher inference calls are in the existing `scripts/rsl_rl/play.py` flow (README §4.3).

Run with `--headless`, a handful of envs (even 1 is fine; more just gives more episodes per launch), fixed seeds.

## 2. Where each signal comes from

Everything is already maintained by the sim; the harness only reads it.

| Signal | Source (per step) |
|---|---|
| Foot contact state (6 feet) | `scene["contact_forces"]` sensor, `net_forces_w` norm > `force_threshold` (1.0 N, same threshold the sensor itself uses). Sensor covers all robot bodies; select the footpad rows by body name (`FL/FR/ML/MR/RL/RR_Footpad`). |
| Air time per foot | Same sensor: `track_air_time=True` already populates `current_air_time` / `last_air_time` / `last_contact_time`. Cross-check the harness's own schedule-derived air time against these buffers once. |
| Footpad world positions | `scene["robot"].data.body_pos_w`, footpad body indices. Used for stride length and clearance. |
| Joint torque / velocity | `scene["robot"].data.applied_torque`, `.joint_vel`, per DOF. |
| Base velocity (actual) | `scene["robot"].data.root_lin_vel_b`, `.root_ang_vel_b`. |
| Base velocity (commanded) | `command_manager.get_term("base_velocity").vel_command_b`. |
| Base orientation | Robot root quaternion → roll/pitch. |
| Actions | The `actions` tensor already in the play loop. |

## 3. Derived metrics (definitions)

Computed per episode from the logged arrays. Definitions are pinned here so before/after comparisons in Tasks 1-3 are apples-to-apples:

- **Contact schedule:** boolean contact per foot per step (the Section 2 threshold test).
- **Tripod phasing (scalar, gateable):** the two tripod sets are A = {FL, MR, RL} and B = {FR, ML, RR}. Computed over the steady-state portion of each command hold:
  - *Duty factor* per foot: fraction of time in stance.
  - *Within-set coherence* per set: fraction of steps where all 3 feet of the set share the same contact state.
  - *Anti-phase score* between sets: negative correlation between set A's and set B's stance counts (a proper tripod alternates the sets; tippy-tap has both sets down almost always, correlation near 0 with duty factors near 1).
  - *Tripod score*: one scalar in [0, 1] combining the above (e.g. mean within-set coherence x max(0, -corr(A, B))). This is the number Task 1's AC and the Task 4 stage gates threshold on; a human never has to look at a plot to score a run.
- **Air time:** duration of each contiguous no-contact interval, per foot. Reported as a distribution (histogram + percentiles), not a mean, because the tippy-tap signature is the mass of the distribution sitting at or just above 50 ms.
- **Stride length:** horizontal (xy) displacement of a footpad between consecutive touchdowns; reported as magnitude and as the component along the commanded velocity direction (so lateral steps in Task 2 are scored the same way forward steps are). Distribution per foot.
- **Swing clearance:** footpad height above terrain during each swing interval; report max-over-swing and min-over-swing. On flat ground, terrain height is a constant; on 2b2-mixed, use the height-scanner value or terrain lookup under the foot.
- **Tracking error:** commanded minus actual base velocity per axis (x, y, yaw), averaged over the steady-state portion of each command-hold window (skip the first ~1 s after a command change).
- **Orientation stability:** roll and pitch RMS over the episode.
- **Action rate:** mean per-step |a_t - a_{t-1}| per DOF, plus a count of sign reversals in commanded joint deltas; this is the smoothness number Task 1's energy retune has to move.

## 4. Gait diagram (debug view, not the gate)

The tripod score in Section 3 is the machine-readable judgment; the diagram is the human debug view rendered from the same contact schedule (a few lines of matplotlib at episode end). One row per foot (6 rows), time on x, a filled bar while the foot is in contact:

- **Healthy tripod:** two alternating groups of 3 bars (FL+MR+RL vs. FR+ML+RR) in a checkerboard, clear white gaps (swing phases) between stance bars.
- **Current tippy-tap:** dense stippling, all six rows nearly solid, gaps too short to see at plot scale.

One PNG per episode next to the metrics files, for diagnosing *why* a tripod score is low (e.g. one lagging leg vs. no phasing at all).

## 5. Eval scenario set

Committed as a config/manifest the harness reads, so later runs execute the identical set:

| Checkpoint | Task/mode | Terrain | Command schedule | Episodes |
|---|---|---|---|---|
| Student `9800` | `Student-v0` | 2b2-mixed | forward holds at low/mid/high of trained range | ≥10, fixed seeds |
| Student `9800` | flat variant | flat | same forward holds | ≥10, fixed seeds |
| Student `9800` | flat variant | flat | off-axis probes: commanded vy, commanded yaw (suspect 2) | ≥5 per probe |
| Teacher `6300` | `Teacher-v0` 2b2 | 2b2-mixed | forward holds | ≥10, fixed seeds |

## 6. Audit and suspects

With baselines in hand, fill one row per reward term in `CrabHexFlatWalkRewardsCfg`, `CrabHexTeacherBridgeRewardsCfg`, `CrabHexStage2BPhase1RewardsCfg`, `CrabHexStage2BPhase2RewardsCfg`, plus the command/DR setup in `crab_hex_env_cfg.py`: term, weight per stage, intended effect, measured effect, pathology attribution. Most rows are reading config against the Section 3 numbers; the three below need explicit verdicts:

1. **Air-time threshold 0.05 s** (`reward_feet_air_time_positive` w=0.40 flat; `reward_feet_air_time_on_flat` w=0.5 bridge, zeroed in 2b2). Verdict from the measured air-time distribution: if the mass sits at or just above 50 ms, the term is confirmed as rewarding micro-steps.
2. **Forward-only command model** (`lin_vel_x` only; heading locked at (0, 0) with stiffness 1.5; `penalty_lin_vel_y` w=-3.0). Verdict from the off-axis probe episodes: does the policy ignore or fight commanded vy/yaw.
3. **Speed-forcing terms** (`penalty_low_forward_speed_when_commanded`, `reward_forward_progress_along_command`). Verdict from comparing gait metrics across the low/mid/high command holds: does micro-step frequency rise at the top of the commanded range.

The diagnosis document closes the task: each observed pathology → the config cause the data supports, or cause-unknown, written in the style of the crab README's per-stage appendices.
