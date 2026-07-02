# Task 3 – Teleop Integration (multi-robot picker + MQTT signaling + STUN/TURN)

## 1. Narrative

M7 already delivered the entire teleop stack — WebRTC edge agent on the robot, GStreamer sensor pipelines, `WebRTCInputController`, browser gamepad UI, the `krabby-control-v1` data channel protocol, all of it. What M7 doesn't have is a way to **pick which robot** to connect to, a way to run over the internet without configuring per-robot signaling servers, or a decided STUN/TURN story. Task 3 adds those three things and folds M7's teleop widget into the fleet portal.

## 2. What already exists (from M7 — do not rebuild)

| Component | Where it lives | M7 task |
|-----------|----------------|---------|
| WebRTC edge agent on the robot (H.264 via nvenc, dynamic stream request/release) | `krabby-research/teleop/edge` | M7 Task 4 |
| GStreamer sensor interface (`list_sensors` / `get_gstreamer_handle` / `build_pipeline`) driving all streams | `krabby-research/hal/` | M7 Task 2 |
| `krabby-control-v1` data channel protocol + `WebRTCInputController` on the robot | `krabby-research/teleop/edge` | M7 Task 5 |
| Browser WebRTC peer, video views, browser Gamepad API + virtual joystick UI, 50+ Hz command send | M7 test web app (`krabby-research/teleop/portal`) | M7 Tasks 4 + 5 |
| SDP/ICE handshake and signaling protocol shape | M7 test web app + edge agent | M7 Task 4 |

**Do not reimplement any of the above.** M10's job is to wrap/route them, not to touch WebRTC media, control, or UI internals.

## 3. Scope (what Task 3 actually adds)

- **Device picker in the portal:** pull the enrolled thing list from the Task 2 fleet-service `GET /devices` endpoint. Each row has an "Open teleop" action; clicking it opens the teleop view for that `thingName` in a new tab. `krabby-fleet teleop <robot>` launches the browser to the same URL. This is the only way M7 becomes multi-robot.

- **Signaling transport swap — M7 WebSocket → MQTT via the fleet service:**
  - Browser side: keep M7's browser peer as-is, but point its signaling source at a Cognito-authenticated fleet-service WebSocket endpoint (`/devices/{thingName}/teleop/signaling`) instead of M7's direct-to-robot WebSocket. The message shape (SDP offer/answer, ICE candidates) does not change.
  - Fleet service: bridge that WebSocket to/from the IoT topics `teleop/{thingName}/signaling/in|out`.
  - Robot side: `krabby agent` (Task 1) already receives on `teleop/{thingName}/signaling/in` and publishes on `.../out`; add a thin shim so it hands signaling to the existing M7 edge agent — either by writing to the same local IPC/ZMQ endpoint M7's WebSocket handler already writes to, or by having the M7 edge agent subscribe directly. Implementer picks whichever is one file's worth of code.

- **ICE server config (STUN + TURN) injected into the browser peer:** the fleet-service teleop endpoint returns an ICE server list to the browser: public STUN (`stun.l.google.com:19302`) and coturn on the same EC2 as the fleet service (UDP 3478, optional TURNS 5349), with short-lived TURN credentials minted per session. Deploy coturn on the EC2 and document the security-group rules.

- **Portal integration:** lift M7's `teleop/portal` React components into a Next.js route under the portal (or embed as-is if the shape allows), Cognito-gated. The only functional change is the signaling source URL + the ICE list — everything else (video views, gamepad handler, InputController wiring) is M7's code unchanged.

- **Mock:** [mocks/mock-teleop-dashboard](mocks/mock-teleop-dashboard).

## 4. Acceptance Criteria

1. Portal device list has "Open teleop" per row; clicking opens the M7 teleop view against that specific `thingName`. `krabby-fleet teleop <robot>` opens the same view.
2. Signaling round-trips over MQTT (`teleop/{thingName}/signaling/in|out`) via the fleet-service WebSocket bridge; no direct WebSocket to the robot.
3. WebRTC media + control still work end-to-end using **M7's existing edge agent, browser peer, gamepad UI, and protocol** — unchanged. Live video visible, gamepad drives at 50+ Hz, latency comparable to M7's local numbers.
4. ICE server list (STUN + coturn on EC2) is served by the fleet-service teleop endpoint; TURN relay verified by forcing a relay-only ICE policy in the browser and confirming the session still connects.
5. Two concurrent teleop sessions supported over the public internet against the two named robots from the OVERVIEW. Steps to reproduce documented, and no extra steps required other than installing krabby on the orin (which auto-registers w/ the fleet manager) and running the CLI or clicking connect on the fleet manager UI.
6. **Automated E2E integration test** runs on **Bruce's bench Krabby** on every build (every PR + every merge) against a real dev-account fleet service + IoT Core — the bench runs the real M7 edge agent + `krabby agent`, no fake device. Test covers, via Playwright: (a) Cognito auth, then open the portal teleop URL for `bench-krabby-ci`, and assert the signaling handshake completes over `teleop/{thing}/signaling/*` within N seconds, (b) assert the `krabby-control-v1` data channel opens and a scripted `InputController` message round-trips (browser → data channel → real M7 edge agent → HAL ack), using a **motion-safe payload** (e.g. zero-magnitude command, or a channel the bench treats as no-op) so the physical robot doesn't move during CI, (c) assert at least one camera track is received from the bench's real ZED, (d) **negative** — an unauthenticated session request is rejected before any MQTT signaling is emitted. Teardown closes session and asserts `teleop/{thing}/signaling/*` idles. Bench offline = CI red.

## 5. Time Estimate (~4–5 days)

| Days | Sub-task title |
|------|----------------|
| 0.5 | Device picker + "Open teleop" per-row in portal; `krabby-fleet teleop` opens the URL |
| 1 | Fleet-service signaling WebSocket bridge ↔ IoT `teleop/{thing}/signaling/*`; robot-side shim in `krabby agent` handing off to M7 edge agent |
| 0.5 | coturn on the EC2; short-lived TURN cred minting; ICE server list served from fleet-service endpoint; force-relay verification |
| 0.5 | Portal integration: M7 `teleop/portal` components → Next.js route, Cognito-gated; wire signaling URL + ICE list |
| 1 | E2E integration test on bench Krabby (Playwright + motion-safe control round-trip + dual session + auth negative) + CI wiring |
| 0.5 | Docs: signaling message shape on MQTT, coturn setup, 2-robot demo steps |

## 6. Dependencies

- **Task 1:** teleop signaling topics + per-device policy; `krabby agent` receives on `teleop/{thingName}/signaling/in` and publishes on `.../out`.
- **Task 2:** fleet-service REST + Cognito + EC2 exist; `krabby-fleet` CLI can add `teleop <robot>` as a browser launcher.
- **M7 (shipped):** WebRTC edge agent, `krabby-control-v1` data channel, `WebRTCInputController`, browser peer, gamepad UI, GStreamer sensor pipelines. Reused as-is; the edge agent gains an MQTT signaling source alongside its existing WebSocket one.

## 7. Deliverables

- Device picker + "Open teleop" in the portal (`krabby-home/fleet/portal/`).
- Signaling WebSocket bridge in the fleet service (`krabby-home/fleet/service/`); robot-side MQTT→M7-agent shim in `krabby agent`.
- coturn on the EC2 + short-lived TURN cred minting; ICE server list served by the fleet service.
- Docs: MQTT signaling message shape, coturn setup, 2-robot demo steps, force-relay verification recipe.
