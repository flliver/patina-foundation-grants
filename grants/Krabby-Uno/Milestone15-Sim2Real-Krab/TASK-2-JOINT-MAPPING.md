# Task 2 — Sim/real joint mapping and retrain

Goal: Give the parkour policy a prismatic action space that matches the real robot's lead-screw actuators, by one of two routes (real closed-loop USD or a sim-side mapping function), mount the ZED on the robot and match the sim camera to it, then retrain teacher and student against the new sim.

Outputs
- A documented audit of every joint transform between sim (`crab_hex.usd`) and real (`MCUSDK`/HAL): units, sign conventions, frames.
- Lever-arm geometry documented for hip yaw, hip pitch, and knee on a representative leg.
- **Either** an updated `crab_hex.usd` with real prismatic joints in a closed-loop articulation (Option 1, preferred), **or** a sim-side prismatic→rotational mapping function (Option 2, fallback).
- ZED depth camera mounted on the real robot in a documented location, with the sim's depth camera in `crab_hex.usd` moved to match.
- A command-parity bench test showing identical prismatic commands drive sim and real leg through the same displacement across the joint range.
- Retrained teacher and student weights against the updated sim, with eval metrics within the M2 ballpark on the in-distribution scenarios.

Acceptance Criteria
- **2a** — Full audit of joint input/output transforms between sim and real documented; every unit, sign convention, and frame called out.
- **2b** — Lever-arm geometry documented for hip yaw, hip pitch, and knee (lever length, actuator anchor points, joint angle range, resulting prismatic↔rotational relationship).
- **2c** — One of:
  - **Option 1**: `crab_hex.usd` updated so each leg DOF is driven by a real prismatic joint coupled through the lever-arm geometry to a rotational joint; closed-loop articulation stable under the M2 training workload; brief writeup of which PhysX/USD primitives were used and any references followed.
  - **Option 2**: Sim-side prismatic→rotational mapping function implemented and unit-tested at the sim's actuation layer; policy sees prismatic action space; rotational joints in `crab_hex.usd` unchanged.
- **2d** — Sim policy action space is prismatic (lead-screw displacement / velocity), matching the real robot's actuators.
- **2e** — Command-parity test: identical prismatic command to one leg drives sim and real leg through the same displacement (within documented tolerance) across the joint range.
- **2f** — ZED depth camera physically mounted on the real robot; mount location and orientation (position relative to body frame, pitch/yaw of optical axis) documented.
- **2g** — Depth camera in `crab_hex.usd` repositioned to match the real ZED mount within documented tolerance.
- **2h** — Teacher and student retrained on the updated sim; weights saved and loadable; metrics within the M2 ballpark on the existing eval scenarios.

---

**NOTE**: All code examples in this document are design guidance. Treat them as design guidance, not production-ready code. Actual implementation may require adjustments based on Isaac Sim / Isaac Lab API versions.

---

## 1. Joint transform audit

Before any code changes, document the full chain from policy action to motor command for one representative leg DOF, then repeat for each unique joint type. The audit catches sign flips, unit mismatches, and frame inconsistencies that would otherwise show up as confusing behavior at deploy time.

For each joint DOF (hip yaw, hip pitch, knee — one representative leg is sufficient), document:

| Layer | Variable | Unit | Sign convention | Frame |
|---|---|---|---|---|
| Policy action output | (e.g. `action[knee_idx]`) | normalized [-1, 1] or m or m/s | "+" means ... | — |
| Sim actuation layer | joint command into USD | rad, rad/s, m, or m/s | "+" means ... | body / world / joint |
| USD joint | joint state | rad or m | "+" means ... | joint local |
| HAL command topic | command to `MCUSDK` | (units) | "+" means ... | — |
| `MCUSDK` setpoint | motor target | encoder counts / mm | "+" means ... | actuator local |
| Real motor | shaft rotation | encoder counts | "+" means ... | actuator local |
| Real joint | joint angle | rad | "+" means ... | joint local |
| Telemetry layer | joint position out | rad | "+" means ... | joint local |

Capture the table in the deliverable. Any row where the unit or sign convention changes is a place where an explicit conversion is needed; verify the conversion is correct and that left/right legs are mirrored as expected.

Common bugs the audit catches:

- Hip yaw left-vs-right sign convention (mirrored legs).
- Knee command "+" means extend in sim but flex on real (or vice versa).
- IMU axes (NED vs ENU vs FRD vs body-local) mismatched between sim observation and real observation.
- Encoder counts per radian off by 2× because of a missed gearbox ratio.

