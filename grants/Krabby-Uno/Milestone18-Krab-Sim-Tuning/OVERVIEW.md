# Patina Foundation Grant - Krabby-Uno Milestone 18: RL Gait Reward Tuning (In-Sim)

## Grant Overview
Take the staged curriculum in `krabby-research/parkour/parkour_tasks/parkour_tasks/crab_hexapod_task` (currently ending at the 2b2 teacher `runs/2026-05-26_21-46-37/model_6300.pt` and distilled student `runs/2026-05-26_22-57-01/model_9800.pt`) and turn it into a standard, reproducible training regimen that produces a good policy from scratch, entirely in simulation. The current policies locomote but are not shippable: a "tippy-tap" micro-stepping gait (rapid tiny steps, almost no swing phase), no lateral or yaw command capability, sim actuator limits never checked against the real motors, and no payload capability on a robot whose product is hauling construction loads. The work runs as Task 0 (baseline eval harness + reward audit, prerequisite scale) plus four substantive tasks. Task 1 reshapes the rewards for gait quality (air-time threshold, stride length, contact-schedule regularity, smoothness), one measured change at a time. Task 2 restores omnidirectional motion: true (vx, vy, ωz) command sampling replacing the heading-locked, lateral-penalized model. Task 3 aligns sim joint force/velocity limits with the real actuators (500 N @ 80 mm/s hip, 400 N @ 60 mm/s knee, 90 W 80:1 planetary yaw) and trains a payload curriculum from 200 lb (~90 kg) working up toward the 600 lb (~270 kg) design target. Task 4 is the milestone deliverable: everything consolidated into a scripted from-scratch pipeline with metric gates, run once end-to-end to produce the final teacher and distilled student, plus a baseline-vs-final metric report against student `9800`. The final weights become the training baseline the sim-to-real milestone starts from.

## Why is this Important?
- The current checkpoints can only be recreated by replaying hand-launched resumes with hand-picked checkpoints. A from-scratch, metric-gated pipeline is needed before the prismatic retrain and the sim-to-real DR retrain, both of which will rerun training.
- Sim-to-real transfers whatever gait exists in sim, and a reward iteration takes minutes in sim vs. a session on hardware.
- Tippy-tap is constant direction reversals with almost no travel: the worst duty cycle for lead-screw actuators. A real swing phase is a hardware-longevity requirement.
- The current command model cannot express lateral or yaw motion at all. The sim-to-real controller demo assumes a policy that follows a command stream; this milestone builds that interface.
- The robot's job is hauling construction loads (600 lb design target), and the sim's actuator limits were inherited from the go2 config and were hand tweaked, but never validated against the real motors (500 N hip, 400 N knee, 80:1 yaw).

## Tasks
Total: **~15.5 dev days assigned** out of ~20 part-time working days in a month (4 weeks x 5 days). Dev days are hands-on time; training runs are unattended and dominate calendar time instead (launch, come back, score with the harness), with Task 4's from-scratch run the longest single wall-clock item. The remaining ~4.5 days are unassigned cleanup / integration / slop — Task 2's lateral-widening instability and Task 3's payload-curriculum instability are the likeliest places to absorb them. Full acceptance criteria and implementation detail live in the task files in this folder (`TASK-0-BASELINE-AND-AUDIT.md` through `TASK-4-REPRODUCIBLE-CURRICULUM.md`). Summaries:

### Task 0 - Baseline eval harness and reward audit
#### Narrative
**~1.5 days (range 1-2).** Prerequisite-scale measurement work; to enable agents to run experiments and track results. Build a metrics logger around the existing play path in `crab_hexapod_task/scripts/` that logs per episode: per-foot contact schedules with a scalar tripod-phasing score (the machine-checkable gait-regularity number later tasks gate on; a gait-diagram PNG is kept as a human debug view), air-time and stride distributions, swing clearance, joint torque/velocity, command tracking error, and orientation stability. Run teacher `6300` and student `9800` on a fixed, versioned eval set to capture baselines. Audit every reward term and command range in `parkour_mdp_cfg.py` / `crab_hex_env_cfg.py` against the measurements, with verdicts on the three flagged suspects (50 ms air-time threshold, forward-only command model, speed-forcing terms). Every later task is scored against these numbers.

