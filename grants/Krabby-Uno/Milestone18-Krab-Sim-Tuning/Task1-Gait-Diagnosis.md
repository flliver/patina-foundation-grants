# Task 1 — Gait Diagnosis and Reward Audit

**Budget:** 1 week, part-time (~10–12 hrs hands-on). No training runs this week — measurement only.
**Prerequisites:** working Isaac Lab env (`env_isaaclab`, `KRABBY_ROOT` set); bundled checkpoints teacher 2b2 `runs/2026-05-26_21-46-37/model_6300.pt` and student `runs/2026-05-26_22-57-01/model_9800.pt` playable per README §4.3.

## Narrative
Before touching any reward term, measure what the current policies actually do and why. The output of this week is (a) an eval harness that every later week reuses, (b) baseline metric reports for teacher `6300` and student `9800`, and (c) a diagnosis document mapping each gait pathology to a config cause.

The harness is a metrics logger built on the existing play path (`scripts/rsl_rl/play.py` one-liners from README §4.3), living in `crab_hexapod_task/scripts/` next to `verify_crab_joint_drive.py` and `verify_crab_contact_physics.py`. It does not need a UI — CSV/JSON per episode plus one rendered gait-diagram PNG is enough. The `.*_Footpad` contact sensors already run with `track_air_time=True` and `force_threshold=1.0` (`crab_hex_scene_cfg.py`), so contact/air-time data is available from the sensor buffers; stride length and clearance come from footpad body positions at touchdown/liftoff.

**Metrics per episode:** per-foot contact schedule (6 footpads, boolean over time), per-foot air time and stride length distributions, swing-phase foot clearance (max and min-over-swing), joint torque/velocity profiles per DOF, commanded vs. actual base velocity (x, y, yaw) with steady-state tracking error, base orientation (roll/pitch RMS), action rate. Plus one gait diagram (feet × time contact plot) per episode.

**Audit:** every reward term in `CrabHexFlatWalkRewardsCfg`, `CrabHexTeacherBridgeRewardsCfg`, `CrabHexStage2BPhase1RewardsCfg`, and `CrabHexStage2BPhase2RewardsCfg` (`config/crab_hex/agents/parkour_mdp_cfg.py`), and the command/DR setup in `crab_hex_env_cfg.py`, gets a row: term, weight per stage, intended effect, measured effect, pathology attribution.

**Suspects to confirm or refute with data:**
1. **Air-time threshold 0.05 s** — `reward_feet_air_time_positive` (flat-walk, w=0.40) and `reward_feet_air_time_on_flat` (bridge, w=0.5; zeroed in 2b2). Compare the measured air-time distribution against 50 ms: if the mass of the distribution sits at or just above the threshold, the term is confirmed as rewarding micro-steps.
2. **Forward-only command model** — `lin_vel_x` (0.30–0.65 flat / 0.45–0.85 teacher), `heading` locked (0.0, 0.0) with stiffness 1.5, `penalty_lin_vel_y` w=−3.0. Confirm measured behavior: inject nonzero vy/ωz commands in play and record that the policy ignores or fights them.
3. **Speed-forcing terms as tippy-tap contributors** — `penalty_low_forward_speed_when_commanded` (−3.0 bridge / −0.8 2b2) and `reward_forward_progress_along_command`. Check whether the micro-step frequency correlates with these terms' pressure (e.g., gait degrades further at the top of the commanded speed range).

## Week plan
- **Session 1 (~3 h):** Metrics logger skeleton wrapping the play loop; log contact states, joint states, base states, commands to per-episode files. Verify on a short student `9800` run.
- **Session 2 (~3 h):** Derived metrics (air time, stride, clearance, tracking error) + gait-diagram rendering. Sanity-check air-time numbers against the contact sensor's own `track_air_time` output.
- **Session 3 (~2–3 h):** Run the fixed eval set: student `9800` on `Student-v0` (2b2-mixed) and flat; teacher `6300` on `Teacher-v0` 2b2 mode. ≥10 episodes per config, fixed seeds. Commit baseline reports.
- **Session 4 (~2–3 h):** Audit table + diagnosis doc; confirm/refute the three suspects with the measured evidence. Commit in README-appendix style.

## Acceptance criteria
- Eval harness committed to `crab_hexapod_task/scripts/`; takes a checkpoint + task/mode + seed set; emits per-episode metrics files and a gait-diagram PNG; documented invocation in a short README section.
- Fixed, versioned eval scenario set defined (tasks, modes, seeds, episode counts) and committed.
- Baseline metric reports for teacher `6300` and student `9800` committed — the before-side of every later comparison.
- Audit table covering all four reward stage classes and the command/DR config committed.
- Suspects 1–3 each confirmed or refuted with attached metric evidence.
- Diagnosis document committed: every observed pathology → config cause, or explicitly cause-unknown.

## Cut line / out of scope
- No reward changes, no retraining, no config edits beyond the harness itself.
- If time runs short: the teacher-`6300`-on-obstacle-terrain baseline can slip to Task 3's first session; the flat-ground student baseline and the three suspect verdicts cannot slip — Task 2 depends on them.