Stash the table at `krabby-research/docs/sim-real-audit.md`.

---

## 2. Lever-arm geometry

A lead-screw linear actuator drives each rotational joint through a lever arm. The relationship between actuator displacement and joint angle is set by the four-bar geometry of the actuator anchor points and the lever arm length. The math here is the same for all three joint types — only the parameter values change per joint.

### 2.1 Geometry

For one joint:

- `O` — joint pivot
- `A` — actuator anchor on body (fixed); `|OA| = a`
- `B` — actuator anchor on lever arm (rotates around `O`); `|OB| = b`
- `φ` — angle at `O` between `OA` and `OB`
- `x` — actuator length, `|AB|`

By law of cosines:

```
x² = a² + b² − 2·a·b·cos(φ)
```

Differentiating with respect to time:

```
dx/dt = (a·b·sin(φ) / x) · dφ/dt
dφ/dt = (x / (a·b·sin(φ))) · dx/dt
```

Interpretation: for a given prismatic actuator velocity `dx/dt`, the joint angular velocity `dφ/dt` is **largest when `φ ≈ π/2`** (actuator perpendicular to lever arm) and **smallest near `φ = 0` or `φ = π`** (actuator aligned with lever arm at extremes of travel). The user-facing summary: *prismatic moves quickly per unit joint rotation when perpendicular, slowly when extended/retracted*.

### 2.2 Singular configurations

`sin(φ) → 0` at `φ = 0` and `φ = π` makes `dφ/dt → ∞` for nonzero `dx/dt`. Joint limits should exclude a margin around these singularities. Document the usable `φ` range and the corresponding `x` range per joint.

### 2.3 Per-joint parameters

For each of the three joint types, document the geometry in `krabby-research/docs/lever-arm.md`:

| Joint | `a` (in / mm) | `b` (in / mm) | `φ` range (deg) | `x` range (mm) | Notes |
|---|---|---|---|---|---|
| Hip yaw | | | | | Lever arm ~8" per krab BOM |
| Hip pitch | | | | | Lever arm ~6" per krab BOM |
| Knee | | | | | Lever arm ~5" per krab BOM |

Confirm values against the current CAD; values in earlier conversations may be stale.

---

## 3. Option 1 — Closed-loop USD with real prismatic joints (preferred)

This is the more physically faithful path: the USD itself contains the prismatic actuator, the lever arm, and a closed-loop constraint coupling actuator displacement to joint rotation. PhysX simulates the actual lead-screw → lever → joint chain. The policy outputs prismatic commands and PhysX handles the kinematics.

### 3.1 Closed-loop articulations in USD/PhysX

USD/PhysX articulations are tree-topology by default. A closed-loop linkage (actuator anchored to body at one end, anchored to a point on the lever arm at the other, with the lever arm itself a child of the joint) cannot be expressed as a pure tree. The standard approach is:

- Keep the main articulation as a tree (body → hip → femur → tibia, with rotational joints).
- Add the prismatic actuator as a separate rigid body.
- Couple the actuator's two ends to the body and to the lever arm using **distance joints** (or D6 joints with the appropriate axes locked) outside the articulation tree.
- Drive the prismatic joint as the actuated DOF; the joint rotation follows from the closed-loop constraint.

NVIDIA documents this pattern. Primary reference:

- **Isaac Sim — closed-loop structures**: https://docs.isaacsim.omniverse.nvidia.com/4.5.0/robot_setup/rig_closed_loop_structures.html
- **PhysX articulation reference**: see joint types `UsdPhysics.DistanceJoint`, `PhysxSchema.PhysxJoint`, and the D6 joint primitive.

### 3.2 Example shape of the USD change

Guidance only — actual prim paths and joint primitives depend on the current state of `crab_hex.usd`:

