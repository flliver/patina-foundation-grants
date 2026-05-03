# Task 3 – Teleop Integration (Portal + 2 Robots)

## 1. Narrative

Integrate M7 WebRTC teleop into the fleet portal: from device detail, “Teleop” starts a session (live streams + control). Live video and pose are sent only over WebRTC while the session is active. Reuse M7 agent and protocol (data channel, `InputController` command struct). Teleop UI shows robot pose plus camera feeds; input via Gamepad API (Bluetooth/USB) and in-browser virtual joystick or keyboard. Demonstrate two concurrent sessions (two robots from one portal). M7 OVERVIEW Tasks 4 and 5 remain the source of truth for streaming and control.

## 2. Scope

- **Entry:** “Teleop” on device list or detail opens the teleop view for that device in a new tab.
- **Streams and pose:** At least one camera stream (M7 sensor interface); optionally multiple (front ZED, side, radar). Robot pose (summary from HAL) displayed with streams. All over WebRTC; no live streams when session is closed.
- **Control:** Browser Gamepad API and virtual joystick/keyboard; commands on WebRTC data channel at 50+ Hz, same struct as M7 `InputController` (leg, hip/knee/yaw).
- **Dual session:** Two tabs; operator can drive two robots from the same page. Document steps to reproduce.
- **Signaling:** Via fleet service; device credentials server-side only. Document signaling path (fleet service + IoT topics to robot). Use a public or SaaS STUN server (e.g. `stun.l.google.com:19302` or your TURN provider’s); add TURN via a provider (e.g. Twilio, Xirsys) or self-hosted coturn (e.g. on EC2) only if direct P2P fails. Document STUN/TURN choice and latency/firewall notes.
- **Mock:** [mocks/mock-teleop-dashboard](mocks/mock-teleop-dashboard).

## 3. Acceptance Criteria

1. “Teleop” per device opens a session with streams and control.
2. At least one camera stream and robot pose visible; multiple streams as in M7.
3. Bluetooth/USB gamepad and in-browser control both work; commands sent at 50+ Hz on data channel (M7 protocol).
4. Two concurrent teleop sessions supported; 2-robot demo documented.
5. Control protocol matches M7; latency and connectivity documented (e.g. target &lt;100 ms where feasible). Signaling path (fleet service + IoT) and STUN (and TURN if used) documented.
6. Implementation follows M7 OVERVIEW Tasks 4 and 5 (streaming agent, `WebRTCInputController`).

## 4. Time Estimate (~5–7 days)

| Days | Sub-task title |
|------|----------------|
| 1 | “Open teleop” entry point; teleop view; signaling (fleet service) |
| 1 | WebRTC connection; at least one video stream |
| 1 | Pose in teleop view; input (Gamepad API + virtual joystick/keyboard), 50+ Hz commands |
| 1 | Dual-robot UI; 2-robot demo and validation |
| 0.5 | STUN (and TURN if needed); latency/connectivity and signaling docs |

## 5. Dependencies

- Task 2: portal and fleet service exist; teleop entry added to device detail/list. M7: WebRTC agent on robot; Tasks 4 and 5 complete.

## 6. Deliverables

- Teleop UI in `krabby-home/fleet/portal/`; 2-robot support.
- Docs: signaling path (fleet service + IoT), STUN/TURN setup, protocol (M7 ref), latency/connectivity, 2-robot demo steps.

## 7. FYI – Signaling and NAT (STUN/TURN)

These notes are for implementers; they don’t add new acceptance criteria.

### Signaling (how browser and robot exchange “how to connect”)

- **Problem:** The browser can reach the fleet service (HTTPS). The robot is behind NAT and can’t be reached directly by the browser. So the fleet service must deliver signaling (SDP offer/answer, ICE candidates) to the robot.
- **Flow:**
  - Browser opens a teleop session (e.g. WebSocket or HTTP to fleet service), sends “I want to talk to device X” and its SDP/ICE.
  - Fleet service authenticates, then publishes the browser’s SDP/ICE to an **AWS IoT topic** the robot subscribes to (e.g. `teleop/{thingName}/signaling/in`). The robot is already connected to IoT, so it receives the message.
  - Robot’s WebRTC agent generates its SDP/ICE and publishes to a topic the fleet service subscribes to (e.g. `teleop/{thingName}/signaling/out`). Fleet service forwards that to the browser over the same WebSocket/HTTP channel.
  - This exchange continues until both sides have enough ICE candidates. **No video or control data goes over IoT**—only signaling.
- **What to implement:** Fleet service endpoints (or WebSocket) for the portal; fleet service ↔ IoT (publish/subscribe) with a clear topic scheme; robot subscribes/publishes on those topics and passes payloads to the M7 WebRTC agent. Document the topic names and message shape in Task 3 docs.

### STUN (discover public IP:port for P2P)

- **What it does:** STUN lets each peer (browser and robot) ask a public server “what’s my public IP and port?” so they can try a direct peer-to-peer connection. It’s stateless and lightweight.
- **You don’t need to run your own.** Use a **public STUN server**, e.g. `stun.l.google.com:19302`. In your WebRTC config (browser and/or robot), set the ICE server list to include that URL. One line of config; no Fargate, no extra infra.
- **Other options:** Many TURN providers (Twilio, Xirsys, etc.) give you a STUN URL as well; you can use that if you’re already using them for TURN. Document which STUN URL(s) you use so operators know.

### TURN (relay when direct P2P fails)

- **When you need it:** Only when ICE can’t establish a direct path (e.g. symmetric NAT, strict corporate firewalls). Try with STUN only first; add TURN if connections fail in your target networks.
- **Easiest:** Use a **TURN SaaS** (Twilio, Xirsys, Metered.ca, etc.). Sign up, get TURN URLs and credentials, put them in your WebRTC ICE server config. Well documented; no servers to run.
- **Self-host:** **coturn** is the standard. Run it on a **small EC2** (or any VPS). It needs UDP (and often TCP) on port 3478 (TURN) and optionally 5349 (TURNS). Well documented; many tutorials. **Fargate** can run coturn in theory, but TURN uses lots of bandwidth (it relays media), needs UDP and fixed ports, and Fargate data transfer cost can add up—so EC2 is usually simpler and cheaper for TURN.
- **What to document:** Whether you use a provider or self-hosted coturn; the ICE server list (STUN + TURN URLs) you use; and any firewall/security-group rules (e.g. 3478 UDP/TCP) if self-hosting.

### Summary

| Piece | Recommendation | Notes |
|-------|----------------|-------|
| Signaling to robot | Fleet service + AWS IoT topics | Robot subscribes to `teleop/{thing}/signaling/in`; publishes to `.../out`. Fleet service bridges browser ↔ IoT. |
| STUN | Public server (e.g. `stun.l.google.com:19302`) or TURN provider’s STUN | No self-host needed. |
| TURN | SaaS first; or coturn on EC2 if self-hosting | Add only if direct P2P fails. Prefer SaaS; Fargate for TURN is possible but usually not worth it. |
