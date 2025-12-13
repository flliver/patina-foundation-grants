# Patina Foundation Grant - Krabby-Uno Milestone 5: Holosoma Hexapod

## Grant Overview
Extend the `holosoma-extensions` packages (training and inference) to add the Krabby hexapod (URDF at `krabby-research/assets/crab_hex.urdf`), train a stable six-legged gait, and integrate the trained policy into `krabby-research` as a HAL model that runs in IsaacSim and on Jetson using the MCU joint I/O from the prior milestone. Holosoma is an open-source legged locomotion framework (training + inference) with a clean extension pattern (`holosoma_ext` and `holosoma_inference_ext`) already demonstrated on the Unitree Go2 quadruped. This milestone brings Krabby into that ecosystem so community members can train, deploy, and iterate on Krabby gait policies using the same tooling, while the `krabby-research` repo supplies the robot assets, HAL integration, and MCU bridge for real hardware.

## Why is this Important?
- Broadens holosoma’s open-source robot catalog beyond quadrupeds, making multi-legged locomotion more accessible to the community.
- Provides a reproducible pipeline from assets → training → inference → HAL integration, lowering barriers for others to add custom robots.
- Connects simulation to real hardware through the same open stack, enabling contributors to iterate on gaits and deploy them on affordable hexapod hardware.
- Delivers an open-source locomotion model specifically for the `krabby-research` hexapod, aligning the training/inference artifacts with the HAL/MCU stack so others can reproduce, benchmark, and improve Krabby’s gait end-to-end.

## Tasks
### Task 0 - Baseline holosoma quadruped bring-up (pre-req)
#### Narrative
Before adding Krabby, bring up the existing holosoma Go2 extension to validate environment, scripts, and simulator wiring. Train/eval the provided quadruped preset and confirm the policy walks in IsaacLab/IsaacSim as described in the holosoma_ext README.
#### Acceptance Criteria
- Follow `src/holosoma_ext/README.md` to install and run the Go2 preset; training script starts and runs without missing assets.
- In IsaacLab/IsaacSim, the trained (or provided) Go2 policy walks in sim using the documented run command; a short clip/log or screenshot is captured as proof.
- Environment notes recorded (conda/env name, simulator version, commands used) to reuse for the hexapod tasks.

### Task 1 - Add hexapod to holosoma_ext (training)
#### Narrative
Create a new branch of `holosoma-extensions` and follow `src/holosoma_ext/README.md` to register the Krabby hexapod for training: add robot configs, observations, rewards, experiment presets, and asset resolution pointing to `@holosoma_ext/` paths that include the Krabby URDF/meshes derived from `krabby-research/assets/crab_hex.urdf`.
#### Acceptance Criteria
- New branch in `holosoma-extensions` adds hexapod configs (robot/observation/reward/command/experiment) and assets under `src/holosoma_ext/data/robots/krabby_hex/`, using `@holosoma_ext/` path resolution.
- `pip install -e src/holosoma_ext` succeeds; `python -m holosoma_ext.train_agent exp:krabby-hex` (or equivalent preset) starts training without missing-asset/path errors.
- README snippet updated (or add a short section) showing the hexapod install/run commands mirroring the Go2 example.

### Task 2 - Add hexapod to holosoma_inference_ext (deployment)
#### Narrative
Mirror the training extension work in `src/holosoma_inference_ext`, following its README to register hexapod inference configs and assets so an exported model can run with the holosoma inference CLI/API.
#### Acceptance Criteria
- Hexapod inference configs added (robot, observation, inference preset) and assets under `src/holosoma_inference_ext/data/robots/krabby_hex/`, using `@holosoma_inference_ext/` paths.
- `pip install -e src/holosoma_inference_ext` works; `python -m holosoma_inference_ext.run_policy inference:krabby-hex` (or equivalent) runs in sim mode with no missing-path errors.
- README or notes updated to show the hexapod inference run command and model-path flag.

