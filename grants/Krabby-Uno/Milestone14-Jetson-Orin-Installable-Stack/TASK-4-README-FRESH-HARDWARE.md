# Task 4 – Continuous Bench Test + Top-Level README

## 1. Narrative

Set up a permanent bench Orin that continuously validates the M14 stack. A small watchdog service polls ECR every minute for the `mainline-latest` digest. When it sees a new digest, it pulls the image, runs a firmware smoke test against three attached Megas, and on failure emails a configurable address and (optionally) opens a GitHub issue. Also write `krabby-research/README.md` as the canonical first-read for a kit owner: kit list, how to flash, how to start the image, how to drive the robot. Bring-up of the bench Orin is what produces and proves the README's happy path; gaps surfaced during bring-up are fixed in-repo on the same branch.

## 2. Scope

- **Bench Orin (always on):** 1× Jetson Orin, 3× Mega 2560, 6× H-bridges, USB hub, bench power supply, paired Pro Controller for the manual gamepad smoke. Lives on the developer's bench long-term; not the M12 robot.
- **Watchdog service:** Small Python package shipped from `krabby-research/bench/` (or similar) and run as a systemd unit + 60s timer.
  - Polls the ECR `mainline-latest` digest. Persists the last-tested digest under `/var/lib/krabby-bench/state.json`.
  - When the digest changes: `krabby --update`, then run the smoke test.
  - **Smoke test:** `krabby firmware --update mainline` on each of the three Megas, then `krabby firmware --show`, then assert all three report the same VER and that the VER matches the S3 manifest for `mainline/latest` (Task 1). Optional follow-on: `krabby run` for ~30s, verify the HAL server emits telemetry on its TCP/IPC endpoint, stop the container.
  - **Dedup:** do not re-test the same digest; do not re-alarm on the same (digest, failure) pair within a configurable window (default 1 hour).
- **Alerter:**
  - Primary: configurable email. Two modes the user picks at install time — SMTP env vars (`BENCH_SMTP_HOST`/`USER`/`PASSWORD`/`FROM`/`TO`) or AWS SES (using the same AWS account as S3/ECR). Default to SMTP.
  - Secondary (optional, default on): open a GitHub Issue against a configurable repo using a fine-scoped PAT or GitHub App. (GitHub has no native "alarm" product; an auto-opened issue with the failing digest, smoke-test logs, and timestamps is the closest equivalent — subscribers receive email automatically.)
  - Both alert paths share the same payload: image digest, smoke-test step that failed, captured stderr/stdout tail, firmware VER strings observed.
- **Config:** `/etc/krabby-bench/config.toml` (or env). Fields: ECR repo URI, polling interval, smoke-test toggles, alert mode (`email` / `github` / `both`), SMTP/SES creds, GitHub repo + token, dedup window.
- **Bring-up:** From a clean Jetson OS, follow Tasks 1–3 (`pip install krabby` → `krabby --install` → `krabby firmware --update` per board → `krabby run` once to verify), then install the watchdog (one `pip install` + one `systemctl enable --now`), wire the bench, and let it run for a day to confirm normal-path quiet and a forced-failure alarm.
- **`krabby-research/README.md`:** Short kit list (1× Orin, 3× Mega, 6× H-bridges; full robot at M12). Numbered software path: install `krabby` → `krabby --install` → `krabby firmware --show` → `krabby firmware --update` per board → wire serial harness → `krabby run` → browser teleop → local gamepad. Links to `firmware/SETUP.md`, `images/locomotion/README.md`, the M10 fleet portal entry, and `controller/scripts/jetson/`. Adds a short "Continuous bench" subsection pointing at the watchdog package.
- **Doc fixes:** Anything broken in `controller/`, `images/locomotion/`, or `firmware/` that blocks the bring-up or the watchdog gets fixed and committed on the same branch.

## 3. Acceptance Criteria

1. From clean OS to running locomotion image by following only `krabby-research/README.md` and the docs it links.
2. All three Megas show up in `krabby firmware --show` with the same version after the user follows the README.
3. `krabby --install` and `krabby run` start the locomotion stack from ECR.
4. Browser teleop drives at least one joint from the M10 portal entry.
5. A Bluetooth Pro Controller drives at least one joint through the `krabby`-started container.
6. Watchdog runs as a systemd unit + 60s timer on the bench Orin and detects a new `mainline-latest` digest within one polling cycle.
7. On a new digest, the watchdog runs the firmware smoke test (`krabby firmware --update mainline` × 3 boards → `krabby firmware --show` → VER matches the S3 manifest) and persists the result.
8. A forced-failure smoke test (e.g. unplug a Mega) triggers exactly one email alarm and, if configured, exactly one GitHub Issue, each carrying the image digest, failing step, and log tail.
9. Repeat polls of the same digest do not re-test or re-alarm.
10. Local unit tests cover digest polling + dedup, smoke-test step orchestration (with mocked `subprocess`), state file read/write, and both alerters (with mocked SMTP/SES and mocked GitHub API). Tests run in CI for the watchdog package.
11. Any doc/install gaps surfaced during bring-up are fixed in-repo on the same branch.

## 4. Time Estimate (~5–6 days)

| Days | Sub-task title |
|------|----------------|
| 0.5 | Wipe a Jetson; clean OS install; bench wiring |
| 1 | Initial bring-up: Tasks 1–3 path on the Orin; flash all three Megas; verify `krabby run` + telemetry |
| 0.5 | Browser teleop + local gamepad smoke tests |
| 1.5 | Watchdog package: ECR digest poll, smoke-test runner, state + dedup; unit tests |
| 1 | Email + GitHub Issue alerter; config file + env wiring; unit tests |
| 0.5 | systemd unit + timer; install path; verify forced-failure alarm end-to-end |
| 0.5–1 | Write `krabby-research/README.md`; fix any doc/install gaps surfaced |

## 5. Dependencies

- Tasks 1, 2, 3 complete.
- Bench hardware: 1× Jetson Orin, 3× Arduino Mega 2560, 6× Krabby H-bridge boards, USB cables, bench power supply.
- Pro Controller (or supported gamepad).
- M10 fleet portal entry available, or a stub URL the README can update later.
- Outbound SMTP host or AWS SES sending verified for the bench's `From:` address.
- Optional: GitHub fine-scoped PAT or GitHub App with `issues: write` on the target repo.

## 6. Deliverables

- New `krabby-research/README.md`.
- New `krabby-research/bench/` package (or named equivalent) with watchdog, alerter, sample config, systemd unit + timer, and unit tests.
- Doc fixes committed under `controller/`, `images/locomotion/`, `firmware/` as needed.
- Bring-up notes (commit log or short report) showing the full path was executed end-to-end on fresh hardware and that the watchdog caught at least one forced failure.
