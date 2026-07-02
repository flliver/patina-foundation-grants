# Task 1 – Control Plane and Onboarding (AWS IoT Core + krabby-launcher)

## 1. Narrative

Stand up the control plane entirely on AWS IoT Core — thing registry, per-thing Classic Shadow (as the "latest state per device" store), Fleet Indexing (to make shadow + connectivity queryable for the portal), and Secure Tunneling. Then extend **`krabby-launcher`** so it both onboards a device (`krabby enroll`) and runs as the device's always-on MQTT client (`krabby agent`). This is the foundation the portal + CLI (Task 2) and teleop (Task 3) build on.

## 2. Scope

- **IoT Core infra (CDK):**
  - Thing type for Krabs.
  - **Classic Shadow** for every thing (available by default once the thing exists). Device writes 1/min health telemetry into `state.reported` (schema owned by Task 2, includes `reported_image`); AWS persists it as the "latest state per device" store.
  - **Fleet Indexing enabled** on `thing`, `connectivity`, and `shadow` (Classic Shadow). This makes both connectivity status (`connectivity.connected`, `connectivity.timestamp`) and shadow contents queryable via `iot:SearchIndex`, which is how Task 2's `GET /devices` works.
  - **Least-privilege IoT policy** using policy variables so a device certificate can connect **only** as its own thing and publish/subscribe **only** on its own topics — `Connect` where client id == `${iot:Connection.Thing.ThingName}`, `Publish`/`Subscribe`/`Receive` on the classic shadow topics for its own thing (`$aws/things/${iot:Connection.Thing.ThingName}/shadow/update`, `.../update/accepted`, `.../update/rejected`), `Publish`/`Subscribe`/`Receive` on `teleop/${iot:Connection.Thing.ThingName}/signaling/*`, and `Subscribe`/`Receive` on the reserved Secure Tunneling notify topic `$aws/things/${iot:Connection.Thing.ThingName}/tunnels/notify`.
  - **Topic scheme** — the shape below is a **starting recommendation, not a fixed contract**. Review the AWS IoT Core reserved topics + Shadow + Secure Tunneling docs and settle on a scheme that (a) gives each thing a per-`thingName` namespace enforced by the policy, (b) covers the three flows this milestone actually uses (1/min shadow update up, WebRTC signaling both directions, Secure Tunneling notify down), and (c) doesn't paint us into a corner if later work adds more. Whatever you land on, document it in `SETUP-FLEET.md` and align Task 2 (shadow reported schema) and Task 3 (teleop signaling) to it.

    | Topic (recommended) | Direction | Purpose |
    |-------|-----------|---------|
    | `$aws/things/{thingName}/shadow/update` | device → cloud | 1/min health published as `{"state": {"reported": {...}}}` (schema in Task 2). Reserved shadow topic. |
    | `teleop/{thingName}/signaling/in` | cloud → device | WebRTC SDP/ICE to robot (Task 3) |
    | `teleop/{thingName}/signaling/out` | device → cloud | WebRTC SDP/ICE from robot (Task 3) |
    | `$aws/things/{thingName}/tunnels/notify` | cloud → device | Secure Tunneling destination-token delivery (reserved by AWS; not negotiable) |

  - **Presence:** enable IoT lifecycle events (`$aws/events/presence/connected|disconnected/{clientId}`) so Fleet Indexing keeps `connectivity.connected` / `connectivity.timestamp` current. No Lambda routing needed — Fleet Indexing consumes the events natively; the portal reads via `iot:SearchIndex`.

- **Device auth — X.509 mutual TLS:** each krab authenticates to IoT Core with its own cert over mutual TLS (port 8883); authorization from the policy above. Keypair generated on-device via CSR so the private key never leaves the Orin. Device also stores the ATS endpoint and Amazon Root CA.

