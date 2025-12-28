# Patina Foundation Grant - Krabby-Uno Milestone 6: Joystick-Controlled Krabby

## Grant Overview
Build a joystick-driven control stack that lets a user drive Krabby in simulation and on hardware through a unified ControlLoop + HAL pipeline. Deliver a reusable InputController singleton for gamepad input, HAL mappers for both IsaacSim and real MCU backends, and package everything as pip-installable CLI tools (`krabby-uno`, `krabby-uno-sim`) with GitHub-based auto-publish to PyPI.

## Why is this Important?
- Enables manual teleop end-to-end: Bluetooth/USB gamepad → InputController → HAL → IsaacSim/MCU.
- Establishes clean controller/mapper/SDK boundaries so new controllers or robots only need new mappers/SDKs, not rewrites.
- Ships reproducible install/run via PyPI and GitHub Actions, simplifying bring-up on desktops and Jetson Orin.
- Aligns HAL and simulation with real hardware to shorten the loop from sim to field testing.

## Tasks
### Task 1 - InputController singleton (gamepad ingestion)
#### Narrative
Add a new `krabby-research/controller/input` package with an `InputController` singleton that reads Bluetooth/USB gamepad events (via `inputs` python library) and normalizes them into a controller state struct (buttons, sticks, triggers). Include the gait-style leg selection and axis mapping described in Directions (LT/LB/LS/RS leg selects, combo triggers, hip/knee/yaw axes). Provide a simple CLI to list controllers and emit live controller-state events for debugging.
#### Acceptance Criteria
- `InputController` lives under `krabby-research/controller/input/`, is a singleton with start/stop/thread-safe event loop, and exposes a typed state struct (buttons/sticks/triggers).
- Supports Bluetooth and USB gamepads on Ubuntu 22.04 (desktop and Jetson) with documented pairing steps.
- CLI: `python -m krabby_research.controller.input --list` shows devices; `--monitor <device>` streams normalized states at ~50–100 Hz to debug log (at speed of InputController game loop via cmdline arg based config)
- Implements the specified leg-selection logic and axis mappings; emits selected legs + hip/knee/yaw values as part of the state. (see example code below)

### Task 2 - ControlLoop wiring + HAL mappers (sim path)
#### Narrative
Implement `controller/control_loop.py` that wires singleton components based on a simple in-code config (at least two modes: InputController → HALClient → IsaacHalServer, and ModelController → HALClient → KrabbyHalServer/MCU). Implement `GamepadToIsaacSimHALMapper` to map controller state into the IsaacSim command struct (normalized joint targets/speeds). Create a proper `IsaacSimMCUSDK` interface used by `hal/server/isaac/hal_server.py` so `apply_command()` hands commands to a standardized SDK wrapper.
#### Acceptance Criteria
- ControlLoop config instantiates and starts InputController, HALClient, mapper, and SDK with clean thread lifecycle.
- `GamepadToIsaacSimHALMapper` maps leg selections and axes to per-joint commands compatible with IsaacSim HAL server; documented defaults and scaling.
- IsaacSim HAL server updated to call a standardized `IsaacSimMCUSDK`/interface instead of ad hoc code; apply_command uses mapper output directly.
- End-to-end sim path: with a paired gamepad, commands flow InputController → mapper → HALClient → IsaacSim HAL server; IsaacSim receives joint targets (verified via logs/telemetry).
- Note: Does not actually have to control the robot at this point, just have to be able to launch the ControlLoop, push buttons on the controller, and see them get sent through the hal+mapper and show up in the IsaacSimMCUSDK log code in Isaac's preferred joint format (instead of as gamepad command structure).

