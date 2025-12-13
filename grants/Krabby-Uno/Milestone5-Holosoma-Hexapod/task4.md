# Task 4 - IsaacSim HAL model integration (Krabby hexapod)

Goal: make the trained hexapod policy runnable via the existing Isaac HAL server so it can be launched from the Krabby IsaacSim container with one command.

## What to touch
- `krabby-research` HAL config/model registry (wherever models are enumerated for IsaacSim runs).
- `hal/server/isaac/hal_server.py` wiring:
  - `_cache_references()` to bind to the hex robot entity.
  - `set_observation()` joint ordering/count and any camera/depth reads.
  - `apply_command()` action dimension/ordering into `action_manager`.

## Steps
1) Add a hexapod model entry in `krabby-research`:
   - Action dimension = 18, joint name/order matches the trained policy.
   - Paths: URDF/assets (copied/linked from the holosoma_ext data) and the trained model file (ONNX/ckpt).
   - Register in whatever launcher/CLI picks the model (mirroring existing entries).
2) Update Isaac HAL wiring as needed:
   - In `_cache_references()`, ensure `self.robot` resolves to the hex entity (name or lookup). Log the selected entity.
   - In `set_observation()`, map the 18 joint positions in the exact policy order; update camera/depth sourcing if the scene provides sensors (currently placeholders).
   - In `apply_command()`, confirm the incoming command vector is length 18; if action_manager expects batching, keep the tensor shape (1,18). Add a clamp/safety if desired.
3) Document the launch command:
   - Single CLI/Docker invocation that starts IsaacSim with the hex model and loads the trained policy.
   - Note expected artifacts (model path) and env vars/flags.
4) Validate:
   - From a fresh clone, run the documented command; robot spawns, model loads, joint commands flow, and the sim walks forward.
   - Capture a brief log/clip and note any sim requirements (IsaacSim version, GPU, etc.).

## Acceptance proof to capture
- Config snippet showing the hex model entry (paths + action_dim=18).
- Log line from `_cache_references()` confirming hex robot binding.
- Command used to launch, plus a short clip/screenshot or log of successful forward motion.