- **`krabby enroll`:** the fleet-onboarding action (run by an operator, not the kit owner). Uses AWS creds available at enroll time to: create/find the thing, generate a cert (CSR), attach the IoT policy, write cert/key + ATS endpoint + root CA to the device, install `aws-iot-securetunneling-localproxy`, enable the `krabby-agent` systemd service, and verify the device connects. AWS creds are used **once** and need not persist.
  - Flags: `--thing-name <name>` (default from hostname/serial), `--endpoint <iot-ats-endpoint>` (default from account).
  - **Scale path (document, don't implement):** AWS IoT Fleet Provisioning by claim, for onboarding without per-device AWS creds beyond the initial small fleet.

- **`krabby agent`:** always-on MQTT client, systemd service:
  - Once per minute, publishes `{"state": {"reported": {...}}}` to `$aws/things/{thingName}/shadow/update` (schema owned by Task 2, including the running image tag as `reported_image`). Uses the AWS IoT device SDK's shadow client so `update/accepted`/`update/rejected` are handled for free.
  - Carries teleop signaling between IoT (`teleop/{thingName}/signaling/in|out`) and the M7 WebRTC edge agent (mechanism documented in Task 3 — local IPC/ZMQ handoff or the app subscribing directly).
  - On a Secure Tunneling notification (`$aws/things/{thingName}/tunnels/notify`), spawns the AWS `localproxy` in destination mode against `localhost:22`; kills it when the tunnel closes.
  - Reconnects forever with backoff; survives reboot. The single MQTT connection carries all three flows above.

- **Where the code goes — extending `krabby-launcher` in place:** the M14 launcher already uses a trivial argparse-per-subcommand pattern. `enroll` and `agent` become peers of the existing `install` / `update` / `run` / `firmware` — same file layout, same dispatch. Concretely:

    - Add two new files in `krabby-research/krabby/`: `enroll.py` (exports `cmd_enroll(...)`) and `agent.py` (exports `cmd_agent(...)`). Mirror the shape of `install.py` — one `cmd_*` function per file with keyword args matching the argparse flags. See `krabby-research/krabby/install.py` and `krabby-research/krabby/__main__.py` for the pattern.
    - In `krabby-research/krabby/__main__.py`, add two `sub.add_parser(...)` blocks for `enroll` and `agent` alongside the existing ones, and two `elif args.command == "enroll" / "agent":` branches that lazily import and call the `cmd_*` function. **Keep the imports lazy** (inside the branch, not at module top) so that `krabby --version` and unrelated subcommands don't pay the cost of loading the AWS IoT SDK.
    - Any code shared between `enroll` and `agent` (MQTT connect helpers, cert paths, ATS endpoint discovery, systemd-unit installer) goes in a new `krabby-research/krabby/_iot.py` — follow the leading-underscore convention already used by `_state.py`, `_docker.py`, `_host.py`. Don't invent a class hierarchy; if it's a function, make it a function.
    - The `krabby-agent.service` systemd unit is built as an inline string in `_iot.py` (or in `enroll.py` if only enroll uses it) and installed with the same helper pattern `_host.py` uses for `krabby-locomotion.service` — see `_BOOT_SERVICE_PATH` in `krabby-research/krabby/_host.py:256`. `ExecStart` is literally `krabby agent`; the daemon IS the CLI subcommand.
    - `pyproject.toml`: add the AWS IoT device SDK to `dependencies` (currently `[]`). The existing `krabby = "krabby.__main__:main"` `[project.scripts]` entry covers the new subcommands.

    Deliverable: one PR against `krabby-research` adding the two files + the `__main__.py` wiring + the `_iot.py` helper + the pyproject dep. Coordinate with the M14 owner (James) so his release cadence keeps working. Document the first-including version.

- **SETUP-FLEET.md:** deploy the CDK infra, run `krabby enroll`, topic scheme + policy, the `krabby-agent` systemd unit, troubleshooting.

## 3. Acceptance Criteria

1. CDK stack creates the IoT Core resources: thing type, per-thing least-privilege policy (covering shadow update, teleop signaling, and Secure Tunneling notify), Fleet Indexing enabled on `thing` + `connectivity` + `shadow`, presence lifecycle events enabled.
2. `krabby enroll` on an Orin provisions identity, enables `krabby-agent`, and the device connects to IoT Core via mutual TLS.
3. After enroll, `iot:SearchIndex` for the thing returns `connectivity.connected: true` with a recent `connectivity.timestamp`; disconnecting flips it to `false` within the presence-event propagation window.
4. `krabby agent` writes 1/min `{"state": {"reported": {...}}}` to the shadow; `iot:GetThingShadow` returns the latest payload conforming to the Task 2 schema (including `reported_image`).
5. Per-device isolation verified: a device's cert cannot publish/subscribe on another device's shadow, teleop signaling, or tunnel notify topics.
6. An AWS IoT Secure Tunnel opened from an operator laptop causes `krabby agent` to launch the destination local proxy against `localhost:22`, and SSH to the source port succeeds. Tunnel close makes both proxies exit. (Task-1 plumbing check; the CLI wrapper is Task 2.)
7. `enroll` + `agent` ship in the `krabby-launcher` package (published via the M14 workflow); the first-including version and `SETUP-FLEET.md` are documented.
8. **Automated E2E integration test** runs on **Bruce's bench Krabby** on every build (every PR + every merge) — the real bench Orin is the test target. The bench is permanently enrolled with a dedicated CI thing name (e.g. `bench-krabby-ci`) and stays online 24/7; CI drives it via the very control plane this milestone builds. Test covers, against a real dev-account IoT Core: (a) CDK deploys the stack into a scratch namespace (or reuses a persistent CI stack), (b) `iot:SearchIndex` on the bench thing returns `connectivity.connected: true` and a shadow `reported.timestamp` within the last N seconds, (c) `iot:GetThingShadow` returns a payload conforming to the Task 2 schema, (d) test writes a scratch shadow update using a **scratch second-device cert** and confirms the write is **rejected** by IoT Core (per-thing isolation — the second cert has no permission on the bench's shadow topic), (e) test calls Secure Tunneling `OpenTunnel` against the bench, asserts `krabby agent` on the bench spawns the destination `localproxy` (log/PID check via a follow-up short SSH), opens a raw TCP connection through the source proxy, and asserts bytes flow. Teardown deletes the scratch cert + closes the tunnel; the bench identity itself is preserved across runs. `pytest` + `boto3`; bench offline = CI red (the point).

