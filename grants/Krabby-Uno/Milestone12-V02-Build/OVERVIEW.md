# Milestone 12 – Krabby-Uno V0.2 Build (Overview)

## Milestone Overview

Design, CNC, and assemble a **V0.2 Krabby-Uno hexapod** that replaces the V0.1 prototype's dimensional lumber legs and ad-hoc body with:

- **CNC-cut plywood legs** (all segments routed on a wood CNC from 13/16″ plywood),
- **Upgraded linear actuators** (24 V, 30–60 W, 8″ travel, ~20 mm/s, with hall encoder or linear potentiometer feedback) for knee and hip joints,
- **120 W brushed planetary-gear motors** (Xingli Z5D120-24, 24 V, 90 mm frame, ~441 mN·m base torque, 1:100 gearhead → ~20 N·m / ~30 RPM output, encoder included) for hip yaw, driving a **slot + cam-follower** linkage,
- **Redesigned yaw hinge** built around a continuously-rotating crank + pin riding in a slot cut into the hip arm (replaces both the V0.1 linear-actuator yaw concept *and* the earlier V0.2 spur-gear concept — see Task 2 appendix for rationale),
- **Shoulder-bolt + 6201-bearing joint stacks** (Design A from the BOM) with actuators mounted directly to shoulder bolts (no separate bracket),
- A **CNC-cut plywood body** with diagonal plywood corner braces, organized compartments, and a hinged trunk-style lid,
- **3D-printed enclosures** for the 6× H-bridge boards and 3× Arduino Mega + shield stacks (from M8 PCB design),
- The existing **V0.1 electronics stack**: 3× Arduino Mega + Krabby-Uno shield + 6× H-bridge power boards, Seeed J4012 Jetson Orin, and 2× 100 Ah 12 V LiFePO₄ batteries (series → 24 V).

The BOM lives in `krabby-research/hardware/diagrams/BOM.md` and is the authoritative parts list; this milestone references it throughout.

---

## Why is this Important?

V0.1 proved the electrical and control architecture works. V0.2 solves two problems: **speed** and **scale**.

**Speed — faster walking.** V0.1's linear-actuator yaw was too slow for a useful gait. The 120 W planetary-gear motor spinning a crank + slot linkage gives the yaw axis a continuous, self-smoothing sweep of the leg — one motor revolution per full back-and-forth stride — without having to reverse the motor direction every half-stride. See gearing analysis below for the stride-speed derivation. V0.2 is a big step up in gait speed over V0.1; hitting sustained human-pace walking is a stretch goal and the explicit target of later milestones (including the BLDC yaw experiment in Milestone 13).

**Scale — 5 robots to the design/build team in 2 months.** V0.1 was a one-off prototype with hand-cut parts that can't be reproduced. V0.2 is designed so that a single person can **assemble one complete robot in under 4 hours** from pre-cut and pre-printed parts. That's the target that lets a small team produce a fleet of 5 units on a compressed schedule.

### Target assembly time breakdown (< 4 hours, one person)

Assumes CNC parts are batch-cut and 3D-printed enclosures are pre-printed.

| Step | Estimated time | Notes |
|------|---------------|-------|
| Assemble 6 legs (press bearings, shoulder bolts, actuators, route wiring) | 90 min | ~15 min per leg |
| Assemble body frame (panels, corner braces, lateral braces, dividers) | 25 min | All pre-cut, screw assembly |
| Mount electronics into body (enclosures, bus bar, breaker, Orin bracket) | 20 min | Snap-in enclosures, screw-down bus bar |
| Route power wiring (2 AWG battery leads, 10 AWG to H-bridges) | 15 min | Pre-crimped lugs and Mega-Fit connectors |
| Route signal wiring (DB-25 cables, serial JSTs) | 15 min | Pre-made cables |
| Install batteries + tie-downs | 10 min | Drop in, strap, connect 2 AWG |
| Attach 6 legs to body (hinge pins + DB-25 connect) | 18 min | ~3 min per leg |
| Lid, latches, final cable dress | 10 min | |
| Power-on check + standing test | 15 min | |
| **Total** | **~218 min (~3.6 hr)** | **Target: < 4 hours** |

