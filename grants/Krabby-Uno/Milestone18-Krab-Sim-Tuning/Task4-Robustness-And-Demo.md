# Task 4 — Robustness, Student Re-Distillation, and the Before/After Gait Demo

**Budget:** 1 week, part-time (~10–12 hrs hands-on). Two long unattended runs this week (robustness retrain, student distillation); the hands-on time is config, scoring, and the demo package.
**Prerequisites:** Task 3's `2b3` teacher checkpoint and the Task 1 harness/baselines.

## Narrative
Three jobs: harden the gait against the physical variation the real robot will impose, re-distill the student once on the stabilized reward set, and produce the milestone deliverable.

**Robustness (gait-relevant DR only).** Extend the existing event machinery — the 2b2 stage already runs moderate push/mass/CoM randomization inherited from the go2 `EventCfg` — with ranges sized to the real robot rather than the go2 defaults: added payload mass and CoM shift chosen around the krab's mass uncertainty (the sim model vs. the ~83 kg physical build), plus ground-friction range. Terrain stays within the existing sub-terrain machinery at 2b2-comparable difficulty — flat, gentle slope, small roughness, low steps. This is *not* the sim-to-real milestone's transfer-gap randomization (motor strength, PD gains, latency, sensor noise, camera pose) and *not* Stage 4 full parkour; if a range destabilizes training, shrink it and document the stable envelope rather than spending the week on curriculum research.

Retrain from the Task 3 checkpoint, verify with the harness: gait metrics hold across the payload distribution and terrain levels, and nominal-mass flat-ground metrics stay within the Task 3 ballpark (robustness must not cost the in-distribution gait).

**Student re-distillation (once).** Per the milestone's distill-after-stabilization rule, this is the single student run of M18: distill from the final robust teacher using the README §4.4 recipe (`Isaac-Crab-Hex-Student-v0`, 256 envs, teacher checkpoint passed via `--checkpoint`), with the student MDP's terrain aligned to the new training mix the same way the current student config is 2b2-aligned. Select by play + harness metrics on the mixed student MDP, per the `9800` selection precedent — not the last log iteration.

**Deliverable.** A before/after package against the current bundled student:
- **Side-by-side video:** student `9800` vs. final M18 student, identical command scripts and seeds — flat forward at low/mid/high commanded speed, loaded (payload event active), and one light-terrain segment. Same camera, same time window.
- **Metric report:** the full Task 1 metric set, baseline vs. final, for both teacher and student — air-time distributions, stride length, gait diagrams, clearance, smoothness profiles, tracking error, orientation stability — plus the robustness curves (gait metrics vs. payload; per-terrain-level metrics).
- **Weights:** final teacher and student committed to `runs/` with READMEs (provenance, configs, selection rationale), explicitly tagged as the starting baseline for the sim-to-real milestone.

## Week plan
- **Session 1 (~3 h):** Payload/CoM/friction ranges into a new event config for the `2b3` mode; terrain-level list finalized; launch the robustness retrain.
- **Session 2 (~2–3 h):** Score the robustness run (payload sweep + per-terrain-level eval with the harness); shrink any destabilizing range and relaunch if needed; align the student MDP terrain config; launch distillation.
- **Session 3 (~2 h):** Score student candidates on the mixed student MDP; select; commit both checkpoints with READMEs.
- **Session 4 (~3 h):** Record the side-by-side video segments (fixed command scripts, fixed seeds); assemble the metric report; write the milestone summary in the README-appendix style.

## Acceptance criteria
- Payload (mass + CoM shift) and friction randomization ranges committed in the event config with the justification tied to the real robot's mass uncertainty; any ranges shrunk for stability documented as the stable envelope.
- Terrain levels committed; Stage 4 geometry explicitly not used.
- Robust teacher holds Task 2/3 gait metrics across the payload distribution and per terrain level; metrics-vs-payload and per-level metric tables committed.
- No in-distribution regression: nominal-mass flat-ground metrics within the Task 3 ballpark; comparison committed.
- Student re-distilled once from the final teacher; selected by play + metrics; committed to `runs/` with README.
- **Milestone deliverable committed:** side-by-side video (student `9800` vs. final M18 student, identical command scripts/seeds, flat/loaded/light-terrain segments) plus the full baseline-vs-final metric report from the Task 1 harness.
- Final teacher and student tagged in their READMEs as the sim-to-real milestone's starting baseline.

## Cut line / out of scope
- No transfer-gap DR (motor strength, latency, sensor noise, camera pose) — sim-to-real milestone.
- No Stage 4 full parkour, no prismatic action-space work, no omnidirectional command work (see Task 3 note).
- If the week runs short: friction randomization and the slope terrain level can drop (payload robustness and the demo package cannot); one distillation run only — if the first distill underperforms, document it and ship the teacher as the tagged baseline with the student gap noted as follow-up.
