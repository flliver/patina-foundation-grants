# Task 1 — Leg Structure (CNC Plywood, Linear Actuators, Shoulder Bolt + Bearing Joints)

**Milestone:** M12 – Krabby-Uno V0.2 Build
**Estimated effort:** ~1 week (part-time, one person + AI assist for CAD iteration)

---

## Narrative

V0.1 legs were hand-cut from dimensional lumber (2×2, 2×4). Joint holes were drilled by hand and nothing was truly interchangeable. V0.2 replaces all of that with CNC-routed 13/16″ plywood parts where every hole, channel, and profile is cut by the machine. The goal is six identical, field-swappable legs that bolt onto the body via two hinge pins each.

This task covers everything about the leg **except** the hip yaw motor and slot + cam-follower yaw drive (that's Task 2). Specifically: the three plywood segments (hip, femur, tibia), the shoulder-bolt + 6201-bearing joint stacks at the knee and hip pivots, the 30–60 W linear actuators that drive those pivots, detachable-leg hinge pins, CNC hole alignment, and wire channels.

Each leg has three segments (hip, femur, tibia) connected by shoulder-bolt + 6201-bearing pivots (BOM Design A). The linear actuators (30–60 W, 8″ stroke, ~20 mm/s, with hall encoder or linear pot) attach directly to the shoulder bolts via clevis ends — no separate bracket needed.

---

## Subtasks

### 1.1 — CAD the leg segments

**Deliverable:** Full parametric CAD (Fusion 360 or FreeCAD) of one leg's three segments with all bearing pockets, shoulder-bolt holes, and actuator mount points. (The yaw-motor interface at the body end of the hip is defined in Task 2 and consumed here as a mating reference.)

Requirements:

- **Tibia:** ~32″ long, ~5″ wide, tapered to 2" at 'toe', 13/16″ plywood, rounded-trapezoid profile (per BOM §1). Bearing pocket for knee joint at the top. Actuator clevis mount (shoulder bolt) near top, ~5" from joint shoulder bolt. Heavy duty rubber machinery foot at end of 'toe'.
- **Femur:** ~27″ long, ~5″ wide, 13/16″ plywood, rounded-rectangle profile. Bearing pockets at both ends (knee and hip joints). Two actuator clevis mounts — knee actuator at the lower end, hip actuator at the upper end.
- **Hip:** ~24″ long, ~5″ wide, 13/16″ plywood, rounded-rectangle profile. Bearing pocket at the femur end. Hip actuator mount. Body-end interface left as a mating surface / bolt pattern to accept the yaw sub-assembly from Task 2.
- **All segments:** 6201 bearing pockets sized for press-fit and/or epoxy (standard 6201: 12 mm ID, 32 mm OD, 10 mm width) Cam follower is also a potential better option (see yaw doc for link). Shoulder-bolt through-holes at 12 mm (h7 tolerance on the bolt, slight clearance in the plywood).
- **Parametric:** Key dimensions (segment lengths, bearing pocket diameter, bolt spacing) should be parameters so they can be adjusted for V0.3 without re-drawing.

### 1.2 — Shoulder-bolt + bearing joint stack design

**Deliverable:** Detailed joint-stack drawing / sub-assembly for knee and hip pivots, with tolerances and BOM line items.

The joint stack follows BOM Design A:

> ~12 mm × ~3″ shoulder bolt → washer → **6201** bearing (press-fit / epoxy into plywood) → 13/16″ plywood → **3/4″ spacer** (4″ HDPE disc or 13/16″ plywood, waxed) → 13/16″ plywood → washer → Belleville washer → lock nut.

Requirements:

- Produce a cross-section drawing of the full stack with dimensions and part numbers.
- Specify bearing press-fit pocket depth, diameter, and surface-finish requirements for the CNC program.
- Specify spacer material and thickness options (HDPE disc vs. waxed plywood). Include both in the CAD so either can be tried.
- Specify Belleville washer size and preload range (target: just enough to eliminate axial play without adding drag).
- Verify that the linear actuator's clevis pin bore aligns with the shoulder-bolt centerline and that the clevis can swing through the full joint range of motion without interference.
- Document the stack in a table suitable for appending to the BOM.

### 1.3 — Linear actuator mounting

**Deliverable:** Actuator mounting geometry integrated into each leg segment's CAD.

Requirements:

- Each linear actuator (30–60 W, 8″ stroke, clevis/pin ends) mounts between two shoulder bolts: one on the "upstream" segment, one on the "downstream" segment.
- The actuator's clevis pin rides directly on the shoulder bolt — the same bolt that serves as the joint pivot. This eliminates a separate bracket and keeps the load path through the bearing.
- Verify range of motion: at full extension and full retraction, the actuator body does not collide with the plywood segment or any other hardware. Model the actuator as a solid body in the CAD assembly and check clearances.
- Specify clevis-pin bore size and any bushing or washer needed between the clevis and the shoulder bolt.
- If the actuator has a fixed-end bracket (motor end), add CNC-drilled mounting holes for it in the plywood.

### 1.4 — Detachable legs via hinge-pin removal

**Deliverable:** Hinge-pin mechanism that allows any leg to be removed from the body by pulling two pins (no tools beyond fingers or a drift punch).

Requirements:

- Each leg attaches to the body at **a single point**: the two door hinges used for the yaw axis (top, vertical — defined in Task 2)
- When both pins are pulled, the leg lifts away from the body as a complete unit (hip + femur + tibia + actuators + motor wires). The hip flange slides out of the body, and the 3d printed/machined yaw slide linkage lifts off the cam follower. The wiring disconnects via the micro-fit molex + 4-pin JST for each of the two linear actuators (per M8 design) accessible from the body interior.
- Legs should be interchangeable and fit any position.

### 1.5 — Pre-drilled alignment for swappability

**Deliverable:** CNC program that drills identical shoulder-bolt, bearing-pocket, actuator-mount, and hinge-pin holes on as many leg pieces as possible from one 4x4 or 4x8 plywood sheet, guaranteeing part-to-part consistency.

Requirements:

- All hole patterns are part of the CNC toolpath — no hand-drilling at assembly time.
- **Bearing pockets:** Routed to depth (10 mm for a 6201) with a flat-bottom end mill. Diameter tolerance: bearing OD + 0.00 / −0.05 mm for a light press fit.
- **Shoulder-bolt through-holes:** 12.1 mm (0.1 mm clearance on a 12 mm bolt) drilled through.
- **Actuator mount holes:** Match the clevis-pin diameter of the chosen linear actuator (typically 8 or 10 mm). Add CNC-drilled holes for actuator body mounting bolts if the actuator has a fixed-end bracket.
- **Test fit:** After cutting the first leg, dry-assemble all joints with bearings, shoulder bolts, and actuators. Measure pin-to-pin distances and compare to CAD. Tolerance target: ≤ 0.5 mm cumulative error across the full leg.

### 1.6 — Wire channels and pilot holes

**Deliverable:** Routed wire channels and pre-drilled pilot holes integrated into the CNC toolpath for each leg segment.

Requirements:

- **Wire channels:** ~1/4″ deep × ~3/8″ wide routed grooves on the **inboard face** of each segment, running from the actuator connection point toward the hip / body connector. Channels should follow the segment centerline and curve smoothly at corners (no sharp 90° turns that pinch wires).
- **Pilot holes for wire holders:** Pre-drill ~1/16″ pilot holes at ~4″ spacing along each wire channel for screw-in cable clips.
- **Strain relief at joints:** Identify appropriate routing for motor wires (especialy knee actuator) to properly extend/retract without putting strain on wire or getting loose.
- **Labeling:** If the CNC can engrave, lightly engrave segment names ("HIP", "FEM", "TIB") on the outboard face of each part. Otherwise, laser-mark or stencil after cutting.

---

## Acceptance Criteria

Task 1 is accepted when:

1. Parametric CAD files for all three leg segments are committed to `krabby-research/hardware/cad/v0.2/legs/`.
2. Joint-stack cross-section drawing with full dimensions and part numbers is included in the CAD package.
3. CNC toolpath files (G-code or equivalent) for all three segments are committed to `krabby-research/hardware/cnc/v0.2/`.
4. CNC toolpath files for all leg segments are placed onto 4x4 or 4x8 plywood sheet and full six legs can be cut in one run.
4. At least one set of segments (1 leg) is CNC-cut and dry-assembled with bearings, shoulder bolts, and actuators — dimensional check passes (≤ 0.5 mm cumulative).
5. BOM is updated with final shoulder-bolt, bearing, spacer, Belleville washer, and actuator model numbers / quantities.
