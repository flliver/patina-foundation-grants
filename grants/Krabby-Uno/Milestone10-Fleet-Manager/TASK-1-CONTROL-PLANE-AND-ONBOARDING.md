# Task 1 – Control Plane and Onboarding (AWS IoT Core + krabby-bootstrap)

## 1. Narrative

Stand up the **control plane** — AWS IoT Core (managed MQTT) plus a DynamoDB device registry — and repurpose **`krabby-bootstrap`** into the device onboarding tool. A fresh Orin runs `krabby-bootstrap` once: it provisions the device's IoT identity (X.509 cert + thing), installs a lightweight **fleet agent**, assigns the device to a cohort, and auto-connects to IoT Core. From then on the device holds one outbound TLS/MQTT connection over which it publishes telemetry and receives commands. This is the foundation the portal (Task 2), teleop (Task 3), and staged OTA (Task 4) all build on.

`krabby-bootstrap` **no longer deploys images** — that is the M14 launcher's job — the `krabby-launcher` package's `krabby install` / `update`. It now only onboards a device onto the fleet.

## 2. Scope

- **IoT Core infra (CDK):**
  - **Thing type** for Krabs; **thing groups** as cohorts: `cohort-test`, `cohort-canary`, `cohort-fleet` (plus `unassigned` for freshly bootstrapped devices).
  - **Least-privilege IoT policy** using policy variables so a device certificate can connect **only** as its own thing and publish/subscribe **only** on its own topics — e.g. allow `Connect` where client id == `${iot:Connection.Thing.ThingName}`, and `Publish`/`Subscribe`/`Receive` scoped to `krab/${iot:Connection.Thing.ThingName}/*` and `teleop/${iot:Connection.Thing.ThingName}/*`. A compromised device cannot touch another device's topics.
  - **Topic scheme** (document and fix this contract):

    | Topic | Direction | Purpose |
    |-------|-----------|---------|
    | `krab/{thingName}/telemetry` | device → cloud | 1/min health (schema in Task 2) |
    | `krab/{thingName}/cmd` | cloud → device | commands: `{ "type": "...", ... }` (e.g. `update`, `reboot`, `ping`) |
    | `krab/{thingName}/event` | device → cloud | command acks, results, lifecycle/log events |
    | `teleop/{thingName}/signaling/in` | cloud → device | WebRTC SDP/ICE to robot (Task 3) |
    | `teleop/{thingName}/signaling/out` | device → cloud | WebRTC SDP/ICE from robot (Task 3) |

  - **Desired/reported image state:** use the device **Classic Shadow** (`desired.image` / `reported.image`) so desired state survives reconnects and the device gets a delta when it comes back online. (Evaluate **AWS IoT Jobs** as the managed alternative for staged rollout in Task 4; Task 1 only needs to make the desired-state mechanism exist.)
  - **Presence:** enable IoT lifecycle events (`$aws/events/presence/connected|disconnected/{clientId}`) and route them (IoT Rule → Lambda or fleet service) to update `online` / `last_seen` in the registry.

- **DynamoDB registry:** one table keyed by `thingName`, holding at least: cohort, online/last_seen, latest 1/min telemetry, `reported.image` / `desired.image`, created_at. Written on provision and on telemetry/presence. This is what the portal lists (Task 2).

