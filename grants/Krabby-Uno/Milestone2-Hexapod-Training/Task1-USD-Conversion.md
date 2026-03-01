# Task 1 — Robot asset and USD conversion

Goal: Import the **realistic Blender model** from `krabby-research/assets` as **USD**; the **USD is the authoritative copy** for training and evaluation. The existing `crab_hex.urdf` in `krabby-research/assets` is a **simple reference for initial tests/training** only. Convert the Blender model to USD with proper prismatic actuator constraints. Validate full range of motion, collision detection, joint limits, and physical properties.

Outputs
- `krabby-research/assets/crab_hex.usd` (authoritative copy) with complete robot definition, produced from the Blender model in `krabby-research/assets`
- Validation script demonstrating joint motion and collision
- Documentation of joint limits, mass/inertia, and prismatic constraints; Blender file location in `krabby-research/assets` and export steps documented
- Video/screenshots showing full range of motion

Acceptance Criteria
- Robot loads in Isaac Sim with all joints functional
- Prismatic actuator properly shortens/lengthens, moving tibia as expected
- All joint stops are set and enforced
- Collision detection works between legs
- Mass, inertia, and friction properties are set on all links
- Video/screenshots showing full range of motion
- Authoritative USD produced from Blender model in `krabby-research/assets`; Blender→USD path and export steps documented

---

**NOTE**: All code examples in this document are design guidance, not production-ready code. Actual implementation may require adjustments based on Isaac Sim API versions and specific requirements.

---

## 1. Asset options

**Asset locations and authority:**
- **Blender model**: In **`krabby-research/assets`**. Realistic source; **goal is to import as USD**; **USD is the authoritative copy** for training and evaluation.
- **URDF**: `krabby-research/assets/crab_hex.urdf` is a **simple reference for initial tests/training** only. Not the authoritative definition.

**Primary deliverable**: Produce `krabby-research/assets/crab_hex.usd` from the Blender model in `krabby-research/assets`, with prismatic constraints and physics. Choose one Blender→USD path (Section 2), document it, and add a short Blender→USD subsection to the task deliverable.