## 4. Time Estimate (~4–5 days)

| Days | Sub-task title |
|------|----------------|
| 1 | CDK: IoT Core (thing type, per-thing policy, presence, Fleet Indexing on thing+connectivity+shadow) |
| 1.5 | `krabby enroll`: CSR + policy attach, install localproxy, write creds/endpoint/CA, verify connect |
| 1 | `krabby agent`: MQTT connect, 1/min shadow update, teleop signaling handoff, reconnect/systemd |
| 0.5 | Secure Tunneling wiring: notify-topic subscribe, spawn/kill destination `localproxy`; SSH sanity from a laptop |
| 1 | E2E integration test on the bench (connectivity + shadow + isolation + tunnel) + CI wiring |
| 0.5 | `SETUP-FLEET.md` + troubleshooting |

## 5. Dependencies

- **M14 (shipped):** `krabby-launcher` is the package we extend. `krabby install` already ships `krabby-locomotion.service`; the new `krabby-agent.service` installed by `krabby enroll` is a **separate** systemd unit alongside it (locomotion owns the app container; agent owns MQTT).
- **AWS:** IoT Core (things, policies, Classic Shadow, Fleet Indexing, Secure Tunneling), IAM in account krabby-company. J4012 (Jetpack 6/7), Docker, network to AWS.
- **Task 2** queries the shadow + connectivity via `iot:SearchIndex` / `iot:GetThingShadow` and calls Secure Tunneling `OpenTunnel`. **Task 3** uses the teleop signaling topics.

## 6. Deliverables

- `krabby enroll` + `krabby agent` in the `krabby-launcher` package (`krabby-research`), published via the M14 workflow.
- `ControlPlaneStack` CDK in `krabby-home/fleet/infra/` (IoT Core: thing type, per-thing policy, Fleet Indexing configuration, presence lifecycle events; Secure Tunneling is a service-level feature, no per-resource CDK). Exports `IotAtsEndpoint` for `FleetServiceStack` (Task 2).
- `SETUP-FLEET.md`.
