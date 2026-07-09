# Task 4 - Reproducible from-scratch training curriculum

Goal: The milestone deliverable. Today the curriculum is a sequence of hand-launched stage resumes with hand-picked checkpoints, documented only in README appendices. Consolidate the Task 1-3 configs into a documented training pipeline that goes from scratch to the final teacher and distilled student, gated by the Task 0 metrics. The upcoming prismatic and sim-to-real DR retrains both rerun this pipeline.

Outputs
- Scripted pipeline (or documented command sequence) that trains teacher from scratch through all stages and distills the student, with pinned seeds and configs.
- Metric gates from the Task 0 harness defining each stage transition (thresholds committed, checked automatically or by a documented scoring step).
- One completed from-scratch run: final teacher and student weights in `runs/` with READMEs, tagged as the sim-to-real training baseline.
- Baseline-vs-final metric report: full Task 0 metric set plus Task 2 tracking grid and Task 3 payload curve, student `9800` vs. the final student.

Acceptance Criteria
- **4a** - Training pipeline committed: from-scratch teacher training through all stages (gait rewards, omnidirectional commands, actuator limits, payload curriculum) plus student distillation, runnable from pinned configs and seeds with no manual checkpoint hand-off between stages.
- **4b** - Stage-transition gates defined as Task 0 harness metrics with committed thresholds; the pipeline documentation states what passing each gate means.
- **4c** - One from-scratch run executed end-to-end; every gate passed; total wall-clock and hardware documented.
- **4d** - Student distilled once from the final teacher (README §4.4 recipe, student MDP terrain aligned to the new training mix); selected by play + harness metrics, not the last log iteration.
- **4e** - Final teacher and student committed to `runs/` with READMEs and tagged as the sim-to-real starting baseline.
- **4f** - **Milestone deliverable:** baseline-vs-final metric report committed comparing student `9800` to the final student across the full Task 0 metric set, the Task 2 tracking grid, and the Task 3 payload curve.

---

**NOTE**: The pipeline shape (single script vs. documented make targets vs. staged commands) should match how the repo already runs training; the requirement is reproducibility, not a specific tool.

---

## 1. Gate metrics

Each stage transition is gated on the Task 0 harness metrics: swing-phase air time, stride length, tripod score, tracking tolerance across the command grid, and the payload curve. All are scalars or scalar tolerances, so a rerun is pass/fail without judgment calls (or a human looking at plots).

## 2. Pipeline contents

- Stage sequence and the config class each stage uses (the Task 1-3 reward/command/actuator configs, consolidated).
- Resume/from-scratch policy per stage, seeds, iteration budgets.
- Gate check after each stage: run the Task 0 harness on the stage output, compare against committed thresholds, stop and flag on failure.
- Distillation as the final stage, with the student selection criterion written down.

## 3. Report

No demo video is required. The comparison artifact is the metric report; anyone who wants to see the gait runs the harness play command on the committed checkpoints, documented in the pipeline README.
