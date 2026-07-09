# Task 2 — Kill the Tippy-Tap: Air-Time, Stride, and Smoothness Rework

**Budget:** 1 week, part-time (~10–12 hrs hands-on). Training runs happen unattended between sessions — the flat-walk task trains at 256 envs headless and saves every 100 iters, so each reward iteration is: edit config, launch, come back, run the Task 1 harness.
**Prerequisites:** Task 1 harness, baselines, and diagnosis verdicts.

## Narrative
Fix the micro-stepping pathology on the fastest retrain loop available: the flat-walk stack (`Isaac-Crab-Hex-Flat-Walk-v0`, `CrabHexFlatWalkRewardsCfg`). Flat-walk is where the gait was originally learned (Stage 1) and where a reward change shows its effect in the fewest iterations; carrying the fixed gait up through the teacher stack is Task 3's job.

All changes go in a **new config class** — `CrabHexFlatWalkV2RewardsCfg` (or similar) extending the existing flat-walk class, selected by a new mode/env variant — so the bundled Stage 1 baseline (Appendix C, `model_6000.pt`) stays reproducible. Training resumes from flat `6000` (or from 2b2 `6300` if the diagnosis shows the flat checkpoint's gait is unsalvageable — contractor's call, documented).

**Changes, in priority order (subject to Task 1 verdicts):**
1. **Air-time term rework.** Raise `threshold` from 0.05 to a value consistent with a real swing phase for this platform — derive the target from the Task 1 stride-timing data, not a guess (rule of thumb: threshold at ~50–70% of the swing duration a non-degenerate tripod would need at the commanded speed range). Shape so one proper swing beats several micro-steps (e.g., per-step air-time capped-linear above threshold rather than binary).
2. **Stride-length term.** New reward on per-step footpad displacement at touchdown, so fewer/longer steps dominate high-frequency shuffling. Implemented in `mdp/` next to the existing observation/action code, config-referenced like the other `mdp_rewards` terms.
3. **Smoothness retune.** The existing penalties demonstrably don't suppress reversals at current weights: `reward_torques` −1e-5, `reward_action_rate` −0.1, `reward_dof_acc` −2.5e-7, `reward_delta_torques` −1e-7. Sweep `reward_action_rate` and `reward_delta_torques` upward until the measured action-rate/torque-reversal profile drops without killing forward motion.
4. **Speed-pressure relief (only if the Task 1 verdict implicates it).** If `penalty_low_forward_speed_when_commanded` / forward-progress pressure correlates with micro-stepping, reduce those weights or widen the low-speed deadband in the new class and measure.

**Discipline:** one change per training run; the Task 1 harness scores every run against the baseline; a change that doesn't move its target metric is reverted, not kept. Expect ~4–6 short runs this week (flat-walk resumes at 100–2000 iters each are enough to see gait changes; full 20k-iter from-scratch runs are not needed and don't fit the budget). Teacher-stack retraining and student re-distillation are explicitly deferred.

## Week plan
- **Session 1 (~3 h):** New `CrabHexFlatWalkV2RewardsCfg` + env variant wiring; air-time threshold change #1; launch run 1.
- **Session 2 (~2 h):** Score run 1 with the harness; implement stride-length term in `mdp/`; launch run 2.
- **Session 3 (~2–3 h):** Score run 2; smoothness sweep (2 runs launched across the session/overnight).
- **Session 4 (~3 h):** Score sweep; apply speed-pressure relief if implicated; final combined run; write the per-term changelog with before/after metrics; commit weights to `runs/` with a README per repo convention.

## Acceptance criteria
- New flat-walk reward class + mode committed; bundled Stage 1 config and checkpoint untouched.
- Air-time term reworked with the threshold derived from measured stride timing; before/after air-time distributions committed.
- Stride-length term implemented in `mdp/` and unit-sane (manual check: value increases with step displacement); before/after stride metrics committed.
- Smoothness weights retuned; before/after action-rate and torque-reversal profiles committed.
- **Tippy-tap eliminated on flat ground:** gait diagram shows distinct swing phases; air-time mass well above the old 50 ms threshold; measured against the Task 1 student/flat baseline.
- Forward-locomotion capability at or above the flat-walk baseline, or the speed trade-off explicitly documented with the gait metrics justifying it.
- Per-term changelog committed: each change, its metric delta, and any reverted attempts with reasons.
- Best flat-walk-V2 checkpoint committed to `runs/` with a README (provenance, config, iteration, selection rationale).

## Cut line / out of scope
- No tripod/contact-schedule term yet (Task 3 decides if it's needed once the air-time/stride terms have had their effect).
- No teacher-stack or student retraining, no command-model changes, no terrain changes.
- If the week runs short: smoothness sweep can shrink to a single weight change; the air-time + stride changes and their measured verdicts cannot slip.
