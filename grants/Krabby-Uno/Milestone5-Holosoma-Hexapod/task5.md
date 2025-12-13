# Task 5 - Jetson/MCU deployment path (Krabby hexapod)

Goal: run the trained hexapod policy on Jetson, pushing 18-DOF commands through the MCU stack and consuming joint feedback (real or mock) via the Jetson HAL server.

## What to touch
- Jetson HAL model/config in `krabby-research` (model path, interface settings, action_dim=18).
- `hal/server/jetson/hal_server.py` (or an adaptor):
  - `_action_dim` set to 18.
  - Command sink: forward joint targets to the MCU transport.
  - Telemetry source: pull joint state back from MCU (or mock) into `KrabbyHardwareObservations`.

## Steps
1) Add Jetson hex model entry/config:
   - Include model artifact path, network/transport settings to the MCU bridge, and action_dim=18.
   - Provide a Docker/venv launch command (mirror existing parkour/Go2 patterns if present).
2) Wire Jetson HAL to MCU:
   - Set `_action_dim = 18`.
   - In `apply_command()`, after `get_joint_command()`, send the 18-float vector to the MCU transport (real or mock). Keep logging timestamp + vector (as current code does).
   - In `set_observation()`, replace placeholder joint echo with MCU feedback; pad/trim to 18 in policy order. Fill cameras/depth if available; otherwise leave zeros with confidence map = 1.
3) Provide a mock path:
   - If hardware absent, include a mock MCU transport that echoes or ramps joint positions so the loop is testable on Jetson/x86 without motors. Document how to enable it (flag/env var).
4) Validate on Jetson:
   - Run the documented launch command; verify commands flow to MCU (or mock) and telemetry returns at target rate.
   - Capture logs showing timestamps + joint vectors; if hardware is connected, a brief bench clip is ideal.

## Acceptance proof to capture
- Config snippet for the Jetson hex model (paths, action_dim, transport settings).
- Code/notes showing `_action_dim=18`, MCU send/receive hooks in the Jetson HAL path.
- Launch command and logs from a successful run (mock or real), demonstrating bidirectional flow.
