# Task 2 – Fleet Service, Web Portal, and `krabby-fleet` CLI

## 1. Narrative

Build the fleet service (on a single EC2), the Next.js portal, and the `krabby-fleet` CLI so operators can list devices, see 1/min telemetry, launch a WebRTC teleop session (Task 3), and open a Secure-Tunnel SSH session to any registered robot from anywhere on the internet. The CLI is the operator entry point.

## 2. Scope

- **Portal (Next.js):**
  - Device list from the registry: `thingName`, online/last-seen, `reported_image`.
  - Device detail with 1/min telemetry (health, IMU/pose, power, red flags).
  - "Open teleop" link (Task 3).
  - "Open SSH" per-device button → calls the tunnel endpoint below and shows the operator a `krabby-fleet ssh <thingName>` one-liner (with a copy-paste `aws-iot-securetunneling-localproxy` invocation as fallback).

- **`krabby-fleet` CLI:**
  - `list` — hit `/devices`, print online/last-seen + latest telemetry summary per robot.
  - `teleop <robot>` — hit the teleop session endpoint (Task 3) and open the browser to the authenticated portal teleop URL.
  - `ssh <robot>` — hit the tunnel endpoint below, start `localproxy` in source mode locally against the returned source token, then `exec ssh operator@localhost -p <local-port>`. On exit, close the tunnel via the force-close endpoint.
  - Config at `~/.config/krabby-fleet/config.toml` (fleet-service URL, Cognito user pool, default SSH user). Short-lived credentials refreshed like `aws sso login`.