#### Acceptance Criteria
- Eval harness committed to `crab_hexapod_task/scripts/`; takes checkpoint + task/mode + seeds; emits per-episode metrics files (including the scalar tripod score) and a gait-diagram PNG as a debug view; invocation documented.
- Versioned eval scenario set committed covering `Flat-Walk-v0` and 2b2-mixed terrain.
- Baseline metric reports for teacher `6300` and student `9800` committed.
- Audit table committed: term, weight per stage, intended effect, measured effect, pathology attribution.
- Diagnosis committed: each pathology mapped to a config cause (or cause-unknown), including verdicts on the three suspects.

### Task 1 - Reward shaping for gait quality
#### Narrative
**~3 days (range 2-4).** Fix the pathologies Task 0 attributed to reward terms, one change at a time, scored by the harness after each change. Changes go in new reward stage classes following the existing subclass pattern so bundled checkpoints stay reproducible. Expected changes: raise the 50 ms air-time threshold to a real swing-phase target from measured stride timing; add a stride-length term; add a contact-schedule (tripod) term only if regular phasing does not emerge from air-time/stride alone; retune clearance and energy/smoothness weights to kill the rapid reversals. Terms that do not move their target metric get reverted, not kept. Iterate on flat-walk where changes show fastest, then carry the final term set up the teacher stack onto 2b2 terrain. Teacher only; the student is distilled once in Task 4.

#### Acceptance Criteria
- New reward stage class(es) committed following the existing subclass pattern; bundled stages and checkpoints untouched.
- Air-time reward reworked from measured stride timing; before/after distributions committed.
- Stride-length term implemented; before/after metrics committed.
- Tripod score improves from baseline to a committed target threshold across the commanded speed range, explicit term or emergent; before/after scores and diagrams committed.
- Energy/smoothness retuned; before/after torque and action-rate profiles committed.
- Tippy-tap eliminated on the Task 0 eval set; forward capability at or above baseline or the trade-off documented.
- Per-term changelog committed; retrained teacher in `runs/` with README.

