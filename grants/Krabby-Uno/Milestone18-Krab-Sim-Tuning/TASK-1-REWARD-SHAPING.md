# Task 1 - Reward shaping for gait quality

Goal: Eliminate the tippy-tap micro-stepping pattern by fixing the reward terms Task 0 implicated, one measured change at a time. The end state is a reward stage where every term has a before/after metric justifying it; that config is what the reproducible curriculum (Task 4) trains from.

Outputs
- New reward stage class(es) in `parkour_mdp_cfg.py` extending the existing subclass pattern; bundled stages untouched.
- New reward functions (stride length, contact-schedule phasing if needed) in `mdp/` alongside the existing terms.
- Per-term changelog: each change, its metric delta, reverted attempts with reasons.
- Retrained teacher checkpoint in `runs/` with README (provenance, config, iteration, selection rationale).

Acceptance Criteria
- **1a** - New reward stage class(es) committed following the existing subclass pattern; bundled stages and checkpoints untouched and still reproducible.
- **1b** - Air-time reward reworked with the threshold derived from measured stride timing; measured air-time distribution shifts to real swing phases; before/after distributions committed.
- **1c** - Stride-length term implemented in `mdp/`; measured stride length improves against baseline; before/after metrics committed.
- **1d** - Tripod score (Task 0 §3) improves from baseline to a committed target threshold across the commanded speed range, via an explicit contact-schedule term or demonstrated as emergent; before/after scores and diagrams committed either way.
- **1e** - Energy/smoothness terms retuned; measured torque and action-rate profiles show reduced high-frequency reversals; before/after profiles committed.
- **1f** - Tippy-tap eliminated on the Task 0 eval set: gait diagram shows distinct swing phases, air-time mass well above the old 50 ms threshold.
- **1g** - Forward capability at or above the `6300` baseline, or the speed trade-off documented with the gait metrics that justify it.
- **1h** - Per-term changelog committed; teacher retrained (resumed from 2b2 `6300`) and committed to `runs/` with README.

---

**NOTE**: The change list below follows the Task 0 verdicts; if a suspect is refuted, skip its fix and note it in the changelog.

---

## 1. Where the work happens

Iterate on the flat-walk stack (`Isaac-Crab-Hex-Flat-Walk-v0`) where reward changes show up in the fewest iterations, then carry the final term set up the teacher stack (2b2-mixed terrain) as a new stage class resumed from `6300`, per the repo's stage-transition discipline (change rewards or terrain, not both at once). Select checkpoints by play + metrics (README §4.2b), not the last saved iteration. Short resumes (100-2000 iters) are enough to see gait changes; from-scratch runs are not needed here.

## 2. Changes, in priority order

1. **Air-time.** Raise `threshold` from 0.05 to a real swing-phase target derived from Task 0 stride timing. Shape so one proper swing beats several micro-steps (capped-linear above threshold rather than binary). Decide with data whether the fixed term returns to the obstacle stack (2b2 currently zeroes it) or stays flat-walk-only.
2. **Stride length.** New reward on per-step footpad displacement at touchdown so fewer/longer steps beat shuffling.
3. **Contact-schedule regularity.** Only if air-time/stride do not produce regular phasing: reward alternating 3-foot stance sets from `.*_Footpad` contact states (leg order pinned by `_CRAB_TIBIA_JOINT_NAMES` / `_CRAB_FOOT_BODY_NAMES`), or penalize stance counts outside {3, 4} during commanded motion. Existing terms constrain contact counts, not phasing.
4. **Clearance retune.** The 2b2 lift stack (`reward_foot_clearance` 2.0, `penalty_swing_min_clearance` -0.4, `reward_swing_vertical_vel` 0.8) was tuned for obstacle lifting; retune so lift and gait terms cooperate. If obstacle clearance regresses while gait improves (or vice versa), retune the relative weights and document the trade.
5. **Energy/smoothness.** Current weights do not suppress reversals (`reward_torques` -1e-5, `reward_action_rate` -0.1, `reward_dof_acc` -2.5e-7, `reward_delta_torques` -1e-7). Sweep `reward_action_rate` and `reward_delta_torques` upward until the reversal profile drops without killing forward motion.
6. **Speed pressure (only if Task 0 implicates it).** Reduce `penalty_low_forward_speed_when_commanded` / forward-progress weights or widen the low-speed deadband, then measure.

## 3. Discipline

One change per training run; score every run with the Task 0 harness against the baseline; revert changes that do not move their target metric. No student distillation in this task; the student is distilled once in Task 4 after the full training regimen stabilizes.
