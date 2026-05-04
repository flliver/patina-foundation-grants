# Task 2 – Locomotion Image from PyPI to ECR

## 1. Narrative

The locomotion image at `krabby-research/images/locomotion/` today builds by `COPY`ing local source from the monorepo. That image stays as-is for development. This task adds a second, production image alongside it that installs the already-published Krabby wheels (`krabby-hal-client`, `krabby-hal-server-jetson`, `compute-parkour`, etc.) from PyPI with pinned versions, and bundles the firmware flash tooling (`avrdude`, `arduino-cli`, `krabby-firmware`) so the same image can both run the locomotion stack and flash MCUs. CI builds the production image on every commit and pushes it to ECR with a predictable tag scheme so Task 3 can resolve `mainline-latest` without parsing.

## 2. Scope

- **New production Dockerfile:** Add a second Dockerfile alongside the existing one (e.g. `images/locomotion/Dockerfile.release`, or a new sibling directory — exact path is the implementer's call). Installs Krabby packages via `pip install -r requirements.txt` from PyPI. Small `COPY` for entrypoint scripts and config is fine. Base image stays the existing NVIDIA Jetson PyTorch (`nvcr.io/nvidia/pytorch:25.10-py3-igpu` or successor documented in the README).
- **Existing dev image unchanged:** The current `images/locomotion/Dockerfile` keeps its monorepo-`COPY` behavior so developers iterating on HAL or compute code do not have to bump PyPI versions to test changes. The README labels the two images clearly: dev (local source) vs. production (PyPI + pushed to ECR).
- **Pinned requirements:** Commit `images/locomotion/requirements.txt` with exact versions for every Krabby package the production image installs. A `pip freeze` snippet in the README shows expected versions.
- **Flash tooling bundled (production image):** Install `avrdude`, `arduino-cli` (with the Mega 2560 core pinned to the same version Task 1's CI uses to build the HEX), and the latest `krabby-firmware` wheel into the production image. A user who only `pip install krabby` can flash MCUs from inside the container without separately installing flash tools on the host. Host-side udev / `dialout` setup is not done here; that is owned by `krabby --install` (Task 3).
- **CI workflow:** On every push to mainline (and `release/*`), build the production Dockerfile and push to ECR. The dev Dockerfile is not built or pushed by CI. Tags: commit SHA, `<branch>-latest` (e.g. `mainline-latest`), and on tag pushes also the semver tag.
- **ECR:** New repo `krabby-locomotion` (or other standardized naming) in the existing Krabby Co AWS account. Tooling for push from GitHub Actions; image pulls should all be public-read on ECR to avoid forcing auth on Task 3 users.
- **Documentation:** `images/locomotion/README.md` documents both images (dev and production), which PyPI packages the production image installs, how to bump pins, the bundled flash tooling, the ECR URI, and the tag scheme.

## 3. Acceptance Criteria

1. The existing dev Dockerfile in `images/locomotion/` still builds with monorepo `COPY` semantics (no regression).
2. The new production Dockerfile builds without copying the Krabby Python tree, by `pip install`ing the published Krabby packages.
3. Versions inside the production image (`pip freeze`) match the committed `requirements.txt`.
4. CI builds and pushes the production image to ECR on every push to a tracked branch.
5. ECR exposes commit-SHA, `<branch>-latest`, and (on tags) semver tags as documented.
6. Container entrypoint matches the existing production behavior (`python -m hal.server.jetson.main` and existing flags) for both images.
7. `docker run --device /dev/ttyACM0 <production-image> krabby-firmware --show` works against a connected Mega and reports its port and version, with no flash tools installed on the host.
8. `images/locomotion/README.md` documents both images, PyPI packages, pin bumps, bundled flash tooling, ECR URI, and tag scheme without duplicating documentation from other areas (reference as needed).
9. CI runs a post-build verification step on the production image that asserts `pip freeze` for Krabby packages exactly matches the committed `requirements.txt`, and that `krabby-firmware`, `avrdude`, and `arduino-cli` are present and the Mega 2560 core matches Task 1's pin. Build fails if any check fails.

## 4. Time Estimate (~3–4 days)

| Days | Sub-task title |
|------|----------------|
| 1 | New production Dockerfile alongside the existing dev one; pinned PyPI installs |
| 0.5 | `requirements.txt` commit; verify `pip freeze` parity in the production image |
| 1 | ECR repo + IAM + GitHub OIDC role; CI workflow build + push on each commit |
| 0.5 | Tag scheme implementation (commit SHA, `<branch>-latest`, semver on tags) |
| 0.5–1 | README updates and end-to-end smoke test on a Jetson |

## 5. Dependencies

- Existing PyPI publish workflow producing the Krabby wheels.
- AWS account with ECR access; permission to create OIDC role.
- Existing locomotion Dockerfile and `images/locomotion/README.md`.

## 6. Deliverables

- New production Dockerfile (e.g. `images/locomotion/Dockerfile.release`) alongside the unchanged dev Dockerfile, plus committed `requirements.txt`.
- Production image bundles `avrdude`, `arduino-cli` (Mega 2560 core), and `krabby-firmware`; smoke-tested via `docker run --device /dev/ttyACM0 <image> krabby-firmware --show`.
- New ECR repo with at least one mainline production image and the documented tag scheme.
- CI workflow that builds and pushes the production image per commit (dev image stays out of CI).
- Updated `images/locomotion/README.md` covering both images.
