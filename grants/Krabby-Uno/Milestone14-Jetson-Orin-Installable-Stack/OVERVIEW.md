# Milestone 14 – Krabby Installable Stack (Overview)

## Grant Overview

Make Krabby easy for a non-technical user. M14 packages the existing firmware, controller, HAL, vision, and locomotion image into a single install path:


- `pip install krabby` to install the image and firmware flashing tool, which enables installing/running the Krabby docker image from ECR onto a Jetson Orin device. The image bundles `krabby-firmware`, `avrdude`, and `arduino-cli` so a single install gives the user both runtime and flashing.
- `krabby firmware --update` (run from the image, or `krabby-firmware --update` installed standalone) to detect plugged-in Krabby MCUs and flash them from a public S3 store containing every published firmware build.
- A short top-level `krabby-research/README.md` that ties them together for a kit owner, so they know which boards to plug in when, and how to run `krabby` / `krabby firmware`.

After M14, a fresh kit goes from boxed parts to a moving robot via one Python install and a few CLI commands. No monorepo checkout, no Arduino IDE, no hand-written `docker run`.

## Why is this Important?

- **Easy install:** Non-technical kit owners reach a working robot without reading the repo.
- **Versioned releases:** Firmware (S3) and image (ECR) both publish on every commit, organized by branch.
- **Self-service updates:** Firmware and image both update with one CLI call.
- **Foundation for M10 fleet:** The same image and same install surface is what M10's `krabby-bootstrap` and Greengrass deployments target.

## Target Hardware

Bench setup for verifying M14:

- 1× Jetson Orin (Seeed J4012 default)
- 3× Arduino Mega 2560 (Krabby motor controllers)
- 6× Krabby H-bridge power boards

Frame, batteries, and legs belong to M12 and are not needed here.

## Tasks

Total: ~1 month part-time (one developer with AI assistance).

| Task | Summary | Doc |
|------|---------|-----|
| **Task 1** | Public S3 firmware bucket; CI publishes per-branch on every commit; `krabby-firmware` PyPI CLI lists/installs builds; firmware grows a `V` serial command so boards can self-report version. ~5 days. | [TASK-1-FIRMWARE-UPDATER-PACKAGE.md](TASK-1-FIRMWARE-UPDATER-PACKAGE.md) |
| **Task 2** | Add a second, production locomotion image alongside the existing dev one — installs from PyPI (no monorepo `COPY`) and bundles `avrdude`/`arduino-cli`/`krabby-firmware` so flashing works from inside the image. CI builds and pushes the production image to ECR per commit; dev image stays out of CI. ~3–4 days. | [TASK-2-LOCOMOTION-IMAGE-PYPI.md](TASK-2-LOCOMOTION-IMAGE-PYPI.md) |
| **Task 3** | `krabby` PyPI CLI with `--install` (image + host udev/dialout setup), `--update`, `run`, and a `firmware` wrapper that flashes via the image; Bluetooth Pro Controller verified end-to-end. ~5–7 days. | [TASK-3-KRABBY-CLI-ECR-INSTALL.md](TASK-3-KRABBY-CLI-ECR-INSTALL.md) |
| **Task 4** | Run the whole flow on a fresh bench setup, then leave the Orin as a permanent watchdog: poll ECR every minute, run a firmware smoke test on each new `mainline-latest`, alarm on failure via email + GitHub Issue. Write `krabby-research/README.md`; fix any doc/install gaps surfaced. ~5–6 days. | [TASK-4-README-FRESH-HARDWARE.md](TASK-4-README-FRESH-HARDWARE.md) |

## Dependencies

- **PyPI publish workflow** in `krabby-research` already produces controller, HAL, and compute wheels on tag pushes ([publish-packages.yml](https://github.com/flliver/krabby-research/actions/workflows/publish-packages.yml)).
- **Firmware sources:** `krabby-research/firmware/` (Arduino sketch, Python SDK, `pyproject.toml` skeleton already named `krabby-firmware`).
- **Locomotion Dockerfile:** `krabby-research/images/locomotion/` (today builds via local `COPY`).
- **AWS account (Krabby Co):** S3 for Task 1, ECR for Task 2, OIDC roles for GitHub Actions.

## Out of Scope

- **Single-cable firmware update across all 3 boards.** The Megas use inter-board serial for runtime data, not for flashing. A bootloader passthrough would mean custom bootloader logic on each board and a flash-vs-run state machine. For this milestone, users plug USB into each Mega in turn (3 connections, 3 commands), then unplug the boards from USB-serial and plug back into the primary via 3-pin JST serial.

## Repos and Artifacts

- **Code:** `krabby-research/` — `firmware/` (Task 1), `images/locomotion/` (Task 2), new `krabby/` package (Task 3), root `README.md` (Task 4).
- **Stores:** new public S3 bucket for firmware (Task 1); new ECR repo for the locomotion image (Task 2).
- **CI:** extend the existing `.github/workflows/publish-packages.yml` or add siblings.
- **Grant docs:** [Milestone14-Jetson-Orin-Installable-Stack](https://github.com/flliver/patina-foundation-grants/tree/main/grants/Krabby-Uno/Milestone14-Jetson-Orin-Installable-Stack).