```python
from pxr import Usd, UsdPhysics, PhysxSchema, Sdf

stage = Usd.Stage.Open('krabby-research/assets/crab_hex.usd')

for leg in ['lf', 'lm', 'lr', 'rf', 'rm', 'rr']:
    for joint in ['hip_yaw', 'hip_pitch', 'knee']:
        # Existing rotational joint stays in the articulation tree
        rot_joint_path = f'/crab_hex/{leg}_{joint}'

        # New prismatic actuator body (lead-screw + nut)
        actuator_body_path = f'/crab_hex/{leg}_{joint}_actuator'
        # ... rigid body + collision setup ...

        # Prismatic joint along the actuator's axis (this becomes the actuated DOF)
        prismatic_path = f'/crab_hex/{leg}_{joint}_prismatic'
        prismatic = UsdPhysics.PrismaticJoint.Define(stage, prismatic_path)
        # ... set body0 = body, body1 = actuator_body, axis, limits ...

        # Closed-loop coupling: distance constraint from actuator tip to lever-arm anchor point
        coupling_path = f'/crab_hex/{leg}_{joint}_coupling'
        coupling = UsdPhysics.DistanceJoint.Define(stage, coupling_path)
        coupling.CreateBody0Rel().SetTargets([actuator_body_path])
        coupling.CreateBody1Rel().SetTargets([f'/crab_hex/{leg}_{joint}_lever_anchor'])
        coupling.CreateMinDistanceAttr(0.0)
        coupling.CreateMaxDistanceAttr(0.001)  # near-zero tolerance for a rigid coupling

        # Disable the rotational joint's actuation; it becomes passive
        # ...

stage.Save()
```

### 3.3 Stability notes

Closed-loop articulations in PhysX can be fiddly:

- Solver iteration counts matter; bump position/velocity solver iterations on the affected articulations.
- Distance joints with `max = 0` can be too stiff; a small tolerance plus high break force is usually more stable than an absolute rigid constraint.
- Mass ratios between the actuator body and the lever can cause instability — keep the actuator body's mass small relative to the lever it drives.
- The training rate (Isaac Lab default is 200 Hz physics, 50 Hz policy) may need to be increased for the closed loop to stay stable; if so, retraining cost goes up.

If the closed loop won't go stable inside the time budget, fall back to Option 2.

### 3.4 What goes in the writeup

- Which PhysX/USD primitives were used (`DistanceJoint`, `D6`, etc.).
- Solver settings that were needed.
- Any references followed beyond the Isaac Sim closed-loop guide.
- Per-joint parameters (actuator body mass, joint limits, lever-anchor positions).

---

## 4. Option 2 — Sim-side prismatic→rotational mapping function (fallback)

If Option 1 won't stabilize, leave the USD's rotational joints alone and put the lever-arm geometry in a sim-side mapping function at the actuation layer. The policy outputs prismatic commands; the mapping function converts them to equivalent rotational commands using the geometry from Section 2.

### 4.1 Function shape

For each joint, given:

- Prismatic command `vp = dx/dt` (or position `xp`)
- Current joint angle `φ`
- Geometry parameters `a`, `b`

Compute:

```python
import math

def prismatic_to_rotational_velocity(vp: float, phi: float, a: float, b: float) -> float:
    """Map prismatic velocity (m/s) to equivalent rotational velocity (rad/s)
    via lever-arm geometry."""
    x = math.sqrt(a*a + b*b - 2*a*b*math.cos(phi))
    sin_phi = math.sin(phi)
    if abs(sin_phi) < 1e-3:
        # Near-singular configuration; clamp or skip
        raise ValueError(f"Singular configuration at phi={phi}")
    return (x / (a * b * sin_phi)) * vp
```

For position commands (`xp` instead of `vp`), invert the law-of-cosines relationship:

```python
def prismatic_to_rotational_position(xp: float, a: float, b: float) -> float:
    """Map prismatic length (m) to corresponding joint angle (rad)."""
    cos_phi = (a*a + b*b - xp*xp) / (2*a*b)
    cos_phi = max(-1.0, min(1.0, cos_phi))  # numerical safety
    return math.acos(cos_phi)
```

### 4.2 Where it lives in the sim

The mapping function must run *between* the policy action output and the rotational joint command in `crab_hex.usd`. In Isaac Lab, this is the actuator/action-manager layer; look at the existing `JointAction` handling and insert a custom action term that applies the mapping per joint.

### 4.3 Unit tests

- Round-trip test: `prismatic_to_rotational_position` then back through the inverse should give the original prismatic length within tolerance, across the usable `φ` range.
- Singular-configuration test: function raises (or clamps to limit) at `φ = 0` and `φ = π`.
- Per-joint test: spot-check several `φ` values for hip yaw, hip pitch, knee with the documented geometry.
- Sign / direction test: a positive `vp` produces the expected sign of `dφ/dt` for the joint's mounting orientation.

### 4.4 Sim observability

When using the mapping function, the policy observes prismatic state, not rotational state. The observation layer also needs the rotational-to-prismatic inverse so that joint positions from the USD become prismatic positions in the observation vector. Mirror the mapping function on the observation side.

