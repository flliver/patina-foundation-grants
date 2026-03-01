# Patina Foundation Grant - Krabby-Uno Milestone 2: Hexapod Training

## Grant Overview
Design, implement, and validate a novel hexapod "crab robot" embodiment within **NVIDIA Isaac Lab**. The **realistic Blender model** lives in `krabby-research/assets`; the goal is to import it as **USD** and treat the **USD as the authoritative copy** for training and evaluation. The existing `crab_hex.urdf` in `krabby-research/assets` is a **simple reference for initial tests/training** only. Convert the Blender model to USD with proper prismatic actuator constraints, configure policy and rewards for six legs, train teacher and student models with the existing Extreme Parkour pipeline, add expanded evaluation scenarios including real-world mesh–derived terrain (e.g. SAM 3D), and validate by driving the robot with a joystick on at least one generated 3D mesh in simulation.

## Why is this Important?
- Establishes a reproducible path from the Blender model in `krabby-research/assets` to authoritative USD, policy config, training, and evaluation for a six-legged platform.
- Aligns with Milestone 3 parkour runtime and HAL/depth pipeline so the crab can run on the same stack (sim and hardware).
- Delivers teacher/student models and 10+ diverse eval scenarios, including at least one from a 3D-from-images pipeline, to identify limitations and support sim-to-real iteration.
- Final joystick-on-mesh validation proves the full loop: generated terrain → sim → human-in-the-loop control.

## Tasks
Full acceptance criteria and implementation detail live in the task files in this folder (`Task0-Prerequisites.md` through `Task5-Expanded-Eval.md`). Summaries:

### Task 0 - Prerequisites and baseline validation
#### Narrative
Run an off-the-shelf NVIDIA/Isaac Lab hexapod training example (ideally legged-gym–style) and validate Milestone 3 x86 parkour simulation (Unitree Go2 gap jump). Document prerequisite hardware and exact commands.
#### Acceptance Criteria
- Hexapod example: repo, command, env documented; run completes (short clip/screenshot optional).
- Parkour: build image, run sample (e.g. `play.py` / evaluation), Go2 jumps gaps in sim; steps documented.
- Hardware spec documented: 16 GB NVIDIA RTX 5080, 32 GB RAM, Core i7 (or equivalent).

### Task 1 - Robot asset and USD conversion
#### Narrative
Import the realistic Blender model from `krabby-research/assets` as USD (USD is the authoritative copy). URDF in `krabby-research/assets` is a simple reference for initial tests/training only. Research and document Blender→USD path; preserve prismatic actuator constraints with concrete examples.
#### Acceptance Criteria
- `crab_hex.usd` (or named variant) with prismatic constraints; validation script (range of motion, collisions); doc (joint limits, mass/inertia, constraints). If Blender: export steps and file location documented.

### Task 2 - Policy configuration
#### Narrative
Define PPO and policy config for the six-legged crab. Ensure 6-leg observation/action and gait config; depth camera configurable and aligned with the parkour HAL/depth pipeline (M3). Double-check rewards for hexapod after Task 0 example.
#### Acceptance Criteria
- Policy config loads; RGBD/depth camera positioned and configurable; robot spawns and “flops” in training env; observation/action sized for hexapod; gait documented; depth alignment noted; pytest-style validation.

### Task 3 - Teacher model training
#### Narrative
Train teacher with crab through same curriculum as Unitree baseline. Run existing parkour pipeline; ensure good progress early in training.
#### Acceptance Criteria
- Teacher completes curriculum; gap navigation; stable, compact locomotion; weights/logs saved and loadable; publish/eval commands work; early-training sanity check documented.

### Task 4 - Student model training
#### Narrative
Train student with depth input; same pipeline as designed. Ensure good progress early; within 5% of Unitree on standard metrics.
#### Acceptance Criteria
- Student trains with depth; within 5% of Unitree baseline; weights loadable; comparison documented; early-training check documented.

### Task 5 - Expanded evaluation and final validation
#### Narrative
Add 10+ evaluation scenarios. Include backyard/real-world scenario generation (e.g. SAM 3D or similar: images → 3D mesh → USD). Final acceptance: joystick the robot in sim on at least one such mesh.
#### Acceptance Criteria
- 10+ scenarios; registry and `make eval`; at least one scenario from 3D-from-images pipeline; joystick-on-mesh demo documented with command and clip/log.

## Information
- Repositories: `krabby-research` (code, assets), `patina-foundation-grants` (this grant; task specs and acceptance criteria).
- Assets: Blender crab model and `crab_hex.urdf` in `krabby-research/assets`. Output: `krabby-research/assets/crab_hex.usd` (authoritative). URDF is a simple reference for initial tests/training only.
- References: Isaac Lab / Isaac Sim parkour stack in `krabby-research/parkour`; parkour HAL/depth pipeline (Milestone 3) for camera alignment.

## FAQ
- **Blender vs URDF?**  
  The Blender model in `krabby-research/assets` is the realistic source; the goal is to import it as USD—**the USD is the authoritative copy**. `crab_hex.urdf` in `krabby-research/assets` is a simple reference for initial tests/training only. Task 1 documents the Blender→USD path and export steps.
- **Hardware for Task 0?**  
  16 GB NVIDIA RTX 5080, 32 GB RAM, Core i7 (or equivalent). Document in Task 0 deliverable.
- **Depth camera and Milestone 7 work?**  
  Depth camera (resolution, FOV, placement) must be configurable and aligned with the HAL/depth pipeline used by the parkour runtime (M3); contractor sets exact values to match.
- **Where are full task details?**  
  In this folder: `Task0-Prerequisites.md`, `Task1-USD-Conversion.md`, `Task2-Policy-Config.md`, `Task3-Teacher-Training.md`, `Task4-Student-Training.md`, `Task5-Expanded-Eval.md`.