### Task 3 - Packaging & auto-publish (sim + real bundles)
#### Narrative
Package the control stack as pip-installable artifacts and set up CI to auto-publish on tags. Provide two entrypoints: `krabby-uno-sim` (InputController + IsaacSim HAL client/server path) and `krabby-uno` (real HAL path). Use `pyproject.toml` patterns from existing HAL packages; add GitHub Actions to build and publish to PyPI with provided token. Include basic regression tests in continuous integration.
#### Acceptance Criteria
- `pyproject.toml` updated/added for control/hal bundles so `pip install .` (editable) exposes `krabby-uno` and `krabby-uno-sim` CLIs.
- GitHub Action workflow builds sdist/wheel and publishes to PyPI on `v*` tags using `PYPI_API_TOKEN`; versions start at `0.1.0` and bump per release.
- CI runs lint/tests for InputController/HAL before publish to prevent bad releases.
- README.md for krabby-research is created/updated to show working install/run examples: `pip install krabby-uno krabby-uno-sim`, `krabby-uno-sim &`, `krabby-uno --InputController <id> --sim <hal_id>`.
- Note: Again, does not yet move robot in simulation, just shows that you can install it on Ubuntu w/ a straight pip install, connect a controller, launch, and see the joint commands flowing all the way through. 

### Task 4 - End-to-end joystick → IsaacSim demo
#### Narrative
Make the base case work: from a fresh desktop, pair a Bluetooth gamepad, run `krabby-uno-sim`, and drive the simulated Krabby joints in IsaacSim via HAL. Implement and validate `GamepadToIsaacSimHALMapper` axis scaling so small stick motions move joints slowly and full deflection moves at max joint speed. Provide runbooks for pairing the controller and launching the stack.
#### Acceptance Criteria
- From fresh clone/install, commands: `pip install krabby-uno krabby-uno-sim`; start IsaacSim HAL server; run `krabby-uno-sim --InputController <id> --sim <hal_id>`; joystick motion moves simulated joints visibly.
- HAL telemetry/logs confirm joint commands arrive at expected rates (should be at 50Hz+); no dropped commands in a 30s run.
- Controller pairing steps (Ubuntu) documented; mapper defaults align with the leg-selection + axis mapping in Directions; right/left combos and single-leg selects behave as specified.
- Debug scripts/logging to visualize controller state and mapper output during the demo, following existing work and best practie for INFO/WARN/DEBUG logging

### Task 5 - Real HAL path (gamepad → Krabby HAL/MCU)
#### Narrative
Implement `GamepadToKrabbyHALMapper` and validate the end-to-end loop on the real robot: InputController on the Orin (or tethered laptop) → HALClient → Krabby HAL Server → KrabbyMCUSDK/MCU. Ensure packaging works on Jetson (ARM) via PyPI. Document hardware steps (USB Bluetooth dongle/drivers on Orin) and provide a test that exercises the real HAL path without IsaacSim.
#### Acceptance Criteria
- `GamepadToKrabbyHALMapper` maps controller state to the real HAL command struct; documented scaling and defaults.
- Real path runs with `krabby-uno --InputController <id> --sim <hal_id>` (or equivalent) against the Krabby HAL server; commands reach MCU (or a mock if hardware absent).
- Packaging works on Jetson (ARM) via PyPI install; no code changes required for architecture.
- Hardware steps documented: install USB Bluetooth, pair controller on Orin, run the CLI, observe commands/telemetry; 10s stable run without crashes.
- Run it on a real robot and share a video to the team of the robot 'walking' around!!

## Information
- Sources: `krabby-research/hal`, `krabby-research/controller` (new), `krabby-research/firmware` (MCUSDK), IsaacSim HAL server, and HAL client.
- Reference contracts: see `krabby-contracts/milestones/M3/*` for HAL/game loop contract expectations.
- Gamepad mapping: follow Directions leg-selection and axis mapping; use `inputs/evdev` or similar to read events.
- Packaging: follow existing HAL package patterns; auto-publish via GitHub Actions with provided PyPI token; versions start at `0.1.0` and increment per release.
- HAL placement: keep IsaacSim HAL plumbing in `hal/server/isaac/hal_server.py` (e.g., `apply_command()` should call a standardized `IsaacSimMCUSDK`), and client wiring in `hal/client/client.py` (`send_commands`/`receive_observations`). New controller code lives under `krabby-research/controller` (e.g., `controller/control_loop.py`, `controller/input`).

