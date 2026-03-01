# Task 0 — Prerequisites and baseline validation

Goal: Run an off-the-shelf NVIDIA/Isaac Lab hexapod training example and validate the existing x86 parkour simulation (Unitree Go2). Document prerequisite hardware and exact steps.

Outputs
- Short doc or README section: commands, env notes, hardware spec
- Optional: short video/screenshots of (0a) hexapod example run and (0b) Go2 gap jump in sim

Acceptance Criteria
- **0a** – Hexapod example: repo, command, and env documented; run completes (short clip/screenshot optional).
- **0b** – Parkour: build image, run sample (e.g. `play.py` / evaluation script), Unitree Go2 model jumps gaps in simulation as intended; exact steps documented.
- **Hardware** – Prerequisite hardware documented: **16 GB NVIDIA RTX 5080**, PC with **32 GB RAM**, **Core i7** (or equivalent).

---

**NOTE**: All code and commands in this document are guidance. Actual paths and APIs may vary; document the exact steps that work in your environment.

---

## 1. Prerequisite hardware

Document and use the following (or equivalent) for training and simulation in later tasks:

| Component | Specification |
|-----------|---------------|
| GPU | 16 GB NVIDIA RTX 5080 |
| RAM | 32 GB |
| CPU | Core i7 or equivalent |

Record the exact hardware used (and any driver/CUDA versions) in the Task 0 deliverable so later tasks can reproduce.

---

## 2. Task 0a – NVIDIA hexapod training example

### Narrative
Find and run an off-the-shelf NVIDIA (or Isaac Lab) example for hexapod training with reinforcement learning, ideally using a legged-gym–style or Isaac Lab legged setup. Goal: confirm the toolchain and that a six-legged agent can be trained in the same stack before touching the crab.

### Steps
1. Identify an example: e.g. Isaac Lab legged-robot examples, or community hexapod adaptations (e.g. legged_gym–style configs migrated to Isaac Lab).
2. Install and run the example (document conda/env, simulator version, and exact command).
3. Confirm training starts and runs (or a provided policy runs in sim). Capture a short clip or screenshot as proof.
4. Record environment notes (env name, simulator version, commands) for reuse in Tasks 1–5.

### Acceptance
- Example repo/source and run command documented.
- Run completes; optional short clip/screenshot.
- Environment notes sufficient to reuse for hexapod tasks.

---

## 3. Task 0b – Validate Milestone 3 x86 parkour simulation

### Narrative
Validate that the existing Unitree Go2 parkour setup runs on x86: build the image, run the sample, and show the Go2 model jumping gaps in simulation. No new features—reproduce existing behavior.

### Steps
1. Follow the parkour stack docs in `krabby-research/parkour` to build the image (or set up the environment) for x86.
2. Run the sample (e.g. `play.py` or the evaluation script used for parkour).
3. Confirm the Unitree Go2 model jumps gaps in simulation as designed.
4. Document the exact build and run steps (commands, paths, any env vars) so anyone can reproduce.

### Acceptance
- Build and run steps documented.
- Go2 gap-jump behavior reproduced in sim (optional short clip/screenshot).
- No code changes required; validation only.

---

## 4. Deliverable checklist

- [ ] Hardware spec (RTX 5080, 32 GB RAM, Core i7) documented.
- [ ] Hexapod example: source, command, env notes; run completed.
- [ ] Parkour: build/run steps documented; Go2 gap jump reproduced in sim.
- [ ] Optional: video or screenshots for 0a and 0b.
