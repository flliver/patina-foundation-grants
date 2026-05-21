# Patina Foundation Grant - Krabby-Uno Milestone X: Hexapod Sim-to-Real Deployment

## Grant Overview
Take the teacher and student parkour policies trained against `crab_hex.usd` in Milestone 2 and run them on real Krabby-Uno hardware. The work has four substantive pieces. First, integration-validate the existing parkour container image on the physical robot via `krabby-launcher`, with auto-start on boot, a live HAL link to the krab MCU so joint commands and telemetry flow between the policy and `MCUSDK`, and a Bluetooth-paired controller delivering inputs to the policy over HAL (each component has been built and tested individually in earlier milestones; this task pulls them together with the parkour policy in the loop, and the robot is expected to flail at this stage — the goal is the data path, not the gait). Second, give the policy a prismatic action space matching the real robot's lead-screw actuators rather than the rotational joints used during M2 training, by one of two routes — preferred: update `crab_hex.usd` with real prismatic joints coupled through the lever-arm geometry to the rotational joints (a closed kinematic loop, handled with the appropriate PhysX/USD primitives); fallback: keep the USD rotational and insert a sim-side mapping function that takes prismatic commands and converts them via the same lever-arm geometry — then retrain teacher and student. Third, close the depth observation loop: write a HAL function that takes the front-facing ZED's depth and downsamples/transforms it to the exact tensor format the M2 student model was trained on, deploy the retrained image with the depth pipeline live, and validate that the policy responds to real-world depth changes. Fourth, close the sim-to-real dynamics gap by adding domain randomization to training (modeled on the Extreme Parkour Unitree config) and tuning sim motor torque to measured real-robot values, then deploy the noise-trained model and produce the milestone deliverable: with the Bluetooth controller, forward drives the robot forward and backward drives it backward, with a gait reasonably matching sim under the same input.

## Why is this Important?
- First time a trained policy actually drives the real Krabby-Uno hexapod under controller input, closing the sim-to-real loop established in M2 with a measurable demo at the end.
- Validates the end-to-end data path (controller → policy → HAL → MCUSDK → motors/sensors → policy) on real hardware.
- Fixes the action-space mismatch between sim (rotational hip/knee joints) and real (prismatic lead-screw actuators); without this the policy emits commands the real robot cannot execute with the right dynamics, since prismatic velocity does not map linearly to joint angular velocity.
- Closes the dynamics gap by porting the Extreme Parkour Unitree domain-randomization approach to the hexapod, so the policy stays in-distribution on the real robot rather than out-of-distribution from the first sensor reading.
- Establishes the deployment workflow (image build → fleet portal / `krabby-launcher` → auto-start → HAL bring-up) that every future milestone with hardware in the loop will depend on.

## Tasks
Full acceptance criteria and implementation detail live in the task files in this folder (`Task1-Parkour-Image-Bringup.md` through `Task4-SimToReal-Tuning.md`). Summaries:

### Task 1 - Parkour image bring-up on real robot
#### Narrative
Pull together the existing pieces and validate them as a working system on the real krab with the M2 parkour image. The individual components below have all been built and tested in earlier milestones — this task is *integration validation*, not new development, and the surprises will be in the seams.

