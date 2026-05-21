# Task 3 – Teleop Integration (over the shared MQTT channel, 2 robots)

## 1. Narrative

Integrate M7 WebRTC teleop into the fleet portal, reusing the **same per-device MQTT channel** the fleet uses for everything else. From device detail, "Teleop" starts a session (live streams + control). The browser exchanges WebRTC signaling (SDP/ICE) with the robot **through AWS IoT Core**: the fleet service bridges the browser to the robot's `teleop/{thingName}/signaling/in|out` topics; the robot's `krabby agent` hands signaling to the M7 WebRTC edge agent. Once connected, **media and control are peer-to-peer** over WebRTC (STUN, with coturn as TURN fallback) — no media touches MQTT. This is the unification: one internet mechanism per krab, no separate teleop transport.

Reuse the M7 agent and protocol (data channel `krabby-control-v1`, `InputController` command struct). Demonstrate two concurrent sessions (two robots from one portal). M7 remains the source of truth for the streaming and control protocol; M10 only changes the signaling **transport** to MQTT and integrates the UI into the portal.

## 2. Scope

- **Entry:** "Teleop" on the device list or detail opens the teleop view for that device (new tab).
- **Streams and pose:** at least one camera stream (M7 sensor interface); optionally multiple (front ZED, side, radar). Robot pose (HAL summary) displayed with streams. All over WebRTC; no live streams when the session is closed.
- **Control:** Browser Gamepad API and on-page virtual joystick/keyboard; commands on the WebRTC data channel at 50+ Hz, same struct as M7 `InputController` (leg, hip/knee/yaw).
- **Signaling over MQTT (the one mechanism):** the fleet service exposes a session endpoint (WebSocket/HTTP) to the browser, authenticates via Cognito, and bridges SDP/ICE to/from `teleop/{thingName}/signaling/in` and `.../out`. The robot is already connected to IoT Core, so it receives signaling with no inbound ports. Device credentials stay server-side. Document the message shape on the topics.
- **STUN/TURN:** use a public STUN server (e.g. `stun.l.google.com:19302`) for the common case; run **coturn on the same EC2** as the fleet service for the hard-NAT/symmetric-NAT fallback (UDP 3478, optionally TURNS 5349). Document the ICE server list and security-group rules.
- **Dual session:** two tabs; operator drives two robots from the same portal. Document steps to reproduce.
- **Mock:** [mocks/mock-teleop-dashboard](mocks/mock-teleop-dashboard).

## 3. Acceptance Criteria

1. "Teleop" per device opens a session with streams and control; signaling goes over MQTT (`teleop/{thingName}/signaling/in|out`) bridged by the fleet service.
2. At least one camera stream and robot pose visible; multiple streams as in M7.
3. Bluetooth/USB gamepad and in-browser control both work; commands sent at 50+ Hz on the data channel (M7 protocol).
4. Two concurrent teleop sessions supported; 2-robot demo documented.
5. Media/control are peer-to-peer (no media over MQTT); STUN works for the common case and coturn (on the EC2) is wired as the TURN fallback. Latency and connectivity documented (target <100 ms where feasible).
6. Implementation reuses the M7 WebRTC agent and protocol; only the signaling transport (MQTT) and portal integration are new.

## 4. Time Estimate (~5–7 days)

| Days | Sub-task title |
|------|----------------|
| 1 | "Open teleop" entry point; teleop view; fleet service session endpoint |
| 1.5 | Signaling bridge: browser ↔ fleet service ↔ IoT `teleop/{thing}/signaling/*`; robot agent → M7 edge agent |
| 1 | WebRTC connection; at least one video stream; pose in view |
| 1 | Input (Gamepad API + virtual joystick/keyboard), 50+ Hz commands; dual-robot UI + demo |
| 0.5 | coturn on the EC2; ICE config; latency/connectivity + signaling docs |

## 5. Dependencies

- **Task 1:** teleop signaling topics + per-device policy; `krabby agent` carries signaling to the M7 edge agent.
- **Task 2:** portal, fleet service, and the shared EC2 (hosts coturn) exist; teleop entry added to device detail/list.
- **M7:** WebRTC edge agent on the robot and the streaming/control protocol (reuse as-is; the agent gains an MQTT signaling source alongside its existing WebSocket one).

## 6. Deliverables

- Teleop UI in `krabby-home/fleet/portal/`; 2-robot support.
- Signaling bridge in the fleet service; robot-side MQTT→M7-agent handoff.
- coturn on the EC2; ICE config documented.
- Docs: MQTT signaling path + message shape, STUN/coturn setup, protocol (M7 ref), latency/connectivity, 2-robot demo steps.

## 7. FYI – Why signaling rides MQTT (not a second channel)

These notes are for implementers; they don't add acceptance criteria.

- **The robot is behind NAT** and can't be reached directly by the browser. It is, however, already holding an outbound MQTT connection to IoT Core (Task 1). So the cleanest place to deliver signaling is that existing connection — no second server, no second outbound channel, no extra NAT story. This is exactly the "one mechanism" goal.
- **Only signaling goes over MQTT** (SDP offer/answer, ICE candidates) — small JSON. **Media and the 50 Hz control data channel are peer-to-peer** over WebRTC, because MQTT/cloud relay would add unacceptable latency and bandwidth cost.
- **STUN** lets each peer discover its public IP:port for a direct path; it's stateless and free (public server). **TURN (coturn)** relays media only when a direct path can't be formed (symmetric NAT, strict firewalls). Co-locating coturn on the fleet EC2 keeps the always-on footprint to one box; revisit a dedicated TURN host only if relayed media bandwidth becomes significant.
- **M7's standalone WebSocket portal** (`teleop/portal`) still works for LAN/direct/dev use; M10 doesn't delete it. For the fleet/internet path, the signaling source is MQTT.
