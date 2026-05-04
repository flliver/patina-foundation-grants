# Task 3 – `krabby` PyPI CLI (Install / Update / Run)

## 1. Narrative

Ship a top-level `krabby` PyPI package whose CLI is the user-facing surface for installing, updating, and running Krabby on Ubuntu or a Jetson Orin. The CLI pulls the locomotion image from ECR (Task 2), manages local install state, and starts the container with the right device, GPU, and Bluetooth flags so a paired Pro Controller works end-to-end without the user writing `docker run`. If integration breaks the manual path, the task includes fixing any straightforward integration gaps in `controller/` and `images/locomotion/` (controller is already installed in locomotion image and tested).

## 2. Scope

- **Package:** PyPI name `krabby`; entry point `krabby`. New top-level package in `krabby-research`. Publish workflow on tag (e.g. `krabby-v*`).
- **CLI:**
  - `krabby --install [<imageRef>]`: pulls the image and does host system setup. Specifically: ensure Docker is usable, log in to ECR if needed (default AWS creds, anonymous pull when public), pull the resolved image (default = `mainline-latest` from Task 2), write a udev rule for the Mega 2560 if necessary, ensure the invoking user is in `dialout` if needed, and record installed identity in local state. Idempotent. Replaces the need to also `pip install krabby-firmware` separately for host setup.
  - `krabby --update`: pull the newest image for the configured channel (or pinned ref); update local state.
  - `krabby run`: start the container with documented flags for MCU serial passthrough (`/dev/ttyACM*` or `KRABBY_MCU_PORT`), GPU (`--gpus all` or Jetson equivalent), and Bluetooth/input device passthrough for a host-paired gamepad.
  - `krabby firmware <args>`: super thin wrapper that runs `krabby-firmware <args>` inside a transient container off the installed image, with `/dev/ttyACM*` and the firmware download cache mounted through. So `krabby firmware --show` and `krabby firmware --update` work for users who only ran `pip install krabby` — the flash tooling lives in the image, not on the host.
  - `krabby --version`: prints CLI version and currently installed image digest.
- **Bluetooth + gamepad:** A Pro Controller paired on the host per [`controller/scripts/jetson/CONNECT_PRO_CONTROLLER.md`](https://github.com/flliver/krabby-research/blob/main/controller/scripts/jetson/CONNECT_PRO_CONTROLLER.md) drives the robot through the `krabby`-started container the same way it works in the manual two-process E2E ([`controller/scripts/jetson/E2E_GAMEPAD_KRABBY.md`](https://github.com/flliver/krabby-research/blob/main/controller/scripts/jetson/E2E_GAMEPAD_KRABBY.md)). Fix any device-passthrough or permissions gaps found during integration.
- **Docs:** Update `controller/scripts/jetson/E2E_GAMEPAD_KRABBY.md`, `controller/cli/README.md`, and `images/locomotion/README.md` so the `krabby` CLI is the canonical install/run path; the manual scripts stay as a debug fallback.

## 3. Acceptance Criteria

1. `pip install krabby` on a clean Ubuntu 22.04+ or Jetson Orin venv installs the CLI.
2. `krabby --install` pulls the default image from ECR, performs host udev/`dialout` setup, and records installed identity; `krabby --install <imageRef>` pulls a specific ref.
3. `krabby --update` refreshes the image and updates local state.
4. `krabby firmware --show` and `krabby firmware --update` work via the installed image, with no `pip install krabby-firmware` on the host and no flash tools installed outside the container.
5. `krabby run` starts the locomotion stack with serial, GPU, and gamepad device passthrough wired correctly.
6. A Bluetooth Pro Controller paired per `CONNECT_PRO_CONTROLLER.md` drives at least one joint through the `krabby`-started container.
7. Controller and locomotion docs name `krabby` as the canonical install/run path; any gaps surfaced during integration are fixed in-repo.
8. Local unit tests cover image-ref resolution (default `mainline-latest` vs explicit ref), state file read/write under a temp dir, and the docker command construction for `krabby run` and `krabby firmware <args>` (verifying `--device /dev/ttyACM*`, GPU flags, BT/input passthrough, and the firmware cache mount) using mocked `subprocess`. Tests run in CI alongside the wheel build.

## 4. Time Estimate (~5–7 days)

| Days | Sub-task title |
|------|----------------|
| 1 | Package scaffold (`pyproject.toml`, entry point, publish workflow on tag) |
| 1 | `--install` (Docker check, ECR auth, pull, host udev / dialout, state write) |
| 0.5 | `--update` |
| 0.5 | `firmware <args>` wrapper (transient container with device + cache passthrough) |
| 1 | Run path: device, GPU, BT input flags; verify against the manual E2E |
| 1–2 | Bluetooth gamepad integration debug (likely the lossy step) |
| 0.5 | Doc updates (controller + locomotion READMEs) |
| 0.5–1 | End-to-end on a Jetson with a Pro Controller |

## 5. Dependencies

- Task 2: ECR image with documented tags.
- Existing `controller/scripts/jetson/CONNECT_PRO_CONTROLLER.md` and `E2E_GAMEPAD_KRABBY.md`.
- Docker on host; user in `docker` group.
- Pro Controller (or supported gamepad).

## 6. Deliverables

- `krabby` PyPI package with `--install`, `--update`, `run`, `firmware <args>`, `--version`.
- Publish workflow on tag.
- Updated `controller/scripts/jetson/E2E_GAMEPAD_KRABBY.md`, `controller/cli/README.md`, and `images/locomotion/README.md`.
- Verified Bluetooth gamepad path on a Jetson against the published ECR image.
