# Task 3 - Actuator dynamics realism and payload curriculum

Goal: Two jobs in dependency order. First, make the sim's joint force/velocity limits match the real motors; with wrong limits, payload training produces a policy that either can't lift what the hardware can or relies on force the hardware doesn't have. Second, train a payload curriculum from 200 lb (~90 kg) working up toward the 600 lb (~270 kg) design target, and document the maximum payload the policy carries with stable gait.

Real actuator specs and lever geometry (per joint, all 6 legs):

| Joint | Actuator | Speed | Lever arm (actuator anchor to pivot) | Segment length |
|---|---|---|---|---|
| Hip (pitch) | Linear lead-screw, 500 N | 80 mm/s unloaded, ~60 mm/s loaded | 4" (~100 mm) | 28" (~710 mm) |
| Knee | Linear lead-screw, 400 N | 60 mm/s unloaded, droops similarly loaded | 5" (~127 mm) | 32" (~813 mm) |
| Yaw | 90 W motor, 80:1 planetary gearbox | Torque/speed from motor datasheet through the 80:1 reduction | n/a (rotary) | n/a |

The actuator speed is not the leg speed. The lever converts slow linear travel into fast angular motion: the hip's 80 mm/s on a ~100 mm lever arm is roughly 0.8 rad/s at the joint, which moves the end of the 28" segment at roughly 560 mm/s — 7x the actuator speed (28"/4"). The knee is about 60/127 ≈ 0.47 rad/s and ~380 mm/s at the end of its 32" segment. Both numbers change through the stroke, because the effective lever arm is the perpendicular distance from the pivot to the actuator's line of action, which varies with joint angle, and they droop under load (hip 80 → ~60 mm/s).

Outputs
- Sim joint effort/velocity limits derived from the actuator specs through the lever-arm geometry; derivation documented.
- Payload + CoM randomization config with a staged mass curriculum.
- Retrained teacher checkpoint in `runs/` with README.
- Documented maximum stable payload with metrics per payload level.

Acceptance Criteria
- **3a** - Joint-level effort and velocity limits derived per joint type from the specs and lever geometry above (hip/knee: force and speed through the 4"/5" lever arms including the angle-dependent range over the stroke and the loaded/unloaded speed droop; yaw: motor torque through the 80:1 reduction); calculation documented in the training repo README.
- **3b** - Sim drive config updated: constant limits set from mid-stroke geometry plus a randomized per-episode velocity/strength scale spanning the loaded-to-unloaded and angle-dependent range; on the Task 0 eval set the retuned no-payload policy's torque and velocity profiles saturate at realistic values, and gait metrics stay within the Task 2 ballpark (retrain to recover if the tighter limits break the gait).
- **3c** - Payload randomization (added mass + CoM shift, attached to the body as carried load) implemented in the event config; ranges and attachment documented.
- **3d** - Staged payload curriculum trained: stable gait and command tracking at 90 kg (200 lb), then increased stepwise toward 270 kg (600 lb); per-level gait and tracking metrics committed.
- **3e** - Maximum stable payload documented with the metric evidence; if below 270 kg, the limiting factor identified (actuator saturation, stability, training instability) so the next iteration knows what to change.
- **3f** - No unloaded regression: zero-payload flat-ground metrics within the Task 2 ballpark; comparison committed.

---

**NOTE**: Numbers below are derivation guidance; pull actual lever-arm geometry from the USD/CAD and the yaw motor torque from its datasheet.

---

## 1. Actuator limits before payload

The current sim drive limits are inherited from the go2-derived config and have never been checked against the krab's motors. Fix that first: derive joint torque and velocity limits from the specs.

- **Hip/knee (linear):** joint torque = actuator force x effective lever arm; joint angular velocity = actuator speed / effective lever arm. Both vary with joint angle (the effective lever arm is the perpendicular distance from pivot to the actuator's line of action) and with load (the speed droop in the table). Compute the range over the joint's stroke and document it.
- **Yaw (rotary):** motor torque from the 90 W motor's datasheet, multiplied through the 80:1 planetary (include gearbox efficiency), gives the joint torque limit directly.

**Set close, randomize wide.** The sim's revolute drives take one constant effort/velocity limit per joint; the real relationship is angle-dependent and load-dependent, and neither M18's rotational joints nor M15's first prismatic pass will model that coupling exactly. So: set the constant limit from the mid-stroke geometry, then randomize a per-episode velocity/strength scale wide enough to cover the real spread — from the loaded floor (~60 mm/s hip equivalent) to the unloaded peak, and the angle-dependence across the stroke. The intent is that a policy robust across that band works directly in M15 without a dynamics-only retrain, and the exact angle-dependent joint model is a later cleanup. Escape hatch if that proves unreasonable: if the policy exploits top speed at joint angles where the real mechanism cannot deliver it (visible as tracking or gait failures concentrated near the stroke ends in the harness data), fall back to the conservative angle-minimum as the constant limit and document the choice.

Sanity check: run the Task 0 eval on the retuned sim with no payload. If the policy suddenly can't walk, the old limits were inflated and a recovery retrain on the Task 2 base is in scope here.

Two known modeling gaps to work around, not fix (see the OVERVIEW FAQ): the real yaw mechanism is a cam+slot (continuous motor rotation → leg oscillation, no absolute position sensing) that sim abstracts as a bounded revolute joint, so only its torque/velocity limits are set here; and the USDA joint ranges/default pose have not been verified against the physical v0.2 robot, so flag any lever-arm-derived number that looks inconsistent with hardware rather than silently trusting the USD.

## 2. Payload curriculum

Model payload as added mass + CoM shift on the body (carried load), extending the event/DR machinery the 2b2 stage already runs. Stage the mass: train to stable gait and tracking at 90 kg (200 lb), then raise in steps (e.g. 90 → 180 → 270 kg) as long as training stays stable. The robot itself is ~83 kg, so the top of this range is over 3x body weight; expect the gait to change character under load (lower, wider stance, slower) and judge it by the gait metrics, not by resemblance to the unloaded gait.

If a payload level destabilizes training, document the stable ceiling and the limiting factor and move on. Keep the existing light terrain mix (2b2-comparable) in the training distribution; Stage 4 parkour geometry stays out of scope.
