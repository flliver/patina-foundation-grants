# Task 2 – Fleet Service and Web Portal

## 1. Narrative

Build the **fleet service** (on the single EC2) and the **web portal** (Next.js) so operators can list devices and open a device detail view with **portal telemetry**, and send commands. Telemetry comes over **AWS IoT Core (MQTT)**: each robot's `krabby agent` publishes once per minute to `krab/{thingName}/telemetry`; the fleet service subscribes, upserts the latest into the DynamoDB registry (Task 1), and serves it to the portal. Device online/last-seen comes from IoT presence events. Teleop (Task 3) and HAL→S3 historical uploads are separate; this task is device list + device detail + 1/min telemetry + send-command. Document auth (Cognito) and AWS prerequisites.

## 2. Scope

- **Portal (Next.js):**
  - Device list from the registry: `thingName`, online/last-seen, cohort, `reported.image`.
  - Device detail page showing 1/min telemetry (health, IMU/pose, power, red flags) in tables/gauges.
  - **Send command** UI: publish a command to the device (e.g. `ping`, `reboot`); show the resulting `event`. "Update" and cohort actions are the staged-rollout UI in Task 4.
  - "Open teleop" link (Task 3).
  - Cognito sign-in; device list/detail/commands restricted to authenticated users.

- **Fleet service (single EC2):**
  - Connects to AWS IoT Core as an MQTT client (IAM/cert creds, server-side only); subscribes to `krab/+/telemetry` and `krab/+/event`; upserts latest telemetry into DynamoDB.
  - Consumes presence events (`$aws/events/presence/...`, directly or via the Task 1 routing) to maintain online/last_seen.
  - REST API for the portal: list devices, get device + latest telemetry, **send command** (publish to `krab/{thingName}/cmd`), and (used by Task 4) set desired image (shadow `desired.image`).
  - Runs as a service (systemd or docker-compose) on the same EC2 that hosts the portal and (Task 3) coturn. This is the one always-on box.

- **Telemetry contract:** topic `krab/{thingName}/telemetry`; payload schema (health, IMU/pose, power, red flags); rate 1/min. The Task 1 `krabby agent` must conform; document the schema and topic here.

- **Deploy:** Provision the EC2 + security groups + IAM (IoT connect/subscribe/publish, DynamoDB read/write) via CDK (Task 1 stack or a sibling). Deploy the fleet service + portal with a documented flow (systemd/docker-compose + a deploy script, or a githook/CI that builds and restarts on the EC2). Document "what must exist in AWS" (IoT Core + registry from Task 1, EC2, Cognito, IAM).

- **Mock:** [mocks/mock-fleet-dashboard](mocks/mock-fleet-dashboard).

## 3. Acceptance Criteria

1. Portal shows the device list (online/last-seen, cohort, reported image); clicking a device opens device detail.
2. Device detail displays 1/min telemetry (health, IMU/pose, power, red flags) sourced from MQTT via the fleet service + DynamoDB.
3. Operator can send a command from the portal (e.g. `ping`/`reboot`) and see the device's `event` response.
4. Fleet service runs on the EC2: subscribes to telemetry/events, maintains presence, exposes the REST API; offline devices still appear with last-known state.
5. Fleet service + portal deploy flow documented and reproducible; "what must exist in AWS" documented.
6. 1/min telemetry payload schema and topic documented; Cognito auth and AWS prerequisites documented.

## 4. Time Estimate (~5–7 days)

| Days | Sub-task title |
|------|----------------|
| 1 | Portal scaffold: device list + device detail routes; call fleet service API; Cognito sign-in |
| 1.5 | Fleet service: IoT MQTT subscribe (telemetry/event), presence, upsert DynamoDB; REST API |
| 1 | Send-command path: portal → REST → publish `cmd`; show `event`; align telemetry schema with Task 1 agent |
| 1 | EC2 + security groups + IAM (CDK); deploy fleet service + portal; verify end to end |
| 0.5 | Deploy flow + "what must exist in AWS" + auth docs |

## 5. Dependencies

- **Task 1:** IoT Core, topic scheme, DynamoDB registry, and at least one onboarded device (telemetry can be stubbed by publishing test messages until the agent is live). 1/min telemetry schema aligns with M7 observations (reference M7 OVERVIEW).
- **Task 3** adds the teleop entry point to device detail and shares the EC2 (coturn).

## 6. Deliverables

- Portal in `krabby-home/fleet/portal/`; fleet service in `krabby-home/fleet/service/`.
- Fleet service + portal running on the EC2; CDK for EC2/IAM/security groups; deploy flow documented.
- 1/min telemetry schema + topic documented; Cognito and AWS prerequisites documented.