### Blender model already includes hip/femur and refinement
The **current Blender file** in `krabby-research/assets` already has the **hip/femur prismatic actuators**, **support structure** (triangular bracket, 3" lever arm), and **refined geometry** (e.g. cylindrical leg segments). There is no separate task for these—ensure the Blender→USD export **preserves** them so the authoritative USD has both knee and hip prismatic actuators and the full range of motion. Validate in Isaac Sim that both actuator systems work (range of motion, no collisions).

### Known issue: hip yaw joints in Blender
In the **current Blender file** (in `krabby-research/assets`), the **hip yaw joints snap off** during simulation or manipulation. Part of this task is to **debug why they snap off** after converting to USD—the hope is that it is a Blender physics/rigging quirk that does not carry over to the USD, but this must be verified. Document findings and any fixes (e.g. joint limits, constraint setup, or USD-side fixes).

---

## 2. Blender → USD conversion

Research and document the best path. Options:

### Option A – Blender built-in USD export
- Blender 4.x: **File → Export → Universal Scene Description**.
- Import the resulting USD into Isaac Sim and add physics (rigid bodies, articulation, joint drives) as needed. Prismatic closed-loop linkages will need to be added in USD (Section 3).

### Option B – NVIDIA Blender Alpha USD branch (Omniverse)
- Use the **Blender Alpha USD** build with Omniverse Connector for better compatibility with Omniverse/Isaac Sim.
- Resources: [NVIDIA NGC – Blender USD](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/omniverse/resources/omni_blender), [Omniverse Blender manual](https://docs.omniverse.nvidia.com/connect/latest/blender/manual.html).
- Export via **File → Export → Universal Scene Description** or Nucleus; then add physics and prismatic constraints in Isaac Sim/USD as in Section 3.

### Option C – Blender → URDF → USD
- You can use a **Blender add-on to export to URDF**. The common one for robot models is **Phobos** (https://github.com/dfki-ric/phobos), which supports URDF, SDF, and SMURF. **Caveat:** using Phobos (or similar) typically **requires redoing all the joints as linkages** in the plugin (rigid bodies, joint axes, limits, etc.); the current Blender scene may not map one-to-one. If you choose this path, document the linkage setup and export steps.
- Then import the URDF into Isaac Sim (URDF importer), save as USD, and apply prismatic constraints (Section 3) and other fixes as in the original Task 1.

Implementer chooses one path, documents it in the deliverable, and records the Blender file location in `krabby-research/assets` and export steps. The output USD (e.g. `krabby-research/assets/crab_hex.usd`) is the authoritative copy.

---

## 3. Prismatic actuator constraints (closed-loop linkages)

The crab uses **closed-loop linkages**: the prismatic actuator connects hip to tibia tip. **URDF does not support closed loops**, so this is only a problem when URDF is used as an intermediary (e.g. Option C: Blender → URDF → USD). **Both Blender and USD support closed-loop constraints**, and the **current Blender file in `krabby-research/assets` already has the closed loop defined**. When exporting Blender → USD directly (Option A or B), preserve or verify the closed-loop setup in the export; when going through URDF, you must re-add the constraints in USD after import.

### 3.1 Structure (from URDF or Blender-derived USD)
- `{leg}_knee_actuator_joint` (prismatic): hip → actuator link
- `{leg}_actuator_tip_joint` (fixed): actuator → tibia_top

Actuator end and tibia top must be constrained (e.g. distance joint) so that as the prismatic extends/retracts, the tibia moves correctly. In Blender this is already set up; in USD it must be present (either carried over from Blender export or added manually if the path went through URDF).

### 3.2 Example: add distance constraints in USD (when URDF was intermediary)

If you used the URDF path (Option C), URDF cannot represent the closed loop, so you must add the constraints in USD after import. For each leg, create a distance constraint between the actuator end and the tibia top. Example (path names may differ if using Blender-derived USD; adjust prim paths to match your stage):

```python
# Script to add proper constraints
from pxr import Usd, UsdPhysics, PhysxSchema

stage = Usd.Stage.Open('crab_hex_initial.usd')

# For each leg (lf, lm, lr, rf, rm, rr)
for leg in ['lf', 'lm', 'lr', 'rf', 'rm', 'rr']:
    tibia_top_path = f'/crab_hex/{leg}_tibia_top'
    actuator_path = f'/crab_hex/{leg}_knee_actuator'

    constraint_path = f'/crab_hex/{leg}_actuator_constraint'
    constraint = UsdPhysics.DistanceJoint.Define(stage, constraint_path)

    constraint.CreateBody0Rel().SetTargets([actuator_path])
    constraint.CreateBody1Rel().SetTargets([tibia_top_path])

    constraint.CreateMinDistanceAttr(0.0)
    constraint.CreateMaxDistanceAttr(0.01)  # Small tolerance

    constraint.CreateBreakForceAttr(1e10)  # Very high, effectively unbreakable

stage.Save()
```

### 3.3 Reference
- NVIDIA guide for closed-loop structures: https://docs.isaacsim.omniverse.nvidia.com/4.5.0/robot_setup/rig_closed_loop_structures.html

**Deliverable:** If you went Blender → USD directly, document how the closed loop was preserved (or re-created) in the export. If you went through URDF, include a concrete example (snippet or script) showing the per-leg constraint setup you added in USD so others can replicate.

---

## 4. URDF to USD conversion (reference only)

### Step 1: Initial import
Example import (run from Isaac Sim Python environment):

```bash
cd krabby-research/assets
python -c "
from isaacsim import SimulationApp
simulation_app = SimulationApp({'headless': False})
from omni.importer.urdf import _urdf
import omni.kit.commands

omni.kit.commands.execute(
    'URDFParseAndImportFile',
    urdf_path='crab_hex.urdf',
    import_config=_urdf.ImportConfig(
        set_default_drive_type=_urdf.UrdfJointTargetType.JOINT_DRIVE_POSITION,
        default_drive_strength=1000.0,
        default_position_drive_damping=100.0,
        merge_fixed_joints=False,
        fix_base=False,
        import_inertia_tensor=True,
        distance_scale=1.0
    )
)
import omni.usd
stage = omni.usd.get_context().get_stage()
stage.Export('crab_hex_initial.usd')
simulation_app.close()
"
```

The URDF in `krabby-research/assets/crab_hex.urdf` is a simple reference for initial tests/training; the authoritative robot is the USD produced from the Blender model. If you use the URDF path for a quick prototype, still produce the final authoritative USD from the Blender model. Apply Section 3 (prismatic constraints) and Sections 5–6 (joint limits, mass, friction, collision) to the resulting USD.

### Step 2: Joint limits example
```python
from pxr import Usd, UsdPhysics

stage = Usd.Stage.Open('crab_hex_initial.usd')

joint_limits = {
    'hip_yaw': (-1.5, 1.5),
    'hip_pitch': (-1.0, 1.0),
    'knee': (-2.0, 0.0),
    'knee_actuator_joint': (0.3, 1.2)
}

for leg in ['lf', 'lm', 'lr', 'rf', 'rm', 'rr']:
    for joint_type, (lower, upper) in joint_limits.items():
        joint_path = f'/crab_hex/{leg}_{joint_type}'
        joint_prim = stage.GetPrimAtPath(joint_path)
        if joint_prim:
            joint = UsdPhysics.Joint(joint_prim)
            if 'actuator' in joint_type:
                joint.CreateLowerLimitAttr(lower)
                joint.CreateUpperLimitAttr(upper)
            else:
                joint.CreateLowerLimitAttr(lower * 180.0 / 3.14159)
                joint.CreateUpperLimitAttr(upper * 180.0 / 3.14159)
stage.Save()
```

---

## 5. Physical properties (mass, inertia, friction)

Set mass and inertia on all links (values should match your design; example below). Set friction and contact properties so feet have higher friction than body. Use Isaac Sim MassAPI and CollisionAPI (see Isaac Sim docs). Document the values and reasoning in `crab_hex_usd_specs.md` (or equivalent).

---

## 6. Validation tests

Provide a validation script (e.g. pytest) that:
- Loads the robot in Isaac Sim
- Moves joints (including prismatic) through range
- Verifies tibia moves with actuator
- Checks joint limits and collision between legs
- Reports pass/fail

Example structure: load `crab_hex.usd`, drive joints, step simulation, assert positions/contacts as expected. Tests should cover: joint count (e.g. 30 for 6 legs × 5 joints), prismatic range, actuator-moves-tibia, joint limits, stability, collision, and mass/inertia.

---

## 7. Expected outputs and docs

### File structure
- **`krabby-research/assets/crab_hex.usd`** — authoritative copy, produced from the Blender model in `krabby-research/assets`
- Blender source: `krabby-research/assets/` (Blender file location documented in deliverable)
- Validation script (e.g. `validate_crab_usd.py` or under `tests/`)
- `crab_hex_usd_specs.md`: joint limits, mass/inertia, prismatic constraints, tooling, how to test range of motion; Blender file location in `krabby-research/assets` and export steps

### Success criteria checklist
- [ ] `crab_hex.usd` loads in Isaac Sim with all joints functional
- [ ] Prismatic actuators extend/retract and move tibia as expected
- [ ] Joint limits enforced; collision between legs works; mass/friction set
- [ ] Validation tests pass
- [ ] Prismatic constraint example included in deliverable
- [ ] Authoritative USD from Blender model in `krabby-research/assets`; export path and steps documented
