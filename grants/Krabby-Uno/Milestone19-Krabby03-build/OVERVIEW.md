# Milestone 19 – Krabby 0.3.1 Build Sprint (Overview)

## Milestone Overview

A **9-day build sprint** to finish Krab 0.2 and cut, assemble, and stand up **Krabby 0.3.1**, executed by a two-person teenage build crew with Dad supervising the tool- and software-heavy work.

**Krabby 0.3.1 is the second robot in the 0.3 line** — it follows Krabby 0.3.0 and folds in the geometry fixes learned from it and from Krab 0.2 (flat hip chassis with hinge pockets, longer knee lever arm, relocated actuator mounts). This sprint produces the 0.3.1 unit specifically.

- **Crew:** two 14-year-olds, working as a pair.
- **Schedule:** 9 days, ~4 hours/day — a **solo block from 3–6pm** (kid-only work) and a **Dad block from 6–8pm** (anything needing supervision, power tools, i3Detroit, or software debugging).
- **Dates:** Wednesday through the following Thursday.
- **Goal:** Krab 0.2 walking by mid-sprint; Krabby 0.3.1 CNC-cut, assembled, and standing by Day 9.

This is a **mini-milestone**, scoped at roughly half a normal milestone: a two-kid crew at kid pace over 9 short days. It has no separate task documents — everything lives in this OVERVIEW — but it carries the same rigor (epics, a build guide, a day-by-day plan, dependencies, and acceptance criteria) as a full milestone.

The BOM and CAD referenced here live in `krabby-research`; confirm all quantities and hole positions against the current CAD before drilling or bolting anything permanent.

---

## Why is this Important?

This sprint moves two robots forward at once and trains the build crew on the full fabrication workflow.

- **Krab 0.2 finished and walking.** Swapping in the new hip linear actuators, mounting and wiring the hip yaw motors, and bringing up the ZED2i + side cameras takes 0.2 from a partial teardown to a walking, sensing robot.
- **Krabby 0.3.1 exists in the physical world.** The second 0.3-line robot gets CNC-cut, assembled, and stood up for the first time, carrying the geometry fixes from 0.3.0 and Krab 0.2.
- **The crew learns the whole pipeline.** Over 9 days the kids run teardown, actuator swaps, crimping and wiring, a CNC cut at i3Detroit, and a six-leg production assembly line. By Day 9 they can build a leg module from the guide start to finish.

---

## Crew / Execution

Two 14-year-olds work as a pair. Each day is split into two blocks:

- **Solo block (3–6pm):** kid-only work — assembly, prep, sanding, labeling, dry-fitting, and hand-testing. No power tools, no i3, no live power.
- **Dad block (6–8pm):** anything that needs supervision, power tools, i3Detroit CNC time, live electrical, or software debugging. Dad also runs the teaching moments (lever-arm geometry, crimping, the CNC workflow).

The two blocks are designed so the kids stay productive solo while the risk-bearing and skill-transfer work is reserved for the supervised block.

---

## Epics

### EPIC 1: Disassemble Krab 0.1
- [ ] 1.1 Complete disassembly of robot 0.1 (about half done)
- [ ] 1.2 Sort and label salvaged hardware into bins

### EPIC 2: Finish Krab 0.2
- [ ] 2.1 Replace the 6 hip linear actuators. Unbolt the old ones, bolt on the new ones, transfer the cables.
- [ ] 2.2 Add the ZED2i camera to the Orin and get it visible and running. Work with Claude and Dad.
- [ ] 2.3 Cut the slot for the ZED camera with drill and jigsaw.
- [ ] 2.4 Add the DFRobot side cameras. Cut mount holes in the chassis and mount them.
- [ ] 2.5 Assemble the hip slot boards.
- [ ] 2.6 Assemble the hip lever arms.
- [ ] 2.7 Cut the slot to lower the center hip yaw slot boards and remount the hinges 1 inch lower.
- [ ] 2.8 Mount all hip yaw motors to the robot.
- [ ] 2.9 Test all hip yaw motors.
- [ ] 2.10 Wire up all hip yaw motor crimp connectors.
- [ ] 2.11 Test the controller with the robot. Walk it around and confirm the gait looks good.

### EPIC 3: Design Krabby 0.3.1
- [ ] 3.1 Fix the hip chassis. Add a pocket cut for the hinges and make the side flat instead of concave.
- [ ] 3.2 Move the printed logos to the knee.
- [ ] 3.3 Move the knee bolt down 1.5 inches for a longer lever arm so the knee reaches its end stop.
- [ ] 3.4 Add a hole 1.25 inches from the tip of the knee and the tip of the hip for the actuator bolt mount.
- [ ] 3.5 Add holes along the femur and hip for motor mounts.
- [ ] 3.6 Export final DXF and toolpaths for the AVID CNC.

