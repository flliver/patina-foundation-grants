# Task 2 – Fleet Portal and Service

## 1. Narrative

Build the fleet **portal** (Next.js) and **fleet service** (Fargate) so operators can list devices and open a device detail view with **portal telemetry**. That telemetry comes from **AWS IoT only**: each robot publishes once per minute (krab health, IMU/pose, power, major red flags); the fleet service reads device shadow and serves it to the portal. Teleop (Task 3) and HAL→S3 historical uploads are separate; this task is device list + device detail + 1/min IoT telemetry only. Deploy fleet service via ECR and githook. Document auth (Cognito) and AWS prerequisites.

## 2. Scope

- **Portal:** Device list (from Greengrass + IoT); device detail page showing 1/min telemetry (health, IMU/pose, power, red flags) in tables/gauges; link for “Open teleop” (Task 3).
- **Fleet service:** REST API; endpoints: list devices (Greengrass list core devices), get telemetry by device ID (from device shadow). Runs in Docker on Fargate. Optionally: read from S3 for historical data (HAL client uploads) if the portal is to show recent history; primary source for “live” portal view is IoT.
- **IoT contract:** Device shadow; payload schema (health, IMU/pose, power, red flags); update rate 1/min. Robot-side publisher (in krabby-bootstrap) must conform; document schema and shadow path.
- **Deploy:** Fleet service Dockerfile, ECR repo, Fargate task definition and service; githook on push to build image and update Fargate. Document flow and “what must exist in AWS” (cluster, ECR, IoT, IAM, VPC/security groups).

- **Mock:** [mocks/mock-fleet-dashboard](mocks/mock-fleet-dashboard).

## 3. Acceptance Criteria

1. Portal shows device list; clicking a device opens device detail view.
2. Device detail displays 1/min telemetry (health, IMU/pose, power, red flags) from IoT via fleet service.
3. Fleet service runs on Fargate; API lists devices (Greengrass) and returns portal telemetry per device (from IoT).
4. Fleet service image built and deployed via ECR + githook; flow documented.
5. 1/min IoT payload schema documented; robot publish path documented (HAL→S3 historical path documented separately if used).
6. Auth (Cognito) and AWS prerequisites documented.

## 4. Time Estimate (~5–7 days)

| Days | Sub-task title |
|------|----------------|
| 1 | Portal scaffold: device list and device detail routes; call fleet service API |
| 1 | Fleet service: list devices (Greengrass), get telemetry (device shadow) |
| 1 | Robot: 1/min publish to IoT (schema); fleet service subscribes; portal displays telemetry |
| 1 | Fargate: Dockerfile, ECR, task definition, service; deploy and verify |
| 0.5 | Githook/CI for fleet service image; document flow |
| 0.5 | Auth and “what must exist in AWS” docs |

## 5. Dependencies

- Task 1: at least one device registered so fleet service can list it; 1/min telemetry can be stubbed until robot publisher exists. M7: schema aligns with M7 observations (reference M7 OVERVIEW Task 3).

## 6. Deliverables

- Portal in `krabby-home/fleet/portal/`; fleet service in `krabby-home/fleet/service/`.
- Fargate service running; ECR + githook/CI documented.
- 1/min payload schema and publish path documented; auth and AWS prereqs documented.
