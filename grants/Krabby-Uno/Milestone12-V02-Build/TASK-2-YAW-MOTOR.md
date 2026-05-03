# Task 2 — Yaw Motor + Slot / Cam-Follower Hinge Design

**Milestone:** M12 – Krabby-Uno V0.2 Build
**Estimated effort:** ~1 week (part-time; builds on Task 1 leg segment CAD)

---

## Narrative

V0.2 drives hip yaw with a **Xingli Z5D120-24** brushed motor (120 W, 90 mm frame, 1:100 gearhead, encoder, ~441 mN·m base torque → ~20 N·m and ~30 RPM at the output). The motor is bolted to the inside of the body cavity using the provided L-bracket. Utilize an additional plywood separator as needed for proper motor position. Its output shaft carries a **crank disc with a cam-follower pin**. The pin rides in a **3D-printed and/or machined slot linkage that is bolted to the hip flange** (see diagrams if needed). One full motor revolution drives the leg through a complete forward-and-back oscillation (~60° total sweep, ±30° from center). The motor runs continuously in one direction during normal walking — the slot takes care of converting rotation into oscillation.

See the **Kinematics — Slot + Cam-Follower Yaw Drive** section in OVERVIEW.md for the motion derivation and nominal stride-speed numbers. Wiring + encoder integration and single-yaw bench testing are handled as part of the existing M8 / M12 Task 4 work; this task is about the **mechanical sub-assembly only**.

---

## Subtasks

### 2.1 — Hip yaw flange

**Deliverable:** CAD for the hip yaw flange — the mating piece on the leg that the slot linkage bolts to. The flange rotates with the hip segment about the yaw axis and is what transfers torque from the cam follower into the leg.

- Simple CNC plywood or aluminum disc/plate that bolts to the top of the hip segment.
- Pattern of holes on the flange face that the slot linkage (2.2) bolts onto.
- Centered bore for the yaw pivot pin / bearing stack.

### 2.2 — 3D-printable slot linkage CAD

**Deliverable:** CAD of the slotted arm that bolts to the hip yaw flange, printable on a standard FDM printer.

- Straight, parallel-sided slot sized for the sourced cam follower (2.3). Slot length accommodates the full ±30° sweep with margin.
- Bolt pattern matching the hip yaw flange (2.1).
- Printable in PETG or ABS without supports if possible. If the print won't stand up to load in bench testing, note that the part is a candidate for machining in aluminum for V0.3 — but ship V0.2 with the printed version.

### 2.3 — Source motor crank + cam follower

**Deliverable:** Parts list (with supplier links) for the crank disc and cam-follower pin, plus a small CAD drawing showing how the crank keys onto the motor output shaft.

- Crank disc: off-the-shelf keyed hub or a simple machined aluminum disc with a bore matched to the gearhead's hollow-shaft output. Confirm bore diameter and key dimensions with Allen (Xingli) before ordering.
- Cam follower: off-the-shelf needle-roller cam follower (McMaster / Misumi), 8–12 mm OD, bolt-in style. Rolling follower preferred over a plain pin.
- Show the mount-up on the crank with one or two candidate radii (r_c) so the user can pick by swapping bolt holes during first install.

### 2.4 — Rigid motor mount (all three motor variants)

**Deliverable:** CAD for the motor mount inside the body cavity that accepts **any of the three motor variants** in hand (main Xingli brushed Z5D120-24, the higher-RPM brushed variant, and the Xingli Z5BLD120 BLDC). All three are 90 mm frame, so a single universal mount is realistic.

- Uses the L-bracket that ships with the motor plus a plywood separator (CNC-cut) to fix the motor at the correct height and orientation relative to the hip yaw flange (2.1).
- Rigid enough that the crank's reaction force doesn't flex the mount visibly during a full leg-travel cycle. Add gussets or a second tie-point to the body wall if needed.
- Positioned so the full ±30° leg sweep clears the motor body, the body wall, and any other hardware at every point in the cycle — verify in CAD assembly.
- Bolt-swap time between motor variants target: under 10 minutes, no body disassembly.

---

## Acceptance Criteria

Task 2 is accepted when:

1. CAD files for the hip yaw flange, 3D-printable slot linkage, crank + cam-follower arrangement, and motor mount are committed to `krabby-research/hardware/cad/v0.2/legs/yaw/`.
2. Parts list (crank, cam follower, hardware) is added to the BOM.
3. At least one full leg travel cycle is demonstrated on the robot (or on a single-leg fixture) with the main Xingli brushed motor, confirming no interference and the cam follower stays seated in the slot through the full ±30° sweep.

---

## Appendix — Why a slot + cam follower? (spur gear, four-bar, and BLDC considered)

The original V0.2 plan called for a **spur-gear pair** between the motor shaft and the yaw axis. After a few weeks of sourcing, thinking through wear and motor behavior, and ordering the much larger Xingli motor (up from the 10 N·m D63 class originally specified to a 20 N·m, 90 mm class motor), we switched to the slot + cam-follower mechanism. This appendix records why, so the decision is not re-litigated six months from now.

### Alternative 1 — Spur gear (rejected)

