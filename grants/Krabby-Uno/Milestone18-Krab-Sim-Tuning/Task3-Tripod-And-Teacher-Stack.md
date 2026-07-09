# Task 3 — Tripod Regularity and Carrying the Gait Through the Teacher Stack

**Budget:** 1 week, part-time (~10–12 hrs hands-on). Teacher resumes are cheap (the bridge/2b1 pattern is 100-iter resumes at LR 3e-5); the long pole is play-and-score cycles, which the Task 1 harness makes fast.
**Prerequisites:** Task 2's flat-walk-V2 checkpoint and final flat reward set.

## Narrative
Two jobs this week. First, decide the tripod question with data: if the Task 2 gait diagrams already show tripod (or another regular hexapod phasing) emerging from the air-time/stride/smoothness terms alone, no explicit contact-schedule term is added — per the remove-what-doesn't-earn-its-place rule. If the schedule is still irregular (stable but degenerate: 4–6 feet down shuffles, inconsistent phasing between speed levels), implement a contact-schedule term: reward alternating 3-foot stance sets using the `.*_Footpad` contact states (leg order is already pinned by `_CRAB_TIBIA_JOINT_NAMES` / `_CRAB_FOOT_BODY_NAMES` in `parkour_mdp_cfg.py`), or penalize stance counts outside {3, 4} during commanded motion. The existing partial machinery — `penalty_excess_feet_contact_forward` (−0.20) and the zeroed `reward_stance_support_feet_when_forward` — constrains contact *counts*, not *phasing*; the new term (if needed) is the phasing piece.

Second, carry the fixed gait up the curriculum so it survives contact with obstacle terrain. Following the repo's own stage-transition discipline (README §2 cheat sheet: change rewards OR terrain, not both at once), this is a new bridge-style pass: a `2b3` mode (new reward class extending the 2b2 stack, with the Task 2 air-time/stride/smoothness terms merged in and the Task 1-implicated speed-pressure terms adjusted), resumed from the flat-walk-V2 checkpoint, on the 2a easy-mixed terrain first, then the 2b2 50/50 terrain. The 2b2 lift stack (`reward_foot_clearance` 2.0, `penalty_swing_min_clearance` −0.4, `reward_swing_vertical_vel` 0.8, `reward_recover_from_stall` 0.2) carries over; the open question this week answers with the harness is whether the lift terms and the new gait terms cooperate or fight — if obstacle clearance regresses while gait improves (or vice versa), retune the relative weights and document the trade.

Selection follows the 2b2 sweet-spot practice (README §4.2b): pick the checkpoint by play + metrics, do not promote the last saved iteration.

## Week plan
- **Session 1 (~2 h):** Score Task 2's checkpoint specifically for contact-schedule regularity across the commanded speed range; make the tripod-term go/no-go call; document either way.
- **Session 2 (~3 h):** If go: implement the phasing term in `mdp/`, add to flat-walk-V2, short retrain, score. If no-go: skip to the `2b3` class construction early.
- **Session 3 (~3 h):** Build `CrabHexStage2B3RewardsCfg` + mode wiring (2b2 stack + Task 2 gait terms + adjusted speed-pressure); launch the easy-mixed-terrain resume from flat-walk-V2; score.
- **Session 4 (~3 h):** Launch the 50/50 2b2-terrain run; score gait metrics AND obstacle metrics (`reward_obstacle_clearance`, `crab_failure`, `mean_episode_length` per README §4.0 guidance); pick the sweet-spot checkpoint; commit to `runs/` with README.

## Acceptance criteria
- Tripod go/no-go decision documented with the Task 2 gait diagrams as evidence.
- If implemented: phasing term committed in `mdp/`, gait diagrams show recognizable tripod (or documented alternative) regularity across the commanded speed range; before/after diagrams committed. If not implemented: emergent regularity demonstrated with the same diagrams.
- New `2b3` reward class + mode committed following the subclass pattern; bundled 2b2 config/checkpoints untouched.
- Teacher retrained through easy-mixed and 2b2 50/50 terrain; on the 2b2-mixed eval set, Task 2's gait metrics hold (no tippy-tap regression on obstacle terrain) AND obstacle capability is within the `6300` baseline ballpark (clearance, failure rate, episode length) — or the gait-vs-lift trade is explicitly documented with the retune record.
- Sweet-spot checkpoint selected by play + metrics (not last iter), committed to `runs/` with a README recording provenance, config, and selection rationale.

## Cut line / out of scope
- No command-model changes (lateral/yaw remains a later milestone item — see note below), no payload/terrain-curriculum expansion beyond the existing 2b2 mix, no student distillation yet.
- If the lift-vs-gait retune isn't converging by session 4: commit the best easy-mixed-terrain checkpoint, document the 50/50 tension quantitatively, and let Task 4 start from easy-mixed — a good gait on easy terrain beats a contested gait on 2b2 terrain for this milestone's purpose.

**Note on omnidirectional motion:** the M18 overview's omnidirectional command work (vy/ωz sampling, retiring `penalty_lin_vel_y`, replacing heading P-control) does not fit the four part-time weeks alongside the gait-quality work and is deliberately excluded from these task files. It should be re-scoped as its own follow-on task or milestone; the command-model audit rows from Task 1 are its starting input.