### Task 2 - Omnidirectional command tracking
#### Narrative
**~4 days (range 3-6).** Make the policy drivable: it goes where the (vx, vy, ωz) command points, anywhere in the envelope, with Task 1 gait quality holding throughout — including sustained sideways travel at fixed heading (the crab knee walk, prerequisite for 30" doorway traversal later). The curriculum resumes from the Task 1 checkpoint and widens the command distribution in stages: yaw arcs first, then progressively wider lateral ranges, with dense intermediate-direction sampling and in-episode command resampling so one network learns a continuous gait family rather than discrete modes (at intermediate angles the feet trace diagonal strides; there is no gait switch). The lateral gait works the leg differently (knee extension/retraction instead of fore-aft sweep) but is expected to keep the same alternating-tripod support; the gate is the direction-agnostic metrics, and any different regular phasing that emerges is documented rather than forbidden. Forward-privileged reward terms are retired. Verification is an omnidirectional eval: a fixed table of command points plus a direction sweep (rotate the commanded direction at constant speed and check for tracking or gait discontinuities at intermediate angles).

#### Acceptance Criteria
- Command model reworked and committed: sampled vy/ωz, commanded yaw rate replacing heading P-control, `penalty_lin_vel_y` and other forward-privileged terms retired or reworked, envelope documented from actuator specs, audit table updated.
- Training curriculum committed: staged command widening from the Task 1 checkpoint with dense intermediate-direction sampling and in-episode command resampling.
- Omnidirectional eval committed: fixed command-point table plus direction sweep, emitting per-point tracking error, net heading drift, and gait metrics.
- Capability gate: retrained teacher meets documented per-axis tracking tolerance at every eval point; pure-lateral points hold heading (no turn-then-walk); gait metrics stay within the Task 1 ballpark at every point including the sweep.
- Selected checkpoint committed to `runs/` with README.

### Task 3 - Actuator dynamics realism and payload curriculum
#### Narrative
**~4 days (range 3-5).** Two jobs in dependency order. First, derive sim joint effort/velocity limits from the real actuators and lever geometry: hip 500 N @ 80 mm/s unloaded (~60 loaded) on a 4" lever arm driving a 28" segment, knee 400 N @ 60 mm/s on a 5" lever driving a 32" segment, yaw 90 W through an 80:1 planetary. The lever converts slow actuator travel into fast leg motion (the hip's segment end moves ~7x the actuator speed) and the ratio changes with joint angle and load; the sim's constant drive limits are set from mid-stroke geometry with a randomized per-episode velocity/strength scale spanning the loaded-to-unloaded and angle-dependent range, so the policy is robust enough to carry directly into M15 with exact joint dynamics as a later cleanup. The current limits are go2-inherited and were never checked. Second, train a staged payload curriculum as added mass + CoM shift on the body: stable gait and tracking at 90 kg (200 lb), then stepwise up toward the 270 kg (600 lb) design target. Document the maximum stable payload and, if below target, the limiting factor. The gait under load will change character (lower, wider, slower); judge it by the metrics, not resemblance to the unloaded gait.

#### Acceptance Criteria
- Joint effort/velocity limits derived per joint type from the specs and lever geometry (angle-dependent range and loaded/unloaded droop included); calculation documented.
- Sim drive config updated: mid-stroke constant limits plus randomized velocity/strength scale spanning the real spread; retuned no-payload policy saturates at realistic values with gait metrics in the Task 2 ballpark.
- Payload randomization (added mass + CoM shift) implemented and documented.
- Staged payload curriculum trained: stable at 90 kg, stepped toward 270 kg; per-level metrics committed.
- Maximum stable payload documented with metric evidence and the limiting factor if below 270 kg.
- No unloaded regression: zero-payload flat metrics within the Task 2 ballpark.

### Task 4 - Reproducible from-scratch training curriculum
#### Narrative
**~3 days (range 2-4), plus the from-scratch run's GPU wall-clock.** The milestone deliverable. Consolidate the Task 1-3 configs into a scripted (or fully documented) pipeline that trains the teacher from scratch through all stages and distills the student once, with pinned seeds/configs and stage-transition gates defined as Task 0 harness metrics with committed thresholds. Execute one from-scratch run end-to-end. Produce the baseline-vs-final metric report (full Task 0 metric set, Task 2 tracking grid, Task 3 payload curve) comparing student `9800` to the final student.

#### Acceptance Criteria
- Pipeline committed: from-scratch teacher through all stages plus distillation, runnable from pinned configs and seeds with no manual checkpoint hand-off between stages.
- Stage gates defined as harness metrics with committed thresholds.
- One from-scratch run executed end-to-end, every gate passed; wall-clock and hardware documented.
- Student distilled once from the final teacher; selected by play + metrics.
- Final teacher and student in `runs/` with READMEs, tagged as the sim-to-real starting baseline.
- **Milestone deliverable:** baseline-vs-final metric report committed (`9800` vs. final student across the full metric set, tracking grid, and payload curve).

## Information
- Repositories: `krabby-research` (task code under `parkour/parkour_tasks/parkour_tasks/crab_hexapod_task`: reward stages in `config/crab_hex/agents/parkour_mdp_cfg.py`, terrain/command modes in `config/crab_hex/crab_hex_env_cfg.py`, scene/sensors in `crab_hex_scene_cfg.py`, scripts in `scripts/`, checkpoints in `runs/`), `patina-foundation-grants` (this grant; task specs and acceptance criteria).
- Assets: canonical hexapod USD `crab_simple.usda` in `krabby-research/assets` (resolved via `KRABBY_HEX_USD_PATH`); baseline checkpoints teacher 2b2 `model_6300.pt` and student `model_9800.pt` (paths above). Output: eval harness, audit/diagnosis, new reward/command/actuator configs, from-scratch training pipeline, final teacher/student weights tagged as the sim-to-real baseline, baseline-vs-final metric report.
- Hardware: none. This milestone is entirely in simulation (Isaac Lab, `Flat-Walk-v0` / `Teacher-v0` / `Student-v0`); no robot access required. Real actuator specs used for sim tuning: hip 500 N @ 80 mm/s, knee 400 N @ 60 mm/s, yaw 90 W with 80:1 planetary; robot mass ~83 kg; payload design target 600 lb (~270 kg).
- References: the crab_hexapod_task README (stage table, §4.2b sweet-spot notes, Appendices C-G) for the curriculum this milestone consolidates; the go2 `extreme_parkour_task` configs as the quadruped reference the crab stages were derived from.

## FAQ
- **Why is the deliverable a training pipeline instead of a demo video?**
  The upcoming prismatic retrain and sim-to-real DR retrain both rerun training from this milestone's configs, so what they need is a working pipeline with metric gates, not a recording of one checkpoint. Anyone who wants to see the gait runs the documented play command on the committed checkpoints.
- **Isn't Task 0 trivial if the reward terms already exist?**
  The audit part is quick, which is why it is Task 0 and not a week. The new work is the metrics harness: nothing today measures air time, stride, contact schedules, or tracking error, and the harness is also the gate mechanism for every stage of the Task 4 pipeline.
- **Why new reward stage classes instead of editing the 2b2 config in place?**
  Bundled checkpoints must stay reproducible from their committed configs (README Appendices C-G). New subclasses extend the pattern the repo already uses for every stage transition.
- **Why a tripod-gait term? Shouldn't the policy discover the gait itself?**
  It should. If regular phasing emerges from the air-time/stride/energy terms alone, the explicit term is dropped. It exists as a fallback because the reward set is derived from a quadruped stack, which never had to rule out hexapod-specific degenerate modes.
- **Why retune actuator limits here instead of in the sim-to-real milestone?**
  Because Task 3's payload curriculum and Task 2's command envelope both depend on the limits being right. Sim-to-real still owns transfer-gap DR (motor-strength variance, latency, sensor noise, camera pose); this milestone sets the central values that DR varies around.
- **Does this milestone touch the action space (prismatic vs. rotational)?**
  No. Training stays on the current rotational action space; the prismatic conversion is the sim-to-real milestone's job. The Task 3 limit derivation maps actuator force through the lever-arm geometry onto the rotational joints.
- **What if 600 lb payload isn't reachable this milestone?**
  Document the maximum stable payload and the limiting factor (actuator saturation, stability, training instability), so the next iteration knows whether the fix is mechanical, reward-side, or curriculum-side.
- **Which M15 (sim-to-real) work is explicitly not happening here?**
  All of it. No prismatic action space or closed-loop lever-arm joints in USD (M15 Task 2); no ZED depth pipeline or camera-pose matching (M15 Tasks 2-3); no transfer-gap domain randomization or latency/sensor-noise modeling (M15 Task 4); no container builds, deployment, or controller demo. One overlap to keep straight: M18 Task 3 sets *rotational* joint effort/velocity limits from the motor specs; M15 Task 4 recomputes them as *prismatic* force limits (F_nut through lead-screw efficiency) after the action-space conversion.The reason we do this in this order is so we can separate creation of a stable gait from the complexities of the slot yaw motor and prismatic joint dynamics. The M18 derivation and its documented lever-arm math are the input to that recompute, not a substitute for it.
- **Is the yaw motor modeled correctly?**
  No, and neither M18 nor M15 fixes it. Sim models yaw as a bounded revolute joint driven directly. The real v0.2 mechanism is a 90 W motor through an 80:1 planetary driving a cam+slot: the motor can rotate continuously in one direction while the leg oscillates through its arc, and there is currently no way to know absolute leg position on that mechanism (no end stops for the M17 self-heal trick to anchor on; the planned fix is an index sensor at a reference angle, per the M17 FAQ, in a future milestone). M18 Task 3 only sets the yaw joint's torque/velocity limits through the 80:1 reduction; the bounded-revolute abstraction stays.
- **Does the sim's leg-position convention match the new firmware telemetry?**
  No, and nothing currently converts between them. The model consumes what the sim trains on: `joint_pos - default_joint_pos` in radians per revolute DOF (`crab_hexapod_task/mdp/observations.py`). The new V0.3 firmware (M17) reports normalized position in [0.0, 1.0] (0.0 fully retracted, 1.0 fully extended) for hip/knee, TBD for yaw. The HAL→model mapper (`compute/parkour/mappers/hardware_to_model.py`, `_extract_proprioceptive`) copies `joint_positions` straight into the model slots with no remap. Deploying on real hardware requires normalized→radians conversion through the lever-arm geometry (nonlinear, per-joint) minus the default pose. That lands in M17 Task 6's mapper audit and M15 Task 2's joint-mapping work, not here; M18's contribution is documenting the lever-arm math the conversion will use.
- **Does the USDA match the real v0.2 robot's joint ranges?**
  Not verified. The full extension/retraction limits (and default pose) in `crab_simple.usda` have never been measured against the physical robot. They are somewhat close, but not perfect. M17's calibration produces the measured end stops per joint; updating the USDA to match them exactly is follow-on work scoped in neither M18 nor M15. Until that lands, sim joint ranges are approximate and the Task 3 lever-arm derivations should flag any range that looks inconsistent with the hardware.
- **What else is out of scope for both M18 and M15?**
  On-hardware gait fine-tuning and rough-terrain (Stage 4) capability; narrow-gap/doorway traversal training (Task 2's heading-held strafe is the prerequisite gait, but training against 30" doorway geometry is a follow-on); the MCU-mounted BMI270 IMU (M16); payload validation on the physical robot (M18's payload work is sim-only); foot-slip and soft/compliant ground modeling (sim terrain is rigid); battery, thermal, and duty-cycle limits under sustained load; fleet/production calibration recall (F6).
