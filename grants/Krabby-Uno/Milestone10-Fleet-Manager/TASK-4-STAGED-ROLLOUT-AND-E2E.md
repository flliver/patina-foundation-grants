# Task 4 – Staged OTA Rollout and E2E Hardware Validate Gate

## 1. Narrative

Deliver the **standard mechanism to push an image to the fleet in stages**, with an automated hardware **validate gate** between stages. When a new image is published to ECR, it rolls out **test harness → canary → fleet**: the fleet manager sets the desired image for a cohort, each device's M14 launcher (`krabby update`) **pulls** it from ECR and reports back, and an automated hardware test must pass before the image is promoted to the next cohort. A bad image is caught on the bench/test krab — and at worst one canary krab — and never reaches the whole fleet at once. The old end-to-end hardware-in-the-loop test (deploy → joint move → verify ~1–2" movement) becomes the **validate gate**, run first on cohort 0 (the test harness).

This replaces Greengrass deployments and `failureHandlingPolicy: ROLLBACK` with a simpler, more honest model: **pull-based update + validate-before-promote**, plus rollback = "set desired image back to the previous one."

## 2. Scope

- **Cohorts (IoT thing groups, from Task 1):**
  - `cohort-test` — the test harness Orin: fixed location, calibration screen in view, visible target so joint motion yields a measurable 1–2" change.
  - `cohort-canary` — one real krab.
  - `cohort-fleet` — the rest.
- **Desired-image mechanism:** set the desired image per cohort (Device Shadow `desired.image`, or DynamoDB + a `cmd` of type `update`). Devices pull from ECR via the M14 launcher and report `reported.image`. Evaluate **AWS IoT Jobs** as a managed alternative that handles rollout rate and abort criteria natively; document the choice.
- **Validate gate (the E2E hardware loop):** after a device reports the new image, run the automated test against it: send joint-movement commands (teleop/HAL path) and verify a visible ~1–2" change (camera snapshot + joint-state); optionally also run the M14 firmware smoke test (`krabby firmware --show` / `--update` consistency). Produce a clear pass/fail with logs/artifacts.
- **Rollout pipeline:** trigger on image publish to ECR (githook/CI on `mainline-latest`) or a manual portal action →
  1. set desired image for `cohort-test`; wait for report; run validate gate;
  2. on pass, promote to `cohort-canary`; wait for report; run validate gate;
  3. on pass, promote to `cohort-fleet`.
  Any failure halts promotion and surfaces logs; **rollback** = set desired image back to the previous tag/digest for the affected cohort.
- **Portal UI:** rollout status per cohort (current vs desired image, last validate result); buttons to start a rollout, promote/hold, and roll back. (Builds on the Task 2 portal.)
- **Docs:** local run (e.g. `./scripts/rollout.sh` and `./scripts/validate-gate.sh`), CI run, prerequisites (test harness registered in `cohort-test`, calibration screen, ECR, IAM, IoT permissions).

## 3. Acceptance Criteria

1. Setting a desired image for `cohort-test` causes the test harness to pull that image (M14 launcher) and report `reported.image`.
2. The validate gate runs automatically after the report: sends joint-movement commands and verifies ~1–2" movement (camera + joint-state); exits pass/fail with logs/artifacts.
3. On a passing gate, the image promotes to `cohort-canary` (one real krab), validates again, then to `cohort-fleet`; on failure, promotion halts and the failure is actionable.
4. Rollback works: setting the desired image back to the previous version returns devices to it.
5. Rollout is triggerable from a published ECR image (githook/CI) and from the portal; rollout status per cohort is visible in the portal.
6. How to run locally and in CI is documented; prerequisites listed; pass/fail meaning defined.

## 4. Time Estimate (~5–7 days)

| Days | Sub-task title |
|------|----------------|
| 1 | Cohorts wired (test/canary/fleet); set-desired-image path; device reports `reported.image` |
| 1.5 | Validate gate: deploy-to-`cohort-test`, joint commands, verify 1–2" (camera + joint-state), pass/fail + artifacts |
| 1.5 | Promotion pipeline: test → canary → fleet with gate between stages; rollback (revert desired image) |
| 1 | Trigger from ECR publish (githook/CI) and portal; rollout status UI |
| 0.5 | Local + CI run docs; prerequisites; pass/fail meaning |

## 5. Dependencies

- **Task 1:** cohorts (thing groups), desired/reported image mechanism, the `krabby agent` that invokes `krabby update`.
- **Task 2:** portal + fleet service (rollout UI and the set-desired/promote API live here).
- **M14:** the launcher — `krabby-launcher`'s `krabby update` — does the actual pull/install; the firmware smoke test can be reused in the gate. Krabby image accepts joint commands (teleop/HAL path).
- **Physical:** test harness Orin in `cohort-test` with calibration screen and target; one real krab for `cohort-canary`.

## 6. Deliverables

- Rollout + validate-gate pipeline in `krabby-home/fleet/` (e.g. `scripts/` + `.github/workflows/`): set desired image per cohort, run gate, promote test → canary → fleet, rollback.
- Rollout status UI in the portal.
- Docs: local and CI run; prerequisites; cohort/promotion model; rollback procedure; pass/fail meaning.