## FAQ
- **How do I pick controllers/modes?** Use ControlLoop config flags to choose InputController + HAL server pairing (sim vs real). CLI examples in docs.
- **Do I need ROS2?** No. Stick to the existing HAL architecture and mappers; keep business logic in controllers and mappers only.
- **What if I don’t have hardware?** Use the IsaacSim path (`krabby-uno-sim`) to validate; real path can run against a mock HAL if MCU unavailable.

## Appendix A - Example Gamepad Polling Snippet
```python
from inputs import get_gamepad

# Button/axis state
state = {
    "LT": False,
    "LB": False,
    "LS": False,
    "RS": False,
    "RT": False,
    "RB": False,
    "LX": 0.0,
    "LY": 0.0,
    "RX": 0.0,
    "RY": 0.0,
}

def update_state(event):
    code = event.code
    val = event.state
    if code == "ABS_Z":      # Left Trigger analog
        state["LT"] = (val > 10)
    elif code == "ABS_RZ":   # Right Trigger analog
        state["RT"] = (val > 10)
    elif code == "BTN_TL":   # Left Button (LB)
        state["LB"] = (val == 1)
    elif code == "BTN_TR":   # Right Button (RB)
        state["RB"] = (val == 1)
    elif code == "BTN_THUMBL":
        state["LS"] = (val == 1)
    elif code == "BTN_THUMBR":
        state["RS"] = (val == 1)
    elif code == "ABS_X":
        state["LX"] = val
    elif code == "ABS_Y":
        state["LY"] = val
    elif code == "ABS_RX":
        state["RX"] = val
    elif code == "ABS_RY":
        state["RY"] = val

def process_controls():
    select_FL = state["LT"] and not state["LB"]
    select_RL = state["LB"] and not state["LT"]
    select_ML = state["LS"]
    select_MR = state["RS"]
    select_FR = state["RT"] and not state["RB"]
    select_RR = state["RB"] and not state["RT"]
    combo_left = state["LT"] and state["LB"]   # FL/RL/MR
    combo_right = state["RT"] and state["RB"]  # FR/RR/ML

    legs = set()
    if combo_left:
        legs |= {"FL", "RL", "MR"}
    if combo_right:
        legs |= {"FR", "RR", "ML"}
    if not combo_left and not combo_right:
        if select_FL: legs.add("FL")
        if select_RL: legs.add("RL")
        if select_ML: legs.add("ML")
        if select_MR: legs.add("MR")
        if select_FR: legs.add("FR")
        if select_RR: legs.add("RR")

    if legs:
        hip_up_down = -state["LY"]
        knee_out_in = state["LX"]
        hip_yaw = state["RY"]
        # TODO: Put HALClient code here, build a proper GamepadControlData struct and pass it to Hal.
        print(f"Active Legs: {legs}")
        print(f"  Hip up/down: {hip_up_down:.3f}")
        print(f"  Knee out/in: {knee_out_in:.3f}")
        print(f"  Hip yaw: {hip_yaw:.3f}")

while True:
    for evt in get_gamepad():
        update_state(evt)
    process_controls()
```

## Appendix B - Gamepad Input Mapping
| Control | Selection/Action | Effect |
| --- | --- | --- |
| LT | Select Front Left (FL) unless LB held | Leg selection |
| LB | Select Rear Left (RL) unless LT held | Leg selection |
| LS button | Select Left Middle (ML) | Leg selection |
| RS button | Select Right Middle (MR) | Leg selection |
| RT | Select Front Right (FR) unless RB held | Leg selection |
| RB | Select Rear Right (RR) unless RT held | Leg selection |
| LT + LB | Select FL, RL, MR together | Tripod combo (left set) |
| RT + RB | Select FR, RR, ML together | Tripod combo (right set) |
| Left stick Y | Hip up/down for selected legs | Signed; invert per mapper config |
| Left stick X | Knee out/in for selected legs | Signed normalized axis |
| Right stick Y | Hip yaw forward/back for selected legs | Signed normalized axis |