- **`krabby-bootstrap` (PyPI):** `pip install krabby-bootstrap`; entry point `krabby-bootstrap`.
  - Default flow uses AWS creds from `~/.aws/credentials` to: create/find the thing, generate a key + certificate, attach the IoT policy, add the thing to a cohort group (default `unassigned`; `--cohort <name>` to override), write credentials + endpoint to the device, install and enable the **fleet agent** (systemd), and verify the device connects (appears `online` in the registry).
  - Flags: `--thing-name <name>` (default derived from hostname/serial), `--cohort <name>`, `--endpoint <iot-ats-endpoint>` (default from account), `--setup` (install agent prerequisites/Docker if needed).
  - **Scale path (document, don't have to implement):** AWS IoT **Fleet Provisioning by claim** — devices ship with a shared claim cert and self-provision a unique cert on first connect, so onboarding needs no per-device AWS creds. Note this as the path beyond the initial small fleet.

- **Fleet agent (installed by krabby-bootstrap):** lightweight systemd service that owns the MQTT connection:
  - Publishes `krab/{thingName}/telemetry` once per minute (schema owned by Task 2).
  - Subscribes to `krab/{thingName}/cmd` and the shadow delta; on an `update`/desired-image change, invokes the M14 launcher (`krabby update [--image <ref>]`) and reports result on `krab/{thingName}/event` and `reported.image` in the shadow.
  - Carries teleop signaling between IoT and the M7 WebRTC agent (mechanism documented in Task 3 — local IPC/ZMQ handoff, or the app subscribing directly).
  - Reconnects forever with backoff; survives reboot.

- **Publish:** GitHub Actions workflow builds the `krabby-bootstrap` wheel and uploads to PyPI on a version tag (e.g. `krabby-bootstrap-v*`), following the krabby-research publishing pattern. Document the tag pattern and package name.

- **SETUP doc:** `krabby-home/fleet/SETUP-FLEET.md`: deploy the CDK infra; `pip install` + run `krabby-bootstrap` on an Orin; topic scheme and policy; fleet-agent service; troubleshooting (cert/policy, endpoint, connection).

## 3. Acceptance Criteria

1. CDK stack creates IoT Core resources: thing type, cohort thing groups, least-privilege policy (per-thing topic isolation), and the registry DynamoDB table.
2. `pip install krabby-bootstrap` works on an Orin; `krabby-bootstrap` provisions identity, installs+enables the fleet agent, and the device connects to IoT Core.
3. After bootstrap, the device appears in the DynamoDB registry as `online` with its cohort and (initially) `reported.image`; disconnecting flips it to offline/last_seen via presence events.
4. The fleet agent publishes 1/min telemetry to `krab/{thingName}/telemetry` and acts on a test command published to `krab/{thingName}/cmd` (e.g. `ping` → `event`).
5. A change to the device's desired image (shadow `desired.image`) triggers the agent to call the M14 launcher and report `reported.image` back.
6. Per-device topic isolation verified: a device's cert cannot publish/subscribe on another device's topics.
7. GitHub Actions publishes the wheel on tag; tag pattern and package name documented. `SETUP-FLEET.md` written.

## 4. Time Estimate (~5–7 days)

| Days | Sub-task title |
|------|----------------|
| 1.5 | CDK: IoT Core (thing type, cohort groups, least-privilege policy, topic scheme, shadow), DynamoDB registry, presence routing |
| 1.5 | `krabby-bootstrap`: provision identity, attach policy + cohort, write creds, verify connect |
| 1.5 | Fleet agent: MQTT connect, 1/min telemetry, cmd + shadow-delta handling, invoke `krabby update`, reconnect/systemd |
| 0.5 | GitHub Actions publish (tag → build → PyPI) |
| 0.5 | `SETUP-FLEET.md` + troubleshooting |

## 5. Dependencies

- **M14:** the `krabby-launcher` package (providing the `krabby` command: `install` / `update`) must be installable on the device; the fleet agent shells out to it for updates.
- **AWS:** IoT Core, DynamoDB, IAM, Lambda (if used for presence routing) in account krabby-company (6329-1496-1627). J4012 (Jetpack 6/7), Docker, network to AWS.
- **Task 2** consumes the registry + topic scheme; **Task 3** uses the teleop signaling topics; **Task 4** uses cohorts + desired/reported image.

## 6. Deliverables

- `krabby-bootstrap` wheel on PyPI; CLI that onboards an Orin (identity + fleet agent + connect).
- CDK infra for IoT Core (things, cohort groups, least-privilege policy, shadow), DynamoDB registry, and presence routing, in `krabby-home/fleet/`.
- Fleet agent (systemd) in `krabby-home/fleet/`.
- GitHub workflow (tag → build → PyPI); `SETUP-FLEET.md` (topic scheme, policy, onboarding, troubleshooting).