---

## 5. ZED mount and sim camera alignment

The camera-frame alignment is the same kind of sim/real frame-matching work as the joint mapping. The policy needs the sim camera pose during training to match the real ZED's pose during deployment, or the depth observations diverge by an offset the policy doesn't know about.

### 5.1 Mount the ZED on the real robot

- Pick a location on the body with a clear forward view that doesn't get occluded by legs during gait.
- Mount it level (or at a documented downward tilt) on the body frame.
- Document position relative to the body frame origin (X forward, Y left, Z up — or whatever the existing convention is from `MCUSDK`) in mm.
- Document optical-axis pitch and yaw in degrees relative to the body frame.
- Bench-test: power the ZED, capture a depth frame, sanity-check that the frame shows the expected forward view.

### 5.2 Move the sim camera in `crab_hex.usd` to match

The depth camera in the M2 USD is positioned wherever the M2 contractor put it. Update its position and orientation to match the real ZED mount within documented tolerance.

```python
from pxr import Usd, UsdGeom, Gf

stage = Usd.Stage.Open('krabby-research/assets/crab_hex.usd')
camera = UsdGeom.Camera(stage.GetPrimAtPath('/crab_hex/depth_camera'))

# Position relative to body frame, in meters; pitch/yaw from mount measurement
xform = UsdGeom.Xformable(camera)
xform.ClearXformOpOrder()
xform.AddTranslateOp().Set(Gf.Vec3d(x_mm/1000.0, y_mm/1000.0, z_mm/1000.0))
xform.AddRotateXYZOp().Set(Gf.Vec3f(pitch_deg, yaw_deg, 0.0))

stage.Save()
```

Document the resulting USD camera transform alongside the real mount measurements; the two should agree within a couple of mm and a degree or two.

### 5.3 Parameter parity

Beyond pose, verify FOV, near/far planes, and resolution match what the real ZED will publish through HAL after the Task 3 conversion function. Lock these in here; Task 3 builds on them.

---

## 6. Command-parity test

Bench test that proves sim and real respond identically to a prismatic command. This is the most important sanity check before retraining.

Procedure:

1. Constrain the real robot (on a stand, leg in the air).
2. Pick one representative leg DOF (e.g. knee on the front-right leg).
3. Send a sequence of prismatic position setpoints to the real robot's MCU (e.g. step from `x_min` to `x_max` in 5 mm increments, hold at each).
4. Capture the resulting joint angle from telemetry at each setpoint.
5. Repeat in sim with the same setpoint sequence to the same DOF in `crab_hex.usd` (closed-loop or mapping-function, whichever option was taken).
6. Plot sim joint angle vs. real joint angle across the setpoint sequence.

Acceptance: sim and real agree within a documented tolerance across the full setpoint range. Tolerance budget should account for real-world play (lead-screw backlash, lever-arm flex) and is contractor's judgment to set — but should be small enough that a policy trained against the sim can drive the real robot meaningfully.

Capture the plot and the data in the deliverable.

---

## 7. Retraining

With the prismatic action space in place (Option 1 or Option 2) and the sim camera matched to the real ZED:

- Retrain the teacher policy through the M2 curriculum.
- Retrain the student with depth input.
- Save weights to `krabby-research/checkpoints/` with version tags.
- Run the existing M2 eval scenarios; metrics should land within the M2 ballpark. A modest regression is acceptable (the action space changed) but a large regression suggests a bug in the mapping or the closed-loop setup.

Document the training command, training time, hardware used, and final eval metrics.

---

## 8. Deliverable checklist

- [ ] Joint transform audit table at `krabby-research/docs/sim-real-audit.md`.
- [ ] Lever-arm geometry documented at `krabby-research/docs/lever-arm.md` with per-joint `a`, `b`, `φ` range, `x` range.
- [ ] **Either** Option 1 closed-loop USD changes committed to `crab_hex.usd` with a writeup of primitives used and references followed, **or** Option 2 mapping function committed to the sim's actuation layer with unit tests.
- [ ] ZED mounted on the real robot; mount location and optical-axis orientation documented.
- [ ] Sim camera in `crab_hex.usd` updated to match the real ZED mount within documented tolerance; FOV, near/far, and resolution recorded.
- [ ] Command-parity test data + plot for one representative DOF; tolerance documented.
- [ ] Teacher and student retrained; weights and training-run metadata committed; eval metrics on M2 in-distribution scenarios within ballpark.