- **Fleet service (application code) — thin auth-proxy over AWS IoT:**
  - Uses the EC2 instance IAM role to call AWS IoT on the operator's behalf. No persistent MQTT subscriber daemon is needed for the list/detail view — IoT is the store.
  - REST API:
    - `GET /devices` → `iot:SearchIndex` (query for the fleet's thing type) returning each thing's `connectivity.connected` / `connectivity.timestamp` + latest `shadow.reported.*` in one call.
    - `GET /devices/{thingName}` → `iot:GetThingShadow` for the full latest `state.reported` payload + `iot:DescribeThing` for connectivity metadata.
  - **Open SSH tunnel endpoint:** `POST /devices/{thingName}/ssh-tunnel` → calls Secure Tunneling `OpenTunnel` (services=`SSH`), which delivers the destination token to the device on `$aws/things/{thingName}/tunnels/notify` (handled by `krabby agent`, Task 1). Returns a **short-lived source client access token** + tunnel metadata; browser never sees any AWS credentials. `DELETE .../ssh-tunnel/{tunnelId}` for force-close (idle sessions also expire on `OpenTunnel`'s max lifetime, default ≤12h).
  - The service also holds a persistent MQTT connection for the **Task 3 signaling bridge only** (browser ↔ `teleop/{thingName}/signaling/in|out`). That connection is unrelated to telemetry storage.

- **Telemetry contract:** device writes 1/min to `$aws/things/{thingName}/shadow/update` (Task 1 topic). Document the `state.reported` payload schema here (health, IMU/pose, power, red flags, `reported_image`).

- **Runtime and hosting — one EC2, three systemd units, Caddy for TLS:**
  - **Instance:** one `t3.small` in `krabby-company` in the fleet region; EBS root, IMDSv2, EC2 instance IAM role that grants only what the fleet service needs (see IAM below). Dedicated Elastic IP so the DNS record is stable across restarts.
  - **What runs on it (all systemd units, all logs to journald):**
    - `krabby-fleet-service.service` — the Python (or Node — implementer's call) backend: REST API (`iot:SearchIndex` / `iot:GetThingShadow` proxy), Secure Tunneling `OpenTunnel`/`CloseTunnel` endpoints, and the Task 3 signaling WebSocket bridge. Bound to `127.0.0.1:8080`.
    - `krabby-fleet-portal.service` — Next.js standalone server (`node server.js` from `next build --output=standalone`). Bound to `127.0.0.1:3000`.
    - `caddy.service` — Caddy 2, listening on 443, terminating TLS via Let's Encrypt (auto-renewal), reverse-proxying `/api/*` to `:8080` and everything else to `:3000`. Also fronts the Task 3 signaling WebSocket.
    - `coturn.service` — coturn (Task 3), UDP 3478 (+ optional TURNS 5349) exposed via the security group; short-lived TURN credentials are minted by the fleet-service `POST /devices/{thing}/teleop/signaling` endpoint using a shared secret.
  - **DNS + TLS:** one Route 53 A record for the fleet host (e.g. `fleet.krabby.co`), pointed at the EIP. Caddy handles the ACME dance automatically. Document the record + DNS delegation in `SETUP-FLEET.md`.

- **CDK layout — two stacks, both in `krabby-home/fleet/infra/`:**
  - `ControlPlaneStack` (owned by **Task 1**): IoT Core thing type + per-thing policy + Fleet Indexing configuration + presence lifecycle events. Exports: `IotAtsEndpoint`.
  - `FleetServiceStack` (owned by **Task 2**): EC2 instance + security groups + EIP + Route 53 record, EC2 instance IAM role granting: `iot:SearchIndex` on the fleet index, `iot:GetThingShadow` / `iot:DescribeThing` scoped to the fleet's things, `iot:Connect`/`iot:Subscribe`/`iot:Receive`/`iot:Publish` on the Task 3 `teleop/*/signaling/*` topics for the signaling bridge, `iotsecuretunneling:OpenTunnel`/`CloseTunnel`/`DescribeTunnel` scoped to the fleet's things, `cognito-identity:GetCredentialsForIdentity` for the OIDC flow. Coturn security-group rules (UDP 3478/5349). Depends on `ControlPlaneStack` via CFN exports.
  - `cdk deploy ControlPlaneStack` and `cdk deploy FleetServiceStack` are the two deploy commands; the CDK app resolves the dependency graph automatically. `SETUP-FLEET.md` documents the sequence for a fresh account.

- **CI/CD — GitHub Actions in `krabby-home/.github/workflows/`, OIDC to AWS, no long-lived deploy keys:**
  - `fleet-ci.yml` runs on every PR + every merge to `main`. Steps: `pnpm install && pnpm test` (portal unit tests) → `pytest fleet/service/tests` (backend unit tests) → run the **bench integration test** (Task 1 + Task 2 + Task 3 tests, all pointing at Bruce's bench Krabby via the currently-deployed dev-account fleet service). Bench offline or any test failure → PR red.
  - `fleet-deploy.yml` runs on push to `main` **after** `fleet-ci.yml` passes. Steps: `cdk synth` → `cdk diff` (posted to the merge commit for review) → `cdk deploy ControlPlaneStack FleetServiceStack --require-approval never` → build+push new fleet-service and portal artifacts to S3 (or a small tar bundle) → SSM Run Command on the EC2: `systemctl restart krabby-fleet-service krabby-fleet-portal`. Rollback is `cdk deploy` at the previous git SHA.
  - AWS access uses **OIDC federation** — a GitHub Actions IAM role provisioned by `FleetServiceStack` trusts the `krabby-home` repo's `main` branch and PR runs from the same repo (not forks). No AWS access keys stored in GitHub.
  - Bench integ tests use a **CI Cognito user** whose credentials live in GitHub Actions secrets and are rotated by a scheduled job. The bench thing (`bench-krabby-ci`) is permanently enrolled and its cert lives on the bench Orin — the tests never re-enroll it.

- **Mock:** [mocks/mock-fleet-dashboard](mocks/mock-fleet-dashboard).

## 3. Acceptance Criteria

1. Portal shows the device list; clicking a device opens device detail with 1/min telemetry.
2. `krabby-fleet list` prints the same list from the CLI (Cognito-authenticated).
3. **Open SSH** — from the portal button *or* `krabby-fleet ssh <robot>` — opens a Secure Tunnel and drops the operator into an SSH session. Demonstrated end-to-end against the two named robots from the OVERVIEW, from an operator laptop anywhere on the internet, by both Fletcher and Bruce (each with their own Cognito account). Force-close from the portal or Ctrl+C from the CLI terminates the session.
4. Only authenticated Cognito users can open a tunnel; browser never sees AWS credentials.
5. Fleet service exposes the REST list/detail API backed by `iot:SearchIndex` / `iot:GetThingShadow`; offline devices still appear with their last-known shadow state and `connectivity.timestamp`.
6. **Runtime as described:** `krabby-fleet-service`, `krabby-fleet-portal`, `caddy`, and `coturn` all run as separate systemd units on one EC2, all logging to journald. `journalctl -u krabby-fleet-service -f` shows live logs. The portal is served over HTTPS on a stable DNS name via Caddy-managed Let's Encrypt; cert auto-renews.
7. **CDK layout as described:** two stacks in `krabby-home/fleet/infra/` — `ControlPlaneStack` (Task 1) and `FleetServiceStack` (Task 2). `FleetServiceStack` consumes `ControlPlaneStack`'s `IotAtsEndpoint` export via CFN; a fresh account is stood up with `cdk deploy ControlPlaneStack FleetServiceStack`. `SETUP-FLEET.md` documents the sequence.
8. **CI/CD as described:** `fleet-ci.yml` runs on every PR + merge and executes portal unit tests, backend unit tests, and the Task 1 + Task 2 + Task 3 bench integration tests against Bruce's bench Krabby; PR red if bench offline or any test fails. `fleet-deploy.yml` runs on merge to `main` after CI passes: `cdk diff` posted to the merge commit, `cdk deploy` runs both stacks, SSM Run Command restarts the systemd units. AWS access via OIDC federation from a GitHub Actions IAM role provisioned by `FleetServiceStack` — no long-lived AWS credentials in GitHub. Rollback documented as `cdk deploy` at a prior git SHA.
9. Deploy flow, "what must exist in AWS," telemetry schema + topic, and auth are documented.
10. **Automated E2E integration test** runs on **Bruce's bench Krabby** on every build (every PR + every merge) against a real dev-account fleet service + IoT Core — the bench is the SSH target, no emulated device. Test covers: (a) a Cognito test user is provisioned (or a persistent CI user is reused), (b) `GET /devices` with the Cognito access token asserts the bench thing is listed with recent telemetry, (c) **negative** — the endpoint returns 401 without a valid token, (d) test invokes the packaged `krabby-fleet ssh bench-krabby-ci` as a subprocess, waits for the local SSH port, runs `ssh -o StrictHostKeyChecking=no operator@localhost -p <port> echo hello` through it, and asserts `hello` comes back — proving the full portal→`OpenTunnel`→notify→destination-`localproxy` (on the bench)→source-`localproxy` (in CI)→ssh path against real hardware, (e) `DELETE /devices/{thing}/ssh-tunnel/{id}` asserts `krabby agent` on the bench kills the destination proxy within N seconds, (f) **negative** — a Cognito user without the `operator` group cannot open a tunnel. Teardown removes any scratch Cognito user + closes the tunnel; the bench identity is preserved. Bench offline = CI red.

## 4. Time Estimate (~7–9 days)

| Days | Sub-task title |
|------|----------------|
| 1 | Portal scaffold: device list + detail routes; Cognito sign-in |
| 1 | Fleet service: REST list/detail proxying `iot:SearchIndex` + `iot:GetThingShadow`; Cognito JWT verification middleware |
| 1.5 | Open-SSH: `OpenTunnel` endpoint, force-close, `krabby-fleet ssh` wrapper, portal button |
| 1 | `krabby-fleet list` + `teleop` (browser launch); CLI packaging (PyPI or single-binary) |
| 1 | `FleetServiceStack` CDK: EC2 + EIP + Route 53 + SG + IAM role + OIDC role for CI; consume `ControlPlaneStack` exports |
| 1 | Systemd units for service + portal + Caddy; Caddy config + Let's Encrypt; deploy end to end and verify against both robots |
| 1 | `fleet-ci.yml` + `fleet-deploy.yml` GitHub Actions: unit tests, bench integ test run, `cdk diff` on PR, `cdk deploy` + SSM restart on merge, OIDC federation |
| 1 | E2E integration test (auth + list + open-SSH round-trip through the CLI + force-close + auth negative) |
| 0.5 | Deploy + AWS prereqs docs (topology diagram, DNS/TLS, rollback recipe) |

## 5. Dependencies

- **Task 1:** IoT Core + per-thing policy + Fleet Indexing enabled + at least one onboarded device (`bench-krabby-ci` for the integration test).
- **Task 3:** teleop session endpoint (this task exposes `krabby-fleet teleop` and the "Open teleop" link; Task 3 provides the backend).

## 6. Deliverables

- Portal in `krabby-home/fleet/portal/`; fleet service in `krabby-home/fleet/service/`; `krabby-fleet` CLI in `krabby-home/fleet/cli/`.
- Fleet service + portal running on the EC2; CDK for EC2/IAM/SG.
- Telemetry schema + topic documented; auth + AWS prerequisites documented.