### Task 3 - Train and tune hexapod gait (six-leg reward/gait)
#### Narrative
Adapt the quadruped gait/reward setup to a six-legged gait (e.g., alternating tripods or other stable hex patterns). Train until the policy demonstrates stable forward walking in simulation with acceptable tracking and smoothness.
#### Acceptance Criteria
- Reward/command/gait configs updated for six legs; documented gait pattern (e.g., tripod phase offsets) checked into the branch.
- Training run completes and produces a usable policy artifact (e.g., ONNX) for the hexapod; include a short training log summary (episodes/returns) or metrics in the repo branch.
- Simulation demo (IsaacSim or MuJoCo) shows stable walking for at least N steps/time window without falls; a simple script/command is documented to reproduce the demo.

### Task 4 - IsaacSim HAL model integration in krabby-research
#### Narrative
Wire the trained hexapod policy into the Isaac HAL server so it is selectable and runnable from the Krabby IsaacSim container. This includes adding a Krabby-hex model definition/config in `krabby-research` (action dimension 18), mapping observation reads and command writes in `hal/server/isaac/hal_server.py` to the hexapod robot entity, and documenting the exact launch command. See `hal/server/isaac/hal_server.py` for how `IsaacSimHalServer` builds `KrabbyHardwareObservations` (18 joint positions, camera/depth placeholders) and how it applies commands via `action_manager`.
#### Acceptance Criteria
- New hexapod HAL model entry/config checked into `krabby-research` (paths to URDF/assets, model weights, action_dim=18) and registered so the Isaac HAL server can select it.
- `hal/server/isaac/hal_server.py` (or adjacent config) is updated only as needed to: (1) point `robot` lookup to the hex entity; (2) ensure joint ordering/count matches the trained policy (18 DOF); (3) optionally source camera/depth from the sim scene. Any helper wiring is documented.
- From a fresh clone, a single documented command starts IsaacSim with the hex HAL model, loads the trained policy, and drives the sim robot; validation clip/log captured showing forward locomotion with no missing-asset/path errors.

### Task 5 - Jetson/MCU deployment path
#### Narrative
Extend the integration to Jetson using the MCU joint I/O from the prior milestone. Add a hexapod model entry/config to the Jetson HAL path and wire the 18-DOF command/telemetry through the MCU bridge. Update `hal/server/jetson/hal_server.py` (or helper layer) to set `_action_dim = 18`, push joint targets to the MCU transport, and consume MCU feedback into `KrabbyHardwareObservations`. Provide a mock path for bench testing without motors.
#### Acceptance Criteria
- Jetson HAL config in `krabby-research` includes a hexapod entry (model paths, interface settings, action_dim=18) and can be launched with a documented Docker/venv command.
- `hal/server/jetson/hal_server.py` (or an adaptor) is updated to source joint state from MCU feedback (or a documented mock) and to send 18-joint targets to the MCU layer; logging shows timestamps and joint vectors as in the existing placeholder.
- End-to-end Jetson run completes: policy produces joint commands, HAL forwards to MCU (real or mock), telemetry returns, and a bench test log/clip is captured. Behavior without hardware is documented (mock transport) so the loop is testable.

## Information
- Repositories: `holosoma-extensions` (training/inference branches), `krabby-research` (assets, HAL integration).
- Assets: hexapod URDF at `krabby-research/assets/crab_hex.urdf`; copy/convert meshes as needed into extension data dirs.
  - This URDR contains a cyclical linkage for the knee joint, which is not supported in URDF, and will require manually linking the linear actuator to the tip of the knee in code, likely somewhere in the holosoma URDF loading code. Feel free to check w/ Holosoma discord if you/AI can't figure out how to add that linkage.
- Setup references: follow `src/holosoma_ext/README.md` and `src/holosoma_inference_ext/README.md` for install/run patterns.
- Simulators: IsaacGym/IsaacSim for training/sim; MuJoCo optional for inference checks.
- Models: store trained artifacts alongside the extension branch and reference them in `krabby-research` HAL config.

## FAQ
- Which branch?  
  Create and work on a new feature branch in `holosoma-extensions` for hexapod support; reference it in docs.
- How are assets resolved?  
  Use `@holosoma_ext/` and `@holosoma_inference_ext/` paths; ensure `patch_asset_path_resolution` (training) and inference path patching pick up the new assets.
- What gait to start with?  
  Begin with a tripod gait (two tripods alternating) or another stable hex pattern; document phase offsets and duty factors.
- How to validate quickly?  
  Provide a minimal sim run command for training and inference presets, plus a HAL launch command in `krabby-research` that shows the policy moving the sim robot.
