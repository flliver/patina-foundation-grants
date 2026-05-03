# Task 4 – End-to-End Hardware-in-the-Loop Deployment

## 1. Narrative

One **test harness** Orin (fixed location, calibration screen in view). On image build/push, a githook triggers a Greengrass deployment of that image to the test harness, then runs an automated test that sends joint commands and verifies the visible target moves ~1–2". Outcome: clear pass/fail for “mainline image works on real hardware”; failures are actionable for debugging or AI-assisted fix. Document how to run the pipeline locally and in CI.

## 2. Scope

- **Test harness:** Single Orin (single thing); visible target so joint motion yields measurable 1–2" change (camera and joint-state check).
- **Trigger:** Githook (e.g. GitHub Actions) on branch/tag.
- **Pipeline:** (1) Image in ECR with expected tag; (2) create Greengrass deployment targeting test harness with that image; (3) wait for deploy (or timeout); (4) run test: send joint movement commands (teleop path and HAL test client); (5) verify 1–2" movement (camera snapshot and joint-state); (6) exit pass/fail and report (e.g. CI logs or artifact).
- **Docs:** Local run (e.g. `./scripts/e2e-deploy-and-test.sh`), CI run, prerequisites (test harness registered, calibration screen, ECR, Greengrass permissions).

## 3. Acceptance Criteria

1. Githook triggers on defined branch/tag and creates Greengrass deployment to test harness with that image.
2. Automated test runs after deploy and sends joint movement commands; produces visible 1–2" change when successful.
3. Verification step (camera and joint-state) checks ~1–2" movement; method documented.
4. How to run locally and in CI documented; prerequisites listed.
5. Pipeline exits pass/fail; on fail, logs/artifacts support debugging.

## 4. Time Estimate (~3–5 days)

| Days | Sub-task title |
|------|----------------|
| 1 | Test harness (single thing), calibration setup; verification method (1–2") |
| 1 | Githook: on branch/tag, call Greengrass API to deploy to test harness |
| 1 | Test script: joint commands after deploy; verification step; pass/fail exit |
| 0.5 | Wire verification (camera and joint-state); document local and CI run |

## 5. Dependencies

- Task 1: Greengrass set up; test harness Orin registered and receiving deployments. Krabby image has HAL and accepts joint commands (teleop path). Physical: calibration screen and robot placement.

## 6. Deliverables

- CI job in `krabby-home/fleet/` (e.g. `.github/workflows/`): deploy to test harness, run test, verify 1–2", pass/fail.
- Docs: local and CI run; prerequisites; pass/fail meaning.