### EPIC 4: Cut and Assemble Krabby 0.3.1
- [ ] 4.1 Nest the full 0.3.1 layout — every leg and chassis part — onto the 4×8 plywood sheet in Inkscape/VCarve and generate the AVID toolpaths.
- [ ] 4.2 CNC-cut the full 0.3.1 plywood kit at i3Detroit with Dad (weekend cut day).
- [ ] 4.3 Cut all the joint disks and apply Teflon tape to each bearing face.
- [ ] 4.4 Sand and deburr all cut parts and label them.
- [ ] 4.5 Assemble 6 leg modules per the build guide, Steps 1 through 5.
- [ ] 4.6 Assemble the center chassis, Step 6.
- [ ] 4.7 Join the legs to the chassis, Step 7.
- [ ] 4.8 Install actuators and motors, Steps 8 and 9.
- [ ] 4.9 Standing check. All six legs bear load and every joint sweeps its full range by hand.

---

## Krabby 0.3.1 Build Guide

**How the leg works.** Each leg has three joints. The hip yaw swings the whole leg forward and back, driven by a brushed gearmotor pivoting on bearings in the slot boards. The hip pitch raises and lowers the femur, and the knee folds the lower leg. Both are driven by Tr12x3 lead-screw linear actuators pushing on lever arms. The lever arm converts a few inches of screw travel into a large joint sweep. Moving the actuator mount closer to the pivot gives more angle and less force. Moving it farther gives more force and less range. This is why the knee bolt moves down 1.5 inches in 0.3.1: the old geometry ran out of throw before the end stop.

Confirm quantities and hole positions against the CAD before drilling or bolting anything permanent.

### Parts List (per leg, ×6)

| Part | Qty per leg |
|---|---|
| 6201RS bearings | 2 |
| Joint disks, Teflon-taped | per CAD |
| 12mm shoulder bolts with washers and nyloc nuts | per CAD |
| Tr12x3 lead-screw linear actuators | 2 |
| Hip yaw gearmotor with mounting screws | 1 |
| Hinges with wood screws | per CAD |
| Connector pigtails and zip ties | as needed |

Each pivot uses a Teflon-taped joint disk as the low-friction spacer in the bearing stack (replacing the waxed spacer). The joint disks are cut on Day 5 and Teflon-taped during Day 6 prep.

### STEP 1: Slot Board Prep
1. Match the left and right slot boards. They mirror each other, so confirm orientation before assembly.
2. Press the 6201RS bearings into their pockets. Seat each one with a mallet and a block of wood.
3. Dry-fit the shoulder bolt through both bearings. It should spin freely with zero wobble. If the fit is loose or the bolt wobbles, flag it for Dad.

### STEP 2: Hip Yaw Assembly
1. Bolt the gearmotor to its mount plate and tighten the screws snug.
2. Fit the motor output to the yaw pivot. Confirm the coupling detail with Dad.
3. Rotate the yaw joint by hand through its full range. It should sweep smoothly. If it rubs, sand the slot until it clears.

### STEP 3: Femur Assembly
1. Screw the femur side plates to the spacer blocks. Square the assembly with a speed square before driving the screws.
2. Install the knee pivot shoulder bolt and bearings the same way as Step 1.
3. Attach the hip-pitch lever arm at the new bolt hole 1.25 inches from the tip.

### STEP 4: Knee and Lower Leg
1. Bolt the lower leg to the femur at the knee pivot. Fold the knee to its end stop by hand and confirm it reaches. If it binds early, flag it for Dad.
2. Attach the knee lever arm.
3. Fit the foot.

### STEP 5: Actuators On
1. Bolt the actuator body ends to their frame mounts and the rod ends to the lever arms. Leave the actuators unpowered.
2. Check that the joint holds position by hand once both actuator ends are pinned. Lead screws hold their position, so a joint that flops means a pin is unseated.
3. Zip-tie the actuator leads along the femur with slack at the joint. Fold the knee fully by hand and confirm the cable stays slack through the whole motion.

### STEP 6: Center Chassis
1. Assemble the chassis panels per the CAD. The 0.3.1 hinge pockets let the hinges sit flush.
2. Mount the hinges into their pockets.
3. Confirm the hip yaw slot boards sit at the lowered height built into the 0.3.1 design.

### STEP 7: Legs Meet Body
1. One kid holds the leg while the other drives the screws. Attach each leg's slot boards to the chassis hinges.
2. After each leg, swing the yaw by hand and confirm a full sweep with clearance from the chassis.
3. Legs 1-3-5 and 2-4-6 mirror each other. Confirm each leg is on the correct side before bolting.

### STEP 8: Motors and Electrical Rough-In
1. Mount all 6 hip yaw gearmotors if any remain from Step 2.
2. Crimp connectors on the motor leads and tug-test every crimp.
3. Label every connector with leg number and joint, for example L3-YAW.

### STEP 9: Standing Check (with Dad)
1. Set the robot on blocks with the legs hanging free.
2. Hand-sweep every joint through its full range and listen for rubbing or clicking.
3. Lower the robot onto its feet. All six feet should touch flat. If only some feet touch, find the twisted assembly and fix it.
4. Dad signs off before any power-up.

---

## Day-by-Day Plan

Each day has a solo block from 3 to 6pm and a Dad block from 6 to 8pm. Anything that needs supervision, power tools, i3Detroit, or software debugging goes in the Dad block.

### Day 1 (Wed): Teardown Done
- **Solo:** Finish the 0.1 disassembly and sort all salvaged hardware into labeled bins.
- **With Dad:** Review the salvage together and walk through the full 9-day plan. Dad starts the 0.3.1 CAD fixes while the kids watch and learn the CAD workflow.

### Day 2 (Thu): 0.2 Actuators + Yaw Motors Mounted
- **Solo:** Swap all 6 hip linear actuators — one kid unbolts and preps while the other transfers cables. Then assemble the hip slot boards and lever arms and mount all six hip yaw motors to the robot.
- **With Dad:** Torque-check the actuator installs, then test all six hip yaw motors and flag any dead channels. After that, work the 0.3.1 yaw-motor CAD to get the hip/slot geometry sorted — hip-chassis pocket, lowered slot-board height, and motor-mount holes. Dad covers lever-arm geometry and a quick crimping demo along the way, which set up tomorrow's wiring and the build guide.

### Day 3 (Fri): Yaw Wiring + 0.2 Walk Test
- **Solo:** Wire up all six hip yaw motors — strip leads, crimp the connectors, and label every connector with leg number and joint (for example L3-YAW).
- **With Dad:** Tug-test every crimp. Cut the slot to lower the center hip yaw slot boards and remount the hinges 1 inch lower. Bench-test yaw and actuator motion, then run the controller walk test — **Krab 0.2 walking.** Write down anything worth changing in 0.3.1.

### Day 4 (Sat): Full-Sheet Layout Day
- **Solo:** Count and label the hardware for all six legs against the parts list and write a shopping list for anything missing. Print the knee logos.
- **With Dad:** Finalize the 0.3.1 CAD, then nest the entire robot — every leg and chassis part — onto the 4×8 plywood sheet in Inkscape/VCarve. Generate the AVID toolpaths and check the nest for part spacing, hold-down tabs, and cut order. (Weekend day — Dad works the layout with the kids.)

### Day 5 (Sun): Cut Day at i3
- **Solo:** Stage the sheet, clamps, and toolpath files for the i3 trip. Final hardware and shopping check before assembly week.
- **With Dad:** Trip to i3Detroit. CNC-cut the full 0.3.1 plywood kit on the AVID and cut all the joint disks. The kids learn the CNC workflow: sheet clamping, zeroing, and running toolpaths. (Weekend day — Dad goes with them.)

### Day 6 (Mon): Prep + Leg Line Starts
- **Solo:** Sand, deburr, and label all cut parts. Apply Teflon tape to every joint disk. Start leg modules with Steps 1 and 2 on legs 1 through 3.
- **With Dad:** QC the bearing fits and pivots. Dad handles any re-cuts. If ahead of schedule, run Steps 1 and 2 on legs 4 through 6 together.

### Day 7 (Tue): Leg Production Line
- **Solo:** Run Steps 3 and 4 on all six legs. One kid builds femurs while the other builds knees, then swap halfway so both learn both.
- **With Dad:** QC each joint against the build guide and fix any binding tonight. Start the center chassis (Step 6).

### Day 8 (Wed): Actuators, Chassis + 0.2 Cameras
- **Solo:** Run Step 5 (actuators on) across all six legs and confirm each joint holds position by hand.
- **With Dad:** Finish the center chassis (Step 6). Then, on Krab 0.2, cut the ZED camera slot, mount the DFRobot side cameras, and bring up the ZED2i on the Orin.

### Day 9 (Thu): Krabby 0.3.1 Stands
- **Solo:** Finish any remaining leg modules. Run the Step 8 electrical rough-in: crimp and label all motor leads.
- **With Dad:** Attach the legs to the chassis (Step 7), then run the Step 9 standing check and sign-off. Close with a sprint retro: the kids write the punch list for what 0.3.1 needs next.

---

## Buffer Tasks

Pull these in whenever a block finishes early or a dependency is blocked.

- [ ] Inventory spreadsheet of parts to order for 0.3.1 electronics
- [ ] Sand and finish spare parts
- [ ] Mock up logo placement on the knee

---

## Dependencies

| Dependency | Status | Notes |
|------------|--------|-------|
| **Replacement linear actuators on hand** | Confirm before Day 2 | Day 2 (Epic 2.1) cannot start without them |
| **0.3.1 actuators and gearmotors sourced** | Confirm | New stock, or drawn from the 0.2 and salvage pool? Needed before Day 6–9 assembly |
| **Yaw joint hardware confirmed** | Confirm | Bearing count per yaw joint and the gearmotor-to-pivot coupling detail (drives Steps 1–2) |
| **Joint-disk stock + Teflon tape** | Confirm | Disks cut Day 5, Teflon-taped Day 6; confirm disk material and tape are on hand |
| **Kid tool clearances** | Confirm | Which tools are the kids cleared to use solo — drill, jigsaw, bandsaw? |
| **i3Detroit CNC time reserved** | Confirm | Day 5 (Sunday) weekend slot on the AVID, plus a backup — Dad attends both the Day 4 layout and the Day 5 cut |
| **0.3.1 electronics plan** | Confirm | Does 0.3.1 get its own electronics, or does the 0.2 brain transplant over later? |
| **0.3.1 CAD fixes complete** | In progress (Dad, Days 1–4) | Design done before the Day 4 layout; nest + toolpaths exported before the Day 5 cut |
| **BOM / parts count** | Required | Hardware counted against the per-leg parts list before the Day 4 layout (Epic 4 prep) |

---

## Repos and Artifacts

| Artifact | Preferred location |
|----------|--------------------|
| 0.3.1 leg + chassis CAD (DXF/SVG for CNC, STEP/F3D for assembly) | `krabby-research/hardware/cad/v0.3/` |
| CNC toolpath files / G-code (AVID) | `krabby-research/hardware/cnc/v0.3/` |
| Knee logo art | `krabby-research/hardware/cad/v0.3/logos/` |
| BOM / parts list | `krabby-research/hardware/diagrams/BOM.md` |
| Assembly photos, walk-test video, standing-check photos | `krabby-research/hardware/docs/v0.3/` |
| Sprint retro + 0.3.1 punch list | `krabby-research/hardware/docs/v0.3/RETRO.md` |
| Milestone contract (ICA) | [krabby-contracts/milestones/M19/M19.md](https://github.com/flliver/krabby-contracts/blob/main/milestones/M19/M19.md) |
| Grant overview (this milestone) | [Milestone19-Krabby03-build on GitHub](https://github.com/flliver/patina-foundation-grants/tree/main/grants/Krabby-Uno/Milestone19-Krabby03-build) |

---

## Acceptance (high-level)

M19 is complete when:

1. **Krab 0.1 stripped (Epic 1):** Disassembly finished and all salvaged hardware sorted into labeled bins.
2. **Krab 0.2 walking (Epic 2):** All 6 hip linear actuators swapped, all 6 hip yaw motors mounted, wired, and passing a channel test, the ZED2i and DFRobot side cameras live on the Orin, and the robot completes a controller walk test with an acceptable gait.
3. **Krabby 0.3.1 designed (Epic 3):** 0.3.1 CAD carries the hip-chassis fix, relocated knee bolt and actuator mounts, femur/hip motor-mount holes, and knee logos, with final DXF and AVID toolpaths exported.
4. **Krabby 0.3.1 cut (Epic 4):** The full 0.3.1 plywood kit is CNC-cut at i3Detroit, deburred, and labeled.
5. **Krabby 0.3.1 standing (Epic 4):** Six leg modules assembled per the build guide, joined to the center chassis, and the robot passes the Step 9 standing check — all six feet touch flat and every joint sweeps its full range by hand, with Dad's sign-off.
6. **Documented:** CAD, toolpaths, photos, walk-test video, and the sprint retro / 0.3.1 punch list are committed to `krabby-research`.

---

## Ground Rules

- Safety glasses for anything spinning or cutting. Power tools only with clearance from Dad.
- Dry-fit every assembly before final fastening.
- Label everything. Tug-test every crimp.
- If stuck for more than 20 minutes, write down what you tried, ask Claude, and flag it for the 6pm block.
- End each solo block with a two-line log: what got done and what is blocked.