**Should already work from earlier milestones (re-validate, don't rebuild):**
- `krabby-launcher` deployment of a container image to the robot.
- Container auto-start on boot.
- HAL transport between an on-robot container and `MCUSDK`.
- `MCUSDK` driving all 18 actuators on joint commands and publishing telemetry (joint positions, currents, IMU).
- Bluetooth pairing of a controller to the on-robot compute and the controller's input surfacing via HAL.

**Needs integration validation in this milestone:**
- The M2 parkour image specifically (built from current `krabby-research` weights) running end-to-end on the robot through all of the above.
- Bidirectional flow specifically between the parkour policy and `MCUSDK` over HAL — policy joint commands reach the MCU, `MCUSDK` telemetry reaches the policy.
- Controller inputs flowing over HAL into the parkour policy as the command stream the policy was trained against.

The robot will spazz out at this stage — that is expected and not a failure. The success condition is the data path through every seam, not the gait.

#### Acceptance Criteria
- M2 parkour image builds and is deployable to the robot via `krabby-launcher`; deploy command documented.
- Container auto-starts on boot; verified by power-cycling the robot and observing the process come up unattended.
- HAL link between the parkour policy container and `MCUSDK` is live; verified with a HAL inspector or log capture.
- Joint commands from the parkour policy reach the MCU; observed motor motion (even if random) on all 18 actuators.
- Telemetry stream from `MCUSDK` back to the parkour policy is live; sample of joint positions, motor currents, and IMU captured in a log.
- Bluetooth controller pairs to the on-robot compute; controller inputs arrive at the parkour policy over HAL as the command stream the policy expects; verified by moving a stick and observing the corresponding change in command input to the model.

### Task 2 - Sim/real joint mapping and retrain
#### Narrative
Make the sim's action space match the real robot, then retrain. The sim trained in M2 uses rotational joints driven directly; the real robot drives those joints through prismatic lead-screw actuators acting on a lever arm, which gives a nonlinear relationship between actuator displacement and joint angle (fast when the actuator is perpendicular to the lever arm, slow near the extremes of travel). The policy needs to be trained against the prismatic action space so it learns the correct dynamics.

There are two acceptable approaches, in order of preference:

**Option 1 (preferred, more "real"): Real prismatic joints in USD.**
Update `crab_hex.usd` so each leg DOF is actually driven by a prismatic joint coupled to a rotational joint through the lever-arm geometry. This creates a closed kinematic loop, which USD/PhysX handles through specific joint primitives (D6, articulation root, mimic/coupled constraints) — not the default articulation tree topology, which is strictly a tree. There are guides on closed-loop articulations in PhysX/Isaac; this is the path if the contractor can get the physics stable.

**Option 2 (fallback): Sim-side prismatic→rotational mapping function.**
Leave `crab_hex.usd` with its rotational joints as-is. Insert a mapping function in the sim's actuation layer that accepts prismatic commands from the policy and converts them to the equivalent rotational joint command using the same lever-arm geometry the real robot has. The policy sees a prismatic action space; the USD physics still runs on rotational joints. Less "real" but avoids the closed-loop USD problem entirely.

Either way: document the lever-arm geometry, retrain teacher and student against the new action space, and validate by sending the same prismatic command to the real robot and the updated sim and confirming the leg moves the same amount.

Also in this task: mount the front-facing ZED depth camera on the real robot in a physically appropriate location (clear forward view, not obstructed by legs during gait), then move the depth camera in `crab_hex.usd` to match that mount location and orientation. This belongs here because it is the same kind of sim/real frame-alignment work as the joint mapping — the policy needs the sim camera pose to match the real camera pose so the depth observations it learned to act on are the depth observations it actually gets on hardware.
#### Acceptance Criteria
- Full audit of joint input/output transforms between sim (M2 `crab_hex.usd`) and real (`MCUSDK`/HAL) documented; every unit, sign convention, and frame called out.
- Lever-arm geometry documented for hip pitch, knee, and hip yaw on a representative leg (lever length, actuator anchor points, joint angle range, resulting nonlinear prismatic↔rotational relationship).
- One of:
  - **Option 1:** `crab_hex.usd` updated so each leg DOF is driven by a real prismatic joint coupled through the lever-arm geometry to a rotational joint; closed-loop articulation stable under the M2 training workload; brief writeup of which PhysX/USD primitives were used and any references followed.
  - **Option 2:** Sim-side prismatic→rotational mapping function implemented and unit-tested at the sim's actuation layer; policy sees prismatic action space; rotational joints in `crab_hex.usd` unchanged.
- Sim policy action space is prismatic (lead-screw displacement / velocity), matching the real robot's actuators.
- Command-parity test: identical prismatic command to one leg drives sim and real leg through the same displacement (within documented tolerance) across the joint range.
- ZED depth camera physically mounted on the real robot; mount location and orientation (position relative to body frame, pitch/yaw of optical axis) documented.
- Depth camera in `crab_hex.usd` repositioned to match the real ZED mount within documented tolerance.
- Teacher and student retrained on the updated sim; weights saved and loadable; metrics within the M2 ballpark (no major regression from the action-space change).

### Task 3 - Depth pipeline from ZED to deployed policy
#### Narrative
Get the depth observation path working end-to-end on hardware. The M2 parkour student model was trained on a specific depth tensor format (resolution, format, normalization, FOV) coming from the sim's depth camera. The real robot's front-facing ZED produces depth at its own native resolution and format, which is not what the policy expects. The substantive work is a HAL function that takes ZED depth and downsamples/transforms it into the exact tensor format the M2 student model was trained against, then publishes that as the policy's depth observation over HAL.

With the camera mount and sim camera matched from Task 2, and the policy retrained against the prismatic action space from Task 2, this task closes the loop: real ZED → HAL depth-conversion → policy → joint commands → `MCUSDK` → motion. Validate by deploying the retrained image with the depth pipeline live and confirming the policy reacts to depth changes in front of the robot (obstacle appears → policy command stream changes).
#### Acceptance Criteria
- Depth format the M2 student model was trained on documented exactly: resolution, channel layout, depth encoding (meters / normalized / inverse), FOV, any clipping or near/far planes.
- HAL depth-conversion function implemented: input is ZED native depth, output is the documented M2 student depth format; downsampling method (interpolation, cropping, FOV adjustment) documented.
- Bench test: a recorded ZED depth frame run through the HAL function produces a tensor that matches a sim-rendered depth frame of an equivalent scene within documented tolerance.
- New container image built with Task 2 weights and the Task 3 depth pipeline; deployed to the real krab via fleet portal if available, otherwise `krabby-launcher`.
- On the deployed robot, the depth tensor reaching the policy matches the M2 student depth format; verified by logging the tensor shape/range from inside the policy container.
- Policy reactivity test: holding an obstacle in front of the robot at varying distances produces a corresponding change in the policy's command output; clip or log captured.

### Task 4 - Sim-to-real tuning and controller-driven hardware demo
#### Narrative
Close the sim-to-real gap on dynamics and produce the milestone's final deliverable: a real Krabby-Uno that, when driven by the Bluetooth controller from Task 1, moves forward when forward is pressed and backward when backward is pressed, with a gait reasonably matching what the policy produced in sim under the same controller input.

Going from "policy reacts to depth on hardware" (end of Task 3) to that demo is sim-to-real gap work. The policy is out-of-distribution on the real robot because the sim's dynamics don't exactly match real friction, body mass, motor strength, control latency, or sensor noise. The standard fix — and the one Extreme Parkour uses to get its student policy zero-shot transferring to a real Unitree — is **domain randomization during training**: randomize the things that differ between sim and real (body mass, CoM, friction, motor strength, PD gains, control/sensor latency, observation noise, camera pose) over wide enough ranges that the real robot's actual values fall inside the distribution the policy was trained against.

The Unitree sim-to-real setup in the Extreme Parkour project is the heaviest reference for this work, since it's a known-working depth-policy student deployed on a real quadruped with the same training pipeline being used here. Study its randomization config in detail — the parameter ranges, the noise schedules, the latency model, the push perturbations during training — and port the equivalent treatment to the crab in the local parkour repo. Reference repos worth mining: `chengxuxin/extreme-parkour` (original), `change-every/Extreme-Parkour-Onboard` (Go2 onboard deployment with documented obs vector and camera randomization).

Alongside randomization, tune the sim's motor torque values against measured real-robot torque output for at least one representative leg DOF, so the sim's actuator model isn't systematically wrong before randomization is even applied.

The deliverable demo doesn't need a finished walking gait across rough terrain — that's reward shaping and fine-tuning for a later milestone. It needs the controller-driven directional motion with a gait that recognizably matches the sim under the same input.
#### Acceptance Criteria
- Domain randomization added to sim training, modeled on the Extreme Parkour Unitree config: at minimum body mass, CoM, ground friction, motor strength, PD gains, control/observation latency, joint position/velocity noise, IMU noise, depth noise, camera pose. Ranges documented and committed in the training config.
- Motor torque values in sim aligned with measured real-robot torque output for at least one representative leg DOF; measurement method (stall test, known-load test, current-to-torque conversion, etc.) and the resulting sim parameter values documented.
- Teacher and/or student retrained on the randomized sim from Task 2/Task 3 base; weights saved and loadable; metrics on the M2 in-distribution eval scenarios within the Task 2 ballpark (robustness gain shouldn't tank in-distribution performance).
- New container image built with the noise-trained weights and deployed to the real robot via fleet portal if available, else `krabby-launcher`.
- **Controller demo (milestone deliverable):** with the Bluetooth controller from Task 1, pressing forward drives the real robot forward and pressing backward drives it backward; the gait recognizably matches the sim's gait under the same controller input. Captured as side-by-side video (sim + real, same controller input, same time window).
- Brief writeup comparing the contractor's randomization config to the Extreme Parkour Unitree reference: what was kept verbatim, what was adapted for the hexapod's prismatic actuators / 6-leg / mass / lever-arm geometry, what was dropped and why.

## Information
- Repositories: `krabby-research` (model, sim, training, action-space code), `patina-foundation-grants` (this grant; task specs and acceptance criteria), `krabby-launcher` (deployment tool), `MCUSDK` (microcontroller firmware + HAL endpoint).
- Assets: M2 teacher/student weights and `crab_hex.usd` in `krabby-research/assets`. Output: retrained weights with prismatic action space, container image deployable via `krabby-launcher` and/or fleet portal.
- Hardware: physical Krabby-Uno hexapod with 18 prismatic lead-screw actuators, krab MCU running `MCUSDK`, on-robot compute running the policy container.
- References: M2 OVERVIEW for training pipeline; M3 (parkour runtime / HAL / depth pipeline) for the runtime stack the deployed image inherits from.

## FAQ
- **Why two options for Task 2?**  
  Both get the policy a prismatic action space; the difference is where the lever-arm geometry lives. Option 1 puts it in `crab_hex.usd` as a real closed-loop articulation, which is more physically faithful (PhysX simulates the actual lead-screw → lever → joint chain, including inertia and contact response at each link), but closed-loop articulations in USD/PhysX are not the default tree topology and need specific joint primitives — they can be fiddly to get stable. Option 2 puts the geometry in a sim-side mapping function and keeps USD as a tree; less faithful but no physics-side risk. Try Option 1 first if there's appetite for the USD work; fall back to Option 2 if it gets stuck.
- **Why retrain at all — why not just convert prismatic commands to rotational at runtime on the real robot?**  
  Because the prismatic↔rotational relationship is nonlinear in joint angle (fast at perpendicular, slow near the extremes). A runtime wrapper on the robot hides that nonlinearity from a policy trained on rotational dynamics; the policy is then over- or under-confident depending on joint configuration. Retraining with the prismatic action space — by either Task 2 option — gives the policy the correct dynamics during training.
- **What if `krabby-launcher` isn't ready when this milestone starts?**  
  Manual `docker run` (or equivalent) with a systemd unit for auto-start is an acceptable fallback for Task 1. Note the gap in the deliverable and roll the launcher integration into Task 3 or Task 4 deploys.
- **What if the fleet portal isn't available for Task 3?**  
  `krabby-launcher` only is fine; the goal of Task 3 is the deployed running model, not the fleet portal specifically. Document the fleet-portal hand-off as a follow-up.
- **What exactly is the Extreme Parkour reference for Task 4?**  
  The Extreme Parkour project (Cheng, Agarwal, Pathak et al., CMU) is the depth-policy student approach this milestone's training is built on, and its sim-to-real setup works zero-shot on a real Unitree. The relevant pieces to study and port are its domain-randomization config (parameter ranges for body mass, CoM, friction, motor strength, PD gains, latency, observation noise), its noise schedules over training, and its push-perturbation regime. Reference repos: `chengxuxin/extreme-parkour` (the original work) and `change-every/Extreme-Parkour-Onboard` (a Go2 onboard-deployment fork with documented observation vector and added camera randomization). The local copy in `krabby-research/parkour` is what gets edited.
- **Is gait quality on the real robot an acceptance criterion?**  
  Partially. Task 4's bar is the gait the policy is *already producing in sim*, reproduced on hardware under controller input — forward press → forward motion, backward press → backward motion, gait recognizably matching sim. That's a sim-to-real fidelity test, not a gait-quality test. Polishing the gait itself (rough terrain, higher speeds, reward shaping) belongs to a later milestone with on-robot fine-tuning.
- **Where are full task details?**  
  In this folder: `Task1-Parkour-Image-Bringup.md`, `Task2-Joint-Mapping-Retrain.md`, `Task3-Depth-Pipeline.md`, `Task4-SimToReal-Tuning.md`.