**How it would work:** motor output shaft carries a pinion; the hip yaw arm is rigidly fixed to a larger driven gear rotating about the yaw axis. To swing the leg forward 30° then back 30°, the motor has to rotate forward a fraction of a revolution, *stop*, reverse, rotate back, *stop*, reverse, and repeat.

**Why we moved away from it:**

- **Continuous direction reversal kills brushed motors.** At a ~1 Hz stride cadence (a reasonable walking target), the motor changes direction **twice per stride**. Over an 8-hour walking day that's ~60,000 polarity reversals. Each reversal hammers the commutator and the brushes, arcing through the neutral point at full current. The Xingli's rated brush life is ~2000 hours; we'd be consuming that budget much faster than any published figure accounts for, because all published life figures assume steady-state operation.
- **H-bridge stress.** Every reversal is also an H-bridge MOSFET switching event under load. BTS7960s handle this, but continuous reversal at load with an inductive motor load increases MOSFET thermal cycling and EMI.
- **Gearbox wear at the backlash zone.** The gearbox planet carrier crosses through zero-load twice per stride, then reloads in the opposite direction. All the backlash slack travels across the teeth on every transition, producing the "clunk" that eventually turns into pitted gears.
- **Planning complexity.** The controller has to compute and track an explicit trajectory for each stride (accelerate, cruise, decelerate, stop, reverse, ...). A missed deceleration slams the leg into the spur-gear backlash at full motor torque.

### Alternative 2 — Four-bar linkage (rejected — geometry)

Reference: [four-bar.txt](four-bar.txt).

**How it would work:** The motor drives a crank arm (short lever at the motor shaft). A connecting rod ties the crank to a follower arm pivoted at the yaw axis. The three moving bars + the fixed ground link form a four-bar mechanism. One motor revolution produces one full oscillation of the follower — same conceptual win as the slot drive (motor runs one direction, linkage converts rotation to oscillation).

**Why we moved away from it:**

- **It's a great mechanism from an efficiency and simplicity standpoint.** Only pin joints, no sliding contact, very low friction, well-understood kinematics.
- **But it takes up too much body volume.** The connecting rod has to swing through a large horizontal envelope between the motor crank and the hip follower arm. For a ±30° sweep at the hip, the connecting rod wants to live in a horizontal "slice" inside the body that we simply don't have — the body already has to fit two 12 V 100 Ah batteries, three Mega stacks, six H-bridge enclosures, the Orin, and the bus bar (see Task 3 layout). The slot mechanism hides all its motion **inside the slot itself** — the only moving volume is the crank disc and pin, which is much tighter than the swept volume of a connecting rod.
- **Decision:** Four-bar is the right answer if body volume ever gets cheap (V0.3 or later, with a larger body or a differently-shaped hip bay). For V0.2 with the current 28″ × 48″ body, it doesn't fit.

### Alternative 3 — BLDC yaw motor (deferred to M13)

The Xingli brushless Z5BLD120 (458 mN·m base torque, 2500 RPM, 90 mm frame, halls included, ~10,000 hr bearing-limited life) is a strong candidate that would sidestep the brush-life concern entirely. We deliberately chose **not** to adopt it for V0.2 because:

- The BLDC needs a 3-phase driver (Xingli's ZBLD.C20 or a SimpleFOC board) instead of our existing BTS7960 H-bridge. That's a real wiring and control-software change, not a drop-in swap.
- The driver's control interface (PWM/analog vs. RS-485 Modbus RTU) needs bench time to work out — it's a whole additional mini-project.
- V0.2's primary goal is to prove the leg and body designs at reproducible scale. We want to validate the mechanical architecture with the known-working brushed-motor electronics first, then upgrade to BLDC on top of a validated platform.

One BLDC motor + driver is being ordered for M13 **Task 1** (Full BLDC Yaw Comparison), where it will be swapped into one leg and benchmarked against the brushed motors in the other five. If the comparison is favorable, a BLDC conversion becomes the V0.3 yaw spec.

### Why the slot + cam follower wins for V0.2

1. **Motor rotates the same direction 99% of the time.** No per-stride reversals. Direction only changes when the robot decides to walk backwards, which is a much rarer event than the stride cadence. This is the single biggest reason for the switch — it meaningfully extends the brushed motor's life and reduces H-bridge / gearbox stress.
2. **Self-smoothing motion profile.** Sinusoidal-ish output velocity naturally decelerates the leg at each end of travel. No software trajectory planning required to avoid slamming into the stops.
3. **Compact in body volume.** Much tighter than a four-bar; nothing swings through the occupied volume except the small crank disc directly under the motor.
4. **Drops into existing electronics.** Brushed motor + BTS7960 H-bridge + quadrature encoder into the existing Mega shield. Zero code architecture changes vs. V0.1.
5. **Easy to tune.** Crank radius r_c is a single bolt-hole choice. We can start conservative (small r_c → small sweep → low peak loads) and open it up once the slot + follower prove themselves in M13 stress testing.

The main thing we're giving up is **sliding friction at the follower / slot interface** vs. the pure pin-joint four-bar — which is why the CAD calls for a needle-roller cam follower rather than a plain pin. The M13 stress test (weighted leg walking for days) is specifically designed to catch any slot/follower wear issues before they become a fleet problem.
