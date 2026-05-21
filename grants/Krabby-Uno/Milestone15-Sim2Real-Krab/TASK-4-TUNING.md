# Task 4 — Sim-to-real tuning and controller-driven hardware demo

Goal: Close the sim-to-real dynamics gap by porting the Extreme Parkour Unitree domain-randomization approach to the hexapod, set sim motor torques from the published motor specs and drivetrain math (lead-screw actuators don't expose useful torque from steady-state current), retrain, and deploy. Final deliverable: a real Krabby-Uno that, when driven by the Bluetooth controller from Task 1, moves forward when forward is pressed and backward when backward is pressed, with a gait recognizably matching the sim's gait under the same controller input.

Outputs
- A documented walkthrough of the Extreme Parkour Unitree sim-to-real configuration with the parts relevant to the hexapod called out.
- Domain randomization config added to the local parkour repo, with parameter ranges and noise schedules documented.
- Sim motor torque (prismatic actuator force) parameters set from published motor specs and lead-screw geometry; calculation shown.
- Retrained teacher and student weights against the randomized sim; eval metrics on M2 in-distribution scenarios still within ballpark.
- Deployed container image with the noise-trained weights.
- The milestone-deliverable controller demo: side-by-side video of sim and real running the same controller input, showing the real robot moving with a gait recognizably matching the sim.
- Writeup comparing the contractor's config to the Extreme Parkour Unitree reference (kept verbatim / adapted / dropped / added), added as a section to the existing README of the parkour training repo. New doc files only if the existing README is already overlong.

Acceptance Criteria
- **4a** — Domain randomization added to sim training, modeled on the Extreme Parkour Unitree config: at minimum body mass, CoM, ground friction, motor strength, PD gains, control/observation latency, joint position/velocity noise, IMU noise, depth noise, camera pose. Ranges documented and committed in the training config.
- **4b** — Sim motor torque parameters set from motor's published torque spec, multiplied through the lead-screw's mechanical advantage and efficiency. Lead-screw actuators are non-backdrivable and draw no power at rest, so steady-state current measurements don't give useful torque numbers — use the spec. Calculation documented (motor stall torque → max linear force at screw nut → sim prismatic effort/force limit).
- **4c** — Teacher and/or student retrained on the randomized sim; weights saved and loadable; metrics on the M2 in-distribution eval scenarios within the Task 2 ballpark.
- **4d** — New container image built with the noise-trained weights and deployed to the real robot via fleet portal if available, else `krabby-launcher`.
- **4e** — **Controller demo (milestone deliverable):** with the Bluetooth controller from Task 1, pressing forward drives the real robot forward and pressing backward drives it backward; the gait recognizably matches the sim's gait under the same controller input. Captured as side-by-side video (sim + real, same controller input, same time window).
- **4f** — Brief writeup comparing the contractor's randomization config to the Extreme Parkour Unitree reference: what was kept verbatim, what was adapted for the hexapod's prismatic actuators / 6-leg / mass / lever-arm geometry, what was dropped and why.

---

**NOTE**: All code examples and parameter ranges in this document are guidance. The reference values come from quadruped configs and need to be adapted for the hexapod's mass, actuator type, and leg count. Treat as design guidance, not production-ready config.

---

## 1. The Extreme Parkour Unitree reference

The Extreme Parkour project (Cheng, Agarwal, Pathak et al., CMU, 2023) is the depth-policy student approach the M2 training is built on. Its sim-to-real setup achieves zero-shot transfer to a real Unitree quadruped, which is exactly the problem this task is solving — different robot, but the same sim-to-real gap and the same training pipeline. **Study its configuration in detail before writing the hexapod config.**

### 1.1 Reference repos

| Repo | What's there | Where to look |
|---|---|---|
| `chengxuxin/extreme-parkour` | The original project. A1-targeted, no deployment code. | `legged_gym/envs/base/legged_robot_config.py` for the `domain_rand` block; observation vector definition; noise scales. |
| `change-every/Extreme-Parkour-Onboard` | Unofficial Go2 deployment fork. Adds onboard deploy code, documents the observation vector, and adds **camera randomization** for the Go2's movable camera. | Same `domain_rand` block plus camera randomization; the on-robot deploy script for reference on the depth-publishing rate (100 Hz to a ROS topic). |
| `krabby-research/parkour` | Local copy used for M2 training. | Already inherits from the parkour codebase; this task edits the `domain_rand` block in the crab task's env config. |

### 1.2 What to read first

In whichever copy of the Extreme Parkour code is most accessible:

1. The `domain_rand` block of the legged-robot env config. Note the parameter list, the value ranges, and which parameters are randomized per-episode vs. per-step.
2. The observation noise scales (usually a `noise_scales` block). Note which observation components get noise (joint pos, joint vel, IMU angular velocity, IMU projected gravity, etc.) and the noise magnitudes.
3. The push-robot logic in the env (usually `_push_robots` or similar). Note the push interval, push velocity magnitude, and whether pushes are randomized in direction.
4. The motor model — Extreme Parkour uses a learned actuator network in some configs; verify whether the M2 setup is using a PD model or a learned actuator network. The hexapod is brushed-DC + lead-screw, so the sim's motor model needs to reflect that, not a Unitree direct-drive QDD.
5. The latency simulation — usually `action_delay` or `obs_delay` with a step-count range.

### 1.3 What's different about the hexapod

Plan adaptations before writing config:

| Aspect | Unitree (reference) | Hexapod | Adaptation |
|---|---|---|---|
| Number of legs | 4 | 6 | Reward terms scaled per leg; gait is ripple, not trot. |
| Actuator type | Direct-drive QDD, position/torque-controlled | Brushed DC → lead screw → lever arm; non-backdrivable | Motor strength randomization range is different; PD gain randomization is different; sim's motor model must match the lead-screw response. |
| Mass | ~12 kg (Go2) | ~63 kg (krab, ~140 lb, already set in USD) | Mass-related randomization ranges scale roughly with nominal mass; krab nominal is fixed, not something the contractor needs to discover. |
| Camera | Mounted, sometimes movable | Mounted (Task 2) | Camera pose randomization range can be tighter since the krab mount is rigid. |
| Depth resolution | (per M2 student spec) | (per M2 student spec) | Same — depth noise model from Extreme Parkour is reusable. |
| Action space | Joint position targets | Prismatic position/velocity (Task 2) | Action noise / latency apply at the prismatic layer; PD-gain randomization applies to whichever loop closes the prismatic command on the real robot. |

---

## 2. Domain randomization config

The randomization block lives in the crab task's env config in `krabby-research/parkour/`. Adapt from the Extreme Parkour reference; the table below is a starting point — actual ranges should be set based on what the krab's measured variability is, not copy-pasted from a Unitree config.

### 2.1 Parameters to randomize

| Parameter | Reference range (Unitree-ish) | Hexapod starting range | Per-episode or per-step |
|---|---|---|---|
| Body mass | ±10% of nominal | ±5% of nominal (krab nominal is known precisely from CAD/USD, narrow band suffices) | per-episode |
| CoM offset | ±5 cm in body XYZ | ±3 cm in body XYZ | per-episode |
| Payload mass | 0–3 kg | 0–200 kg (krab is designed to haul heavy payload — up to ~270 kg / 600 lb design max) | per-episode |
| Ground friction coefficient | [0.5, 1.25] | [0.4, 1.4] (krab may run on rougher surfaces) | per-episode |
| Motor strength scale | [0.9, 1.1] | [0.8, 1.2] (cheaper brushed DC has wider variance) | per-episode |
| PD gains (Kp, Kd) | ±10% | ±15% | per-episode |
| Joint position noise (obs) | 0.01 rad std | 0.01 rad std (encoder dependent) | per-step |
| Joint velocity noise (obs) | 0.5 rad/s std | 0.5 rad/s std | per-step |
| IMU angular velocity noise | 0.2 rad/s std | 0.2 rad/s std | per-step |
| IMU projected gravity noise | 0.05 std | 0.05 std | per-step |
| Depth noise | Salt-and-pepper + Gaussian | Same model | per-step |
| Control / action latency | 0–20 ms | 0–40 ms (krab's HAL + MCU loop may be slower) | per-episode |
| Observation latency | 0–10 ms | 0–20 ms | per-episode |
| Push velocity | 0.5–1.0 m/s every 7s | 0.5–1.0 m/s every 7s | per-episode push event |
| Camera pose (pitch/yaw) | ±5° / ±5° | ±2° / ±2° (rigid mount) | per-episode |

Lock in the actual ranges with comments referencing the Extreme Parkour reference value and a one-line justification for the hexapod adaptation.

### 2.2 Example shape of the config

Guidance — match the existing config style in `krabby-research/parkour/`:

```python
class CrabHexEnvCfg(LeggedRobotCfg):
    class domain_rand:
        randomize_friction = True
        friction_range = [0.4, 1.4]

        randomize_base_mass = True
        added_mass_range = [-3.0, 3.0]  # kg, ±5% of ~63 kg (krab nominal from USD)

        randomize_com = True
        com_displacement_range = [-0.03, 0.03]  # m

        randomize_motor_strength = True
        motor_strength_range = [0.8, 1.2]

        randomize_kp_kd = True
        kp_factor_range = [0.85, 1.15]
        kd_factor_range = [0.85, 1.15]

        randomize_action_latency = True
        action_latency_range = [0, 0.04]  # s

        push_robots = True
        push_interval_s = 7.0
        max_push_vel_xy = 1.0

        randomize_camera_pose = True
        camera_pitch_range_deg = [-2.0, 2.0]
        camera_yaw_range_deg = [-2.0, 2.0]

    class noise:
        add_noise = True
        class noise_scales:
            dof_pos = 0.01
            dof_vel = 0.5
            ang_vel = 0.2
            gravity = 0.05
            depth = 0.05  # depends on the depth encoding
```

### 2.3 Schedules

Some parameters (camera pose, push perturbations) sometimes anneal in over training — start with no/light randomization, ramp up after the policy learns basic locomotion. Match what Extreme Parkour does and document the schedule.

---

## 3. Set sim motor torque from published specs

Before randomization helps, the sim's central estimate for actuator force needs to be right. If the sim's prismatic actuator can produce 800 N where the real one can produce 400 N, no amount of `motor_strength_range = [0.8, 1.2]` will rescue it.

**Skip the temptation to back this out from steady-state motor current.** The krab's lead-screw actuators are non-backdrivable — at rest, the screw holds the load mechanically (self-locking), and the motor draws no current. Stall tests and known-load hold tests give zero or garbage readings. Use the motor's published torque from the label and propagate it through the drivetrain math.

### 3.1 The drivetrain chain

For each leg DOF, the chain is:

```
motor torque (Nm, from label)
  → screw torque (after any gearbox between motor and screw, if present)
  → linear force at the screw nut (through lead-screw mechanical advantage and efficiency)
  → joint torque at the lever arm (through the geometry from Task 2 Section 2)
```

What the sim needs is the *linear force at the screw nut* — that's what becomes the effort/force limit on the prismatic joint in the sim's actuator model. The lever-arm geometry from Task 2 then handles the prismatic-to-rotational conversion (Option 1 closed-loop USD, or Option 2 mapping function) dynamically.

### 3.2 Linear force at the screw nut

For a lead screw with **lead** *L* (linear distance per revolution, m) and **efficiency** *η* (typically 0.3–0.5 for non-backdrivable Acme or Tr screws — that low efficiency is the price of self-locking), the maximum linear force at the nut from a motor producing torque *T_m* through the screw is:

```
F_nut = T_m × (2π × η) / L
```

For the krab actuators (Tr12×3 lead screw per the M2 mechanical design: *L* = 3 mm = 0.003 m; assume *η* ≈ 0.4 for an Acme-style Tr screw, but pull the actual efficiency from the screw's datasheet if available):

```
F_nut = T_m × (2π × 0.4) / 0.003 ≈ T_m × 838 N/Nm
```

So a motor with stall torque of e.g. 0.5 Nm gives a maximum linear force at the nut of ~420 N. If there's a gearbox between motor and screw with reduction *N*, multiply by *N* before plugging in.

### 3.3 Setting the sim parameter

In the sim's actuator model, find the parameter that limits the prismatic joint's force output — commonly `effort_limit`, `force_limit`, or `max_force` on a `UsdPhysics.PrismaticJoint`'s drive, or a `motor_strength` scaling factor on a PD model. Set it to *F_nut* from Section 3.2 for the representative DOF.

Document, in the writeup section of the parkour repo's README:

- Motor part number and the torque spec value used (stall torque or rated torque — be explicit).
- Lead-screw part and *L*.
- Lead-screw efficiency *η*, with source (datasheet or assumed Acme value).
- Any gearbox between motor and screw, and its reduction *N*.
- Computed *F_nut*.
- Sim parameter name and value set.

Repeat the calculation per joint type (hip yaw, hip pitch, knee) only if the actuators differ between joints; otherwise the same *F_nut* applies to all 18.

### 3.4 Sanity check

After setting the parameter, run a no-randomization sim with the M2 in-distribution evals. Performance should be close to M2 Task 2 results. If the policy can suddenly walk much faster than it could before, the new *F_nut* is too high (and the previous parameter was probably too low). If the policy can no longer walk, *F_nut* is too low. Adjust and document.

Note: the `motor_strength_range = [0.8, 1.2]` randomization in Section 2.2 then varies the policy's training around this nominal *F_nut*, which gives robustness to real-world variation (motor-to-motor differences, screw efficiency variation with wear, lubrication state).

---

## 4. Push perturbations and observation noise

These two are usually under-tuned and cause real-world OOD failures.

### 4.1 Pushes during training

Apply random base-velocity pushes during training to teach the policy to recover. Reference Extreme Parkour pushes every 5–10 seconds with a base XY velocity perturbation up to ~1 m/s. The krab is heavier than a Go2 (~63 kg vs. ~12 kg) but what matters for the policy is the *relative dynamics*, not the absolute push energy. Keep the velocity range similar to the reference.

### 4.2 Observation noise schedule

Per-step noise on joint position, joint velocity, IMU angular velocity, and IMU projected gravity. The standard pattern is Gaussian noise with the scales in Section 2.1. Verify these are *added during training*, not just clipped at deploy — sometimes the noise model only fires on `add_noise = True` and the config has it off by default.

### 4.3 Depth noise

The depth observation in the sim is rendered cleanly. The ZED's real depth output has speckle, holes (NaN), edge-bleeding, and quantization. Match these in sim:

- Per-pixel Gaussian noise on depth values.
- Random salt-and-pepper holes (set to far plane or 0 to match Task 3 handling).
- Edge artifacts can be approximated by jittering depth at depth-discontinuities.

A simple model is usually enough. Reference what Extreme Parkour does; the hexapod's ZED is similar enough to the RealSense D435i used in the Onboard fork that the noise model carries over.

---

## 5. Retraining

With the randomization config and sim motor torque parameters set:

```bash
cd krabby-research/parkour
python train.py --task crab_hex --domain_rand --headless --max_iterations <N>
```

(or whatever the existing train command is).

Train teacher and student in sequence as in M2. Expect somewhat slower convergence than M2 in-distribution training — randomization increases variance. Watch the training curves; if they're not converging at all, the randomization ranges are too wide (or the motor model is wrong). Pull ranges back and re-run.

### 5.1 Sanity checks during training

- Reward curve eventually rises and plateaus (not noise-locked).
- Teacher achieves comparable performance to M2 on in-distribution scenarios at the end of training.
- Student distillation tracks teacher.

### 5.2 Eval

Run the existing M2 eval scenarios with the new student. Metrics should be within Task 2's ballpark. A modest regression is acceptable (randomization adds an in-distribution cost); a large regression means randomization is over-tightening the policy.

Save weights, training-run metadata, and eval metrics.

---

## 6. Deploy and controller demo

### 6.1 Build and deploy

Rebuild the parkour image with the noise-trained weights from Section 5. Reuse the Task 3 build process; only the weights change.

```bash
make parkour-image WEIGHTS=path/to/task4/student.pt \
                   DEPTH_CONVERTER=zed_to_m2 TAG=parkour-m3t4
```

Deploy via fleet portal or `krabby-launcher` as in earlier tasks.

### 6.2 Controller demo procedure

This is the **milestone deliverable**. Set up:

1. Robot in an open, flat indoor space with at least 3 m of forward/backward clearance and someone holding the E-stop.
2. Bluetooth controller paired (from Task 1).
3. Two synchronized recordings:
   - **Sim:** running the same image in a sim-only mode (or the M2 sim with the Task 4 student) with the same controller input piped in.
   - **Real:** robot running the deployed image with the same controller.

Sequence of inputs over a single 30–60 second clip:

- Neutral stick — robot in idle/standing pose.
- Forward stick — robot moves forward; hold for several gait cycles.
- Return to neutral.
- Backward stick — robot moves backward; hold for several gait cycles.
- Return to neutral.

The acceptance bar is **the gait recognizably matches sim under the same input**. It does not have to be a finished walking gait. Foot placement timing and overall body motion should be visibly the same pattern in sim and real.

If the gait diverges sharply:

- Check observation parity: log policy observations in real and sim under the same scene; look for systematic offsets.
- Check action parity: same observations should produce same actions; if not, weight differences (wrong checkpoint loaded?) or numerical issues.
- Check sim torque parameter: if the F_nut calculation used wrong efficiency or wrong lead, sim and real have different available actuator force; recompute and adjust.
- Pull randomization ranges wider and retrain (the policy is OOD on something).

### 6.3 Capture

Side-by-side video of sim + real under the same controller input, plus logged controller input + policy command output + telemetry. Link or embed everything in the existing README of the relevant repo (the parkour repo's README is the natural home for the demo). Don't create a new docs folder for these artifacts — if a binary needs a place to live (video file, log captures), drop it next to other binaries the repo already keeps and link from the README. Make new files only when there's no existing place for the artifact to go.

---

## 7. Writeup: hexapod vs Extreme Parkour Unitree

The comparison writeup lands in the **existing README** of whichever repo owns the parkour training code (most likely `krabby-research/parkour/README.md`). Add a section to it; don't create a new doc file. Structure:

1. **Kept verbatim** — which parameters and ranges were copied as-is from the Extreme Parkour reference (e.g. IMU noise scales, depth noise model).
2. **Adapted** — which parameters were scaled or modified for the hexapod, with one-line justifications (mass scaling, prismatic action space, ripple gait, brushed-DC motor model).
3. **Dropped** — which Extreme Parkour parameters didn't apply (e.g. Unitree-specific actuator-network parameters, anything related to a movable camera if the krab's is rigid).
4. **Added** — anything the hexapod needed that the Unitree config didn't have (e.g. wider control latency range due to HAL+MCU stack, hexapod-specific gait or contact randomization).

Treat this section as substantive, not a bullet list — it's the reference for future sim-to-real work on the krab. But it lives inside the existing README alongside the rest of the parkour repo's documentation; only spin off a new file if the README is already so long that further additions hurt it more than they help.

---

## 8. Deliverable checklist

- [ ] Extreme Parkour Unitree config walkthrough completed; key files and decisions noted in the writeup.
- [ ] Domain randomization config committed to `krabby-research/parkour/`; parameter ranges and schedules documented inline.
- [ ] Sim motor torque (prismatic force) parameter set from motor's published torque spec, lead-screw lead, lead-screw efficiency, and any gearbox reduction. Calculation shown and final *F_nut* documented in the parkour-repo README.
- [ ] Teacher and student retrained on the randomized sim; weights, training metadata, and eval metrics committed.
- [ ] Container image built with noise-trained weights and deployed to the real robot.
- [ ] **Controller demo:** synchronized side-by-side video of sim + real under the same controller input (forward, neutral, backward, neutral). Gait recognizably matches.
- [ ] Logged controller input + policy command output + telemetry captured from the demo and linked from the relevant README.
- [ ] Writeup (kept / adapted / dropped / added) added as a section to the existing parkour-repo README. New doc files only if necessary.