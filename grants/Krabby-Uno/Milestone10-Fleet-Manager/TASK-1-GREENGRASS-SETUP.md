# Task 1 – Greengrass Setup (J4012 + krabby-bootstrap)

## 1. Narrative

Deliver a **krabby-bootstrap** PyPI package installable on an Orin (`pip install krabby-bootstrap`) with a **krabby-bootstrap** CLI that uses default AWS credentials to talk to Greengrass and deploy the mainline Krabby image (optional `--image <uri>` for a custom ECR image). The tool is the primary path to bring up an Orin with the Krabby app. Publish via GitHub Actions on version tags, following the krabby-research pattern (`docs/PUBLISHING.md`, `.github/workflows/publish-packages.yml`). Package lives in krabby-home/fleet.

## 2. Scope

- **Package:** PyPI name `krabby-bootstrap`; entry point `krabby-bootstrap`. Build produces a wheel; workflow runs on tag (e.g. `krabby-bootstrap-v*`), builds, runs tests, uploads to PyPI.
- **CLI behavior:** Reads default AWS creds from `~/.aws/credentials`. Calls Greengrass APIs to create/update a deployment that targets the current device with the Krabby component. Default image = mainline (well-known ECR URI/tag). `--image <ECR-uri-or-tag>` overrides. The tool can install Docker and optionally Greengrass during `krabby-bootstrap --setup`; document pre- vs post-provision usage.
- **Greengrass / component:** J4012 (Jetpack 6 and 7 supported); Greengrass Core on device; Krabby app runs as a Greengrass Docker application component (image from ECR). Component definition references ECR image; no special base image inside the app. ECR repo in account krabby-company 6329-1496-1627.
- **Revert:** Document manual redeploy to previous image (new deployment with previous ECR tag). Deployments must use `failureHandlingPolicy: ROLLBACK`.

## 3. Acceptance Criteria

1. `pip install krabby-bootstrap` works on any Linux with Python and AWS creds (tested on Orin).
2. `krabby-bootstrap` runs and deploys the mainline image using default AWS creds.
3. `krabby-bootstrap --image <uri>` deploys the given image.
4. GitHub Actions workflow publishes the wheel to PyPI on version tag; tag pattern and package name documented.
5. Greengrass Core can be installed and provisioned on J4012; device appears as a Greengrass core device. Intended use (bootstrap tool pre- vs post-provision) documented.
6. Krabby component definition exists (ECR image, Docker Application Manager); revert procedure documented.
7. SETUP.md at `krabby-home/fleet/SETUP-GREENGRASS.md`: install Greengrass, `pip install` and usage of krabby-bootstrap, component/revert, troubleshooting.

## 4. Time Estimate (~1 day)

| Hours | Sub-task title |
|-------|----------------|
| 2 | Package (pyproject.toml, entry point), CLI (AWS creds, Greengrass deploy mainline + `--image`) |
| 1 | GitHub Actions workflow (tag-based build + PyPI upload); document tag pattern |
| 1 | Greengrass install/provision on J4012; component definition; verify deploy with krabby-bootstrap |
| 0.5 | Revert procedure + SETUP.md |

## 5. Dependencies

- ECR repo for Krabby images; IAM for Greengrass pull and for Orin creds (deployments). J4012 (Jetpack 6 and 7), Docker, network to AWS.

## 6. Deliverables

- krabby-bootstrap wheel on PyPI; CLI (default mainline, optional `--image`).
- GitHub workflow (tag → build → PyPI); SETUP.md at `krabby-home/fleet/SETUP-GREENGRASS.md`.
- Greengrass on at least one J4012; component definition; revert documented.
