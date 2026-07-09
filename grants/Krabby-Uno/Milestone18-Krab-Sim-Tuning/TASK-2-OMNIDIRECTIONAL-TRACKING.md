# Task 2 - Omnidirectional command tracking

Goal: A drivable policy: the robot goes where the command vector points, at the commanded speed, anywhere in the (vx, vy, wz) envelope, with the Task 1 gait quality holding throughout. That includes sustained sideways travel at fixed heading (the crab knee walk), which is the prerequisite gait for eventually traversing 30" doorways the robot cannot fit through frontally. The current command model cannot do any of this: `lin_vel_x`-only sampling, heading locked under P-control, lateral velocity penalized at -3.0.

Outputs
- Reworked command model in `crab_hex_env_cfg.py` with a documented (vx, vy, wz) envelope derived from the actuator specs.
- Command-widening training curriculum resumed from the Task 1 checkpoint.
- Omnidirectional eval added to the Task 0 harness (fixed command points plus a direction sweep).
- Retrained teacher checkpoint in `runs/` with README; audit rows updated for every changed or retired term.

Acceptance Criteria
- **2a** - Command model reworked and committed: sampled vy and wz; heading P-control replaced with commanded yaw rate; `penalty_lin_vel_y` retired; forward-axis-privileged terms (`reward_forward_progress_along_command`, `penalty_body_heading_error_l2`, `penalty_backward_along_command`) retired or reworked; envelope ranges documented with the actuator-based justification; audit table updated for every change.
- **2b** - Training curriculum committed: staged command widening from the Task 1 checkpoint (arcs before full lateral, or a documented alternative), with dense intermediate-direction sampling and in-episode command resampling so the policy learns to change gait while moving.
- **2c** - Omnidirectional eval committed to the harness: a fixed table of command points held for a set duration each, plus a direction-sweep scenario, emitting per-point steady-state tracking error, net heading drift, and the Task 0 gait metrics.
- **2d** - Capability gate: the retrained teacher meets the documented per-axis tracking tolerance at every eval point; pure-lateral points hold heading (net yaw drift within tolerance, no turn-then-walk); gait metrics stay within the Task 1 ballpark at every point including the direction sweep (no discontinuity at intermediate angles).
- **2e** - Selected checkpoint committed to `runs/` with README (provenance, config, selection rationale).

---

**NOTE**: Term and config names in this document are from the current repo state; verify before editing.

---

## 1. Curriculum: strafe comes after forward

Resume from the Task 1 checkpoint; do not train omnidirectional from scratch here (Task 4's pipeline decides final stage ordering from what this task learns). Widen the command distribution in stages, each a short resume gated by the harness:

1. **Arcs:** add sampled yaw-rate commands (`track_ang_vel_z_exp` already exists but has only ever seen the P-controller's output) while vy stays near zero. Forward walking bent into curves is the smallest step from the Task 1 gait.
2. **Lateral widening:** widen the vy range progressively (e.g. 25% → 50% → 100% of envelope), sampling all directions in between, not just the axes. If a widening step destabilizes training or breaks the forward gait, halve the step and document.

The envelope limits (max forward/backward, lateral, yaw rate) come from the Task 3 actuator specs through the leg geometry, not from round numbers.

## 2. What the strafe gait is, and what the reward has to change

On this morphology, lateral displacement comes from knee extension/retraction: leading-side legs reach and pull, trailing-side legs push, while forward walking sweeps legs fore-aft. Different joints do the work, but the support structure is expected to stay the same alternating tripod, because the two 3-foot sets (A = FL/MR/RL, B = FR/ML/RR) are left-right symmetric and provide the same stability margin for any travel direction.

Do not hard-require that expectation. The gate is the direction-agnostic Task 0 metrics: air time, stride measured along the commanded direction, smoothness, and contact-schedule regularity. If training converges to a different regular phasing for lateral travel (e.g. a wave pattern), that is acceptable if those metrics hold; document the emergent phasing and, if the A/B tripod score misreads it, commit the variant scoring alongside.

Reward changes are mostly removals, not additions: the Task 1 gait terms are already direction-agnostic, and this task retires the forward-privileged terms (2a). One check to do in the audit: the remaining default-pose and hip-position terms were tuned for forward stance and may fight the lateral stance; retune or note them if the lateral gait metrics stall against them.

## 3. Transition across the arc: one policy, continuous gait family

There is no gait switch to design. The policy is one network conditioned on the command vector; trained on a dense, continuous command distribution it learns a continuous family of gaits, and at intermediate angles the feet trace diagonal strides. Two training requirements make that real rather than assumed, and both are in 2b: intermediate directions are sampled densely, and commands resample mid-episode so the policy practices morphing gait while moving.

The check is the direction-sweep eval: hold speed constant and rotate the commanded direction slowly from +x through +y (and onward), logging tracking error and gait metrics continuously. A policy with a hidden mode seam shows a spike (stumble, phase glitch, tracking dropout) at some angle; a clean policy shows flat curves through the sweep.

## 4. Omnidirectional eval

Two scenarios added to the Task 0 harness manifest, both emitting the standard metrics files:

- **Command points:** a fixed table covering the envelope (directions x speeds x yaw rates, axes and diagonals, forward and backward), each held for a set duration; per-point steady-state tracking error, net heading drift, and gait metrics. Runs are diffed numerically point by point.
- **Direction sweep:** the Section 3 rotation scenario, logged continuously.

Both become permanent gates in the Task 4 pipeline.