### Other benefits

- **Repeatable leg geometry:** CNC-cut plywood replaces hand-cut dimensional lumber. Every leg is identical, bolt holes line up, and a broken leg can be re-cut and swapped in minutes.
- **Faster iteration:** Detachable legs (2× hinge-pin removal), pre-drilled alignment holes, and wire channels make field repair practical.
- **Organized body:** Battery bays, controller mounts, and a trunk-style lid turn the robot from a wiring nest into something that can be transported and demoed.

---

## Motor & Actuator Specifications

### Linear actuators — knee and hip joints (×12, 2 per leg)

| Parameter | Value |
|-----------|-------|
| Type | Brushed DC linear actuator |
| Voltage | 24 V DC |
| Power | 30–60 W (sourcing both; evaluate on bench) |
| Gear / drive | Internal leadscrew |
| Output speed | ~20 mm/s |
| Output torque / force | TBD from supplier (request rated push/pull force) |
| Current | TBD from supplier (no-load, load, stall) |
| Stroke | 8″ (~200 mm) |
| Feedback | Hall encoder **or** 10 kΩ linear potentiometer (5-wire) |
| Shaft / mounting | Clevis / pin ends; attach directly to shoulder bolt (no bracket) |
| Source | [Alibaba — mini linear actuator w/ position feedback](https://www.alibaba.com/product-detail/Position-Feedback-Dc-Motor-Electric-Mini_60591353891.html?spm=a2756.trade-list-buyer.0.0.750976e9CyczCq) |

### Planetary-gear motors — hip yaw (×6 + spares, 1 per leg)

**Main motor (production):** Zhejiang Xingli **Z5D120-24 + 5GU 1:100 gearhead + encoder**. 8 units ordered at $108/unit (6 legs + 2 spares), procured through Allen Hu (Xingli).

| Parameter | Value |
|-----------|-------|
| Type | Brushed DC with planetary gearhead |
| Voltage | 24 V DC |
| Power | 120 W |
| Frame | 90 mm diameter (2.2 kg motor + gearhead) |
| Base motor torque | ~441 mN·m (before reduction) |
| Gearhead ratio | 1:100 (5GU series) |
| Output speed | ~30 RPM rated (Xingli's technician-recommended 3000 RPM winding through 1:100) |
| Output torque | ~20 N·m at gearhead cap (rated continuous) |
| Current | ~5 A rated; no-load / load / stall TBD on bench |
| Stroke | N/A (continuous rotation) |
| Feedback | Built-in quadrature encoder on motor rear shaft |
| Shaft / mounting | Hollow-bore (parallel-shaft) output — robot provides a through-shaft that keys into the gearhead bore. Exact bore diameter and key dimensions to be confirmed with Allen during Task 2. |
| Brush life | ~2000 hr (standard Xingli rating) |
| Lifetime note | Yaw is the highest-wear axis on the robot. The brushed motor is the production choice for M12 because it drops straight into the existing H-bridge architecture; M13 evaluates a BLDC replacement for long-term service life (see M13 Task 1). |

**Experimental variants (1 each, for M13 hardware experiments — see [Milestone 13 V0.2 hardware experiments](../Milestone13-V02-Hardware-Experiments/OVERVIEW.md)):**

| Qty | Variant | Purpose |
|-----|---------|---------|
| 1 | Xingli Z5BLD120 brushless (90 mm, 458 mN·m base, 2500 RPM) + matched 5GU gearhead + ZBLD.C20-120L2R driver | Full BLDC yaw drive for M13 Task 1 comparison (life, efficiency, driver integration) |
| 1 | Higher-RPM brushed (Xingli 90 W at ~1800–3000 RPM with lower reduction) | A/B speed comparison vs. the main brushed motor |
| — | Forto / Kstone / Mindong bare motors already on hand | Used for M13 Task 2 (DIY linear actuator experiments), not yaw |

---

## Kinematics — Slot + Cam-Follower Yaw Drive

### Goal

Sweep each leg through a **~60°** yaw arc per stride (±30° from center). The motor runs continuously in one direction for normal forward walking; one motor revolution = one complete leg oscillation (forward swing + return). The direction of the motor is only reversed to change the robot's overall walking direction — not every half stride.

### Mechanism

```
Motor output shaft (~30 RPM, continuous rotation)
  → Crank disc (rigid, keyed to output shaft)
    → Cam-follower pin (bolted to crank at radius r_c from shaft center)
      → Rides in a straight slot cut into the hip yaw arm
        → Hip yaw arm oscillates about the yaw axis (at distance d from motor shaft)
          → Leg swings ±30° about the yaw pivot
```

This is a **crank + slotted-lever (Whitworth-style)** mechanism. As the motor rotates at constant ω, the pin traces a circle of radius r_c around the motor shaft. The pin is constrained to stay inside the slot on the hip arm, so the hip arm is forced to rotate back and forth about its yaw pivot. The angular amplitude of the hip arm is set by r_c / d (crank radius relative to the pivot-to-shaft center distance). Natural sinusoidal-like velocity profile — the leg decelerates smoothly to zero at each end of travel and accelerates through the middle of the stroke — no commanded-stop needed.

Exact geometry (r_c, d, slot length, follower bearing size) is designed in Task 2.1; the numbers below assume a symmetric (non-quick-return) layout tuned for ±30° sweep.

### Stride geometry

- **Femur length:** 28″ = 0.711 m
- **Yaw sweep per stride:** 60° (±30° from center)
- **Forward displacement per stride (foot path arc approximation):** 2 × L × sin(30°) = 2 × 0.711 × 0.5 = **0.711 m**
- **Motor revolutions per stride:** 1 (one full rotation = one complete oscillation of the hip arm)
- **Stride frequency:** Motor RPM / 60 = (30 / 60) = **0.5 strides/s** at rated output speed
- **Nominal walking speed:** 0.711 m × 0.5 strides/s = **~0.36 m/s** (~0.8 mph)

Max walking speed scales directly with motor output RPM. Pulling above rated load slows the motor down (and eventually stalls the gearhead at ~20 N·m), so worst-case speed is whatever the motor actually delivers under the leg's swing inertia + stance load.

### Yaw torque

Yaw torque at the leg varies through the stroke because the mechanical advantage of a slotted-link mechanism is a function of crank angle:

- **At mid-stroke** (pin 90° from the slot axis), follower travels fastest along the slot, so torque at the yaw arm is lowest.
- **Near the ends of travel** (pin aligned with the slot axis), follower velocity along the slot is near zero and the effective lever arm is largest — output torque is maximum.

For a first-order estimate with crank radius r_c and pivot-to-shaft distance d:

- Mean yaw torque ≈ motor output torque × (d / r_c), times efficiency (~0.85 for a greased steel pin on a hardened steel slot insert).
- For d/r_c ≈ 3 (reasonable starting point for a ±30° sweep): mean yaw torque ≈ 20 N·m × 3 × 0.85 ≈ **~51 N·m** at the yaw arm.
- Peak torque at the ends of the stroke can be 2–3× this; peak force on the follower pin is correspondingly higher at mid-stroke where it absorbs the swing inertia.

Task 2.2 nails down r_c, d, and peak-torque / peak-bearing-load numbers from a worst-case leg-inertia model.

### Why this drive mechanism

- **Motor always rotates forward during normal walking.** The H-bridge spends 99% of operating time pushing the motor in one direction. This avoids the constant polarity reversals a spur-gear driven yaw would demand (2 per stride × continuous walking), which would hammer the brushes, the H-bridge MOSFETs, and the gearbox backlash zone.
- **Self-smoothing motion.** The sinusoidal-ish output profile naturally decelerates the leg at the stroke endpoints. No explicit PID trajectory planning needed to avoid slamming the leg into its limits.
- **Compact vs. alternatives.** Less body volume than a four-bar linkage at the same yaw amplitude (four-bar was considered; see [four-bar.txt](four-bar.txt) and the Task 2 appendix).
- **Tolerates the 30 RPM output.** Because the mechanism uses one full motor revolution per stride, a ~30 RPM output shaft produces a plausible ~0.5 Hz stride cadence with no further reduction. This is deliberately conservative for V0.2 — longevity and reliability matter more than top speed until the stress tests in M13 say otherwise.

### Recommendation

Use the main Xingli Z5D120-24 + 1:100 gearhead as the V0.2 yaw drive. Task 2.2 does a full force/torque pass to pick r_c and d. The 1:20 sample variant and the BLDC sample are held for M13 experiments and are **not** required to finish M12 — see [Milestone 13 overview](../Milestone13-V02-Hardware-Experiments/OVERVIEW.md).

---

## Reference: Joint Stack (from BOM — Design A)

Each knee and hip pivot uses the following stack (per BOM §3):

> ~12 mm × ~3″ shoulder bolt → washer → **6201** bearing (press-fit / epoxy into plywood) → 13/16″ plywood → **3/4″ spacer** (4″ HDPE disc or 13/16″ plywood, waxed) → 13/16″ plywood → washer → Belleville washer → lock nut.

Actuators mount directly to the shoulder bolt (clevis end over bolt), eliminating a separate bracket and reducing slop.

---

## Electronics & Power (carried from V0.1 — no redesign)

These items are **not** redesigned in M12; they are assembled into the new body as-is:

| Subsystem | Components | Notes |
|-----------|------------|-------|
| MCU | 3× Arduino Mega 2560 + Krabby-Uno shield (M8 PCB) | Each Mega drives 2 legs (6 actuators + 2 yaw motors) |
| H-bridge | 6× Krabby-Uno H-bridge power boards (M8 PCB) | 1 board per leg; 3 BTS7960 channels per board |
| Connectors | JST XA (pot feedback), Molex Micro-Fit (signal), Molex Mega-Fit (power) | Per BOM §4 |
| Brain | 1× Seeed J4012 Jetson Orin | Main autonomy / fleet computer |
| Batteries | 2× 12 V 100 Ah LiFePO₄, wired in series → 24 V | Per BOM §6 |
| Distribution | 1× marine-style bus bar (boat distribution block, 6+ posts) | Upgraded from Maierke block — [Amazon B0FL28CCC9](https://www.amazon.com/dp/B0FL28CCC9) |
| Protection | 150 A inline circuit breaker on + rail | Per BOM §6 |
| Wiring | 2 AWG battery leads, 10 AWG to H-bridges, ring lugs | Per BOM §6 |

### 3D-printed enclosures (new in M12)

| Qty | Enclosure | Key features |
|-----|-----------|--------------|
| 6 | H-bridge board case | Snap lid, M3 standoffs, cutouts for ~10 LEDs, 1× DB-25, 3× Micro-Fit, 3× JST XA 4-pin, 1× Mega-Fit; open rear for heatsink |
| 3 | Arduino Mega + shield case | Snap lid, cutouts for 2× DB-25, USB, barrel jack, 2× 3-pin JST serial, power LED |

---

## Tasks

Tasks are **sequential** — each builds on the prior week's output. Target ~1 week per task.

| Task | Week | Summary | Doc |
|------|------|---------|-----|
| **Task 1** | 1 | **Leg structure** — CAD all three leg segments for CNC, design shoulder-bolt + 6201-bearing joint stacks, linear-actuator mounting, detachable hinge pins, pre-drilled alignment, wire channels. Dry-fit one leg. | [TASK-1-LEG-STRUCTURE.md](TASK-1-LEG-STRUCTURE.md) |
| **Task 2** | 2 | **Yaw motor + slot/cam-follower hinge** — CAD the yaw sub-assembly (Xingli 120 W motor, crank + cam-follower pin, slotted hip arm, yaw bearings, motor bracket), force/kinematics analysis, hardware selection, motor wiring/encoder interface, single-yaw bench test. | [TASK-2-YAW-MOTOR.md](TASK-2-YAW-MOTOR.md) |
| **Task 3** | 3 | **Body redesign** — CNC plywood body (all panels, braces, dividers, trunk-style lid), organized compartments (batteries, controllers, Orin, yaw motors), 3D-printed electronics enclosures for H-bridge and Mega boards. | [TASK-3-BODY-REDESIGN.md](TASK-3-BODY-REDESIGN.md) |
| **Task 4** | 4 | **Full V0.2 assembly & test** — Production run of 6 legs, body assembly + wiring, power-on smoke test, single-leg integration, six-leg standing test, documentation. | [TASK-4-ASSEMBLY-AND-TEST.md](TASK-4-ASSEMBLY-AND-TEST.md) |

**Estimated duration:** ~4 weeks for one person (part-time with AI assistance), ~1 week per task, sequential.

---

## Dependencies

| Dependency | Status | Notes |
|------------|--------|-------|
| **BOM finalized** | In progress | `krabby-research/hardware/diagrams/BOM.md` — linear actuator and planetary-gear motor sourcing underway (Alibaba quotes sent) |
| **M8 PCBs fabricated** | Required | Arduino Mega shield + H-bridge boards must be in hand for bench test (Task 2) and body mount design (Task 3) |
| **Wood CNC access** | Required | Need a CNC router that can cut 13/16″ plywood panels up to ~36″ long |
| **3D printer access** | Required | FDM printer for electronics enclosures; build volume ≥ 250 mm in each axis |
| **Linear actuator samples** | Required | At least 2 units for single-leg bench test before committing to 12-unit order |
| **Xingli yaw motors** | Ordered (8 units at $108) | On the water from Allen Hu; at least 1 unit needed before Task 2.4 bench test. Experimental BLDC + high-RPM brushed variants are shelved for M13 and are not required for M12 acceptance. |
| **Cam-follower hardware** | Required | Hardened-steel pin or needle-bearing cam follower, slot-insert material (hardened flat stock or bronze) — sized in Task 2.2 |
| **Shoulder bolts, 6201 bearings, hardware** | Required | Per BOM §3 — can order immediately |

---

## Repos and Artifacts

Artifact locations are **TBD** — the following are preferred / example locations:

| Artifact | Preferred location |
|----------|--------------------|
| BOM (authoritative) | `krabby-research/hardware/diagrams/BOM.md` |
| Leg CAD files (DXF/SVG for CNC, STEP/F3D for assembly) | `krabby-research/hardware/cad/v0.2/legs/` |
| Yaw sub-assembly CAD | `krabby-research/hardware/cad/v0.2/legs/yaw/` |
| Body CAD files | `krabby-research/hardware/cad/v0.2/body/` |
| Enclosure STLs (H-bridge, Mega) | `krabby-research/hardware/cad/v0.2/enclosures/` |
| CNC toolpath files / G-code | `krabby-research/hardware/cnc/v0.2/` |
| Milestone contract (ICA) | [krabby-contracts/milestones/M12/M12.md](https://github.com/flliver/krabby-contracts/blob/main/milestones/M12/M12.md) |
| Grant overview & task specs (this milestone) | [Milestone12-V02-Build on GitHub](https://github.com/flliver/patina-foundation-grants/tree/main/grants/Krabby-Uno/Milestone12-V02-Build) |
| Assembly photos / bench test video | `krabby-research/hardware/docs/v0.2/` |

---

## Acceptance (high-level)

M12 is complete when:

1. **Leg structure proven (Task 1):** Parametric CAD for all three segments, CNC toolpaths, and at least one dry-fit leg with bearings, shoulder bolts, and linear actuators passing dimensional checks.
2. **Yaw motor proven (Task 2):** Yaw sub-assembly CAD with crank + slot geometry selected, wiring diagram, and one yaw joint passing a bench test (encoder tracking, full ±30° sweep at target speed, motor temperature within spec).
3. **Body built (Task 3):** CNC plywood panels assembled into a body frame, with all electronic enclosures printed and test-fit, and the trunk-style lid functioning.
4. **Six legs produced (Task 4):** All six legs CNC-cut, assembled, QC-inspected, and confirmed interchangeable.
5. **Robot assembled and standing (Task 4):** Body fully wired with all electronics, six legs attached, power-on smoke test passed, and the robot stands under its own power on a flat surface with all 18 joints (12 linear + 6 yaw) holding position for ≥ 60 seconds.
6. All CAD files, CNC toolpaths, STLs, and assembly docs are committed to `krabby-research`.
