# Task 1 – Control Plane and Onboarding (AWS IoT Core + krabby-launcher)

## 1. Narrative

Stand up the **control plane** — AWS IoT Core (managed MQTT) plus a DynamoDB device registry — and extend **`krabby-launcher`** (the M14 on-device CLI) so it both **onboards** a device onto the fleet and runs as the device's **always-on MQTT client**. A fresh Orin runs `krabby enroll` once: it provisions the device's IoT identity (X.509 cert + thing), attaches a topic-scoped policy, assigns a cohort, and enables `krabby agent` (a systemd service) which holds the outbound TLS/MQTT connection. From then on the device publishes telemetry and receives commands over that one connection. This is the foundation the portal (Task 2), teleop (Task 3), and staged OTA (Task 4) all build on.

There is **no separate `krabby-bootstrap` package** — onboarding is part of `krabby-launcher`. Per the milestone decision, the new `enroll` and `agent` subcommands are added to the **`krabby-launcher`** package in `krabby-research` (extending M14's tool); the fleet *service*, *portal*, and *CDK infra* live in `krabby-home/fleet`. `krabby-launcher` still never **pushes** images — updates are pulled by `krabby update` from ECR (M14).

## 2. Scope

- **IoT Core infra (CDK):**
  - **Thing type** for Krabs; **thing groups** as cohorts: `cohort-test`, `cohort-canary`, `cohort-fleet` (plus `unassigned` for freshly enrolled devices).
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

- **Device authentication — X.509 mutual TLS:** each krab authenticates to IoT Core with its **own X.509 certificate** over mutual TLS (port 8883); authorization comes from the topic-scoped IoT policy above. Prefer generating the keypair **on-device via CSR** so the private key never leaves the Orin. The device also stores the account **ATS endpoint** and the **Amazon Root CA**. No long-lived AWS/IAM credentials live on the device in steady state. (The fleet service, by contrast, talks to IoT Core/DynamoDB using its **EC2 instance IAM role** — Task 2.)

- **DynamoDB registry:** one table keyed by `thingName`, holding at least: cohort, online/last_seen, latest 1/min telemetry, `reported.image` / `desired.image`, created_at. Written on enroll and on telemetry/presence. This is what the portal lists (Task 2).

- **`krabby enroll` (new `krabby-launcher` subcommand):** the fleet-onboarding action (run by an operator, not the kit owner).
  - Default flow uses AWS creds available **at enroll time** to: create/find the thing, generate a cert (CSR), attach the IoT policy, add the thing to a cohort (default `unassigned`; `--cohort <name>` to override), write the cert/key + ATS endpoint + root CA to the device, enable the `krabby agent` systemd service, and verify the device connects (appears `online` in the registry). The AWS creds are used **once at enroll** and need not persist.
  - Flags: `--thing-name <name>` (default derived from hostname/serial), `--cohort <name>`, `--endpoint <iot-ats-endpoint>` (default from account).
  - **Scale path (document, don't have to implement):** AWS IoT **Fleet Provisioning by claim** — devices ship with a shared low-privilege claim cert and self-provision a unique cert on first connect, so onboarding needs no per-device AWS creds. Note this as the path beyond the initial small fleet.

- **`krabby agent` (new `krabby-launcher` subcommand):** the always-on MQTT client, run as a systemd service:
  - Publishes `krab/{thingName}/telemetry` once per minute (schema owned by Task 2).
  - Subscribes to `krab/{thingName}/cmd` and the shadow delta; on an `update`/desired-image change, invokes `krabby update [--image <ref>]` and reports result on `krab/{thingName}/event` and `reported.image` in the shadow.
  - Carries teleop signaling between IoT and the M7 WebRTC agent (mechanism documented in Task 3 — local IPC/ZMQ handoff, or the app subscribing directly).
  - Reconnects forever with backoff; survives reboot.

- **Packaging:** `enroll` and `agent` ship **inside the existing `krabby-launcher` package** and are published by M14's existing publish workflow — **no new PyPI package**. Document the version that first includes them.

- **SETUP doc:** `krabby-home/fleet/SETUP-FLEET.md`: deploy the CDK infra; run `krabby enroll` on an Orin; topic scheme and policy; the `krabby agent` service; troubleshooting (cert/policy, endpoint, connection).

## 3. Acceptance Criteria

1. CDK stack creates IoT Core resources: thing type, cohort thing groups, least-privilege policy (per-thing topic isolation), and the registry DynamoDB table.
2. `krabby enroll` on an Orin provisions identity (X.509 cert + thing + policy + cohort), enables the `krabby agent` service, and the device connects to IoT Core via mutual TLS.
3. After enroll, the device appears in the DynamoDB registry as `online` with its cohort and (initially) `reported.image`; disconnecting flips it to offline/last_seen via presence events.
4. `krabby agent` publishes 1/min telemetry to `krab/{thingName}/telemetry` and acts on a test command published to `krab/{thingName}/cmd` (e.g. `ping` → `event`).
5. A change to the device's desired image (shadow `desired.image`) triggers the agent to call `krabby update` and report `reported.image` back.
6. Per-device topic isolation verified: a device's cert cannot publish/subscribe on another device's topics.
7. `enroll` + `agent` ship in the `krabby-launcher` package (published via the M14 workflow); the including version and `SETUP-FLEET.md` are documented.

## 4. Time Estimate (~5–7 days)

| Days | Sub-task title |
|------|----------------|
| 1.5 | CDK: IoT Core (thing type, cohort groups, least-privilege policy, topic scheme, shadow), DynamoDB registry, presence routing |
| 1.5 | `krabby enroll`: provision identity (CSR), attach policy + cohort, write creds/endpoint/CA, verify connect |
| 2 | `krabby agent`: MQTT connect, 1/min telemetry, cmd + shadow-delta handling, invoke `krabby update`, reconnect/systemd |
| 0.5 | Presence → registry; per-device isolation check |
| 0.5 | `SETUP-FLEET.md` + troubleshooting |

## 5. Dependencies

- **M14:** `krabby-launcher` (the `krabby` command) is the package we extend; `krabby update` does the actual pull/install. Coordinate with the M14 owner since `enroll`/`agent` land in that package.
- **AWS:** IoT Core, DynamoDB, IAM, Lambda (if used for presence routing) in account krabby-company (6329-1496-1627). J4012 (Jetpack 6/7), Docker, network to AWS.
- **Task 2** consumes the registry + topic scheme; **Task 3** uses the teleop signaling topics; **Task 4** uses cohorts + desired/reported image.

## 6. Deliverables

- `krabby enroll` + `krabby agent` added to the **`krabby-launcher`** package (`krabby-research`), published via the M14 workflow.
- CDK infra for IoT Core (things, cohort groups, least-privilege policy, shadow), DynamoDB registry, and presence routing, in `krabby-home/fleet/`.
- `SETUP-FLEET.md` (topic scheme, policy, `krabby enroll`/`krabby agent`, troubleshooting).
