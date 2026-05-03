# Task 4 — Full V0.2 Assembly and Test

**Milestone:** M12 – Krabby-Uno V0.2 Build
**Estimated effort:** ~1 week (part-time; begins after Tasks 1–3 deliver tested components)
**Dependencies:** Task 1 (leg structure), Task 2 (yaw motor), Task 3 (body + enclosures) must all be substantially complete.

---

## Narrative

Tasks 1–3 produce individually tested sub-systems: CNC-cut leg segments with linear-actuator joints, a working yaw-motor hinge, and a CNC plywood body with all mounts and enclosures. Task 4 brings everything together into a single, complete Krabby-Uno V0.2 robot that stands and responds to commands.

This is where the single-leg bench test becomes a six-leg integration test, all wiring is finalized, and the robot proves it can bear its own weight. This is also the proof that the design can be assembled in **< 4 hours** from pre-cut parts (see OVERVIEW.md for the target time breakdown).

---

## Subtasks

### 4.1 — Leg production run (×6)

**Deliverable:** Six identical, interchangeable legs — CNC-cut, assembled, and wired.

Requirements:

- Apply any corrections discovered during the Task 1 dry-fit and Task 2 yaw bench test to the CAD and CNC programs before the production run.
- CNC-cut all segments for six legs in a single production batch (or two batches if CNC bed size requires).
- Press-fit and/or epoxy all 6201 bearings.
- Assemble all six legs with shoulder bolts, spacers, Belleville washers, lock nuts, and linear actuators.
- Install the slotted yaw arm on each hip segment (with the slot insert / cam-follower hardware from Task 2).
- Route wiring through channels on each leg and terminate to H-Bridge boards.
- **Interchangeability check:** Pick any two legs, swap their positions on a test fixture, and confirm both fit without modification.

### 4.2 — Body assembly and electronics installation

**Deliverable:** Fully assembled body with all electronics mounted, wired, and mechanically secured.

Requirements:

- Assemble the body frame from Task 3 (bottom panel + side panels + corner braces + lateral braces + battery dividers + lid).
- Mount the 6× yaw motors + crank shafts (with cam-follower pins) to the body underside / leg-attachment points (per Task 2 motor bracket design).
- Install all 3D-printed enclosures into the body:
  - 3× Mega + shield enclosures
  - 6× H-bridge enclosures
  - 1× Orin
- Install the marine bus bar to the bottom panel.
- Route all power wiring:
  - 2 AWG from Battery 1 → Battery 2 (series) → bus bar
  - 10 AWG from bus bar to each of the 6 H-bridge boards (Mega-Fit connectors)
  - Breaker on the + rail within 7″ of the second battery's + terminal
- Route all signal wiring:
  - DB-25 cables from each Mega shield to its two H-bridge boards
  - 3-pin JST serial from each Mega to mega-1
  - USB to orin + 24v power to orin
 - Install batteries in their bays.

### 4.3 — Power-on and smoke test

**Deliverable:** Body powers up with no faults; all controllers boot and respond.

Requirements:

- **Before connecting batteries:** Continuity-check all power wiring with a multimeter. Verify no shorts between + and − rails.
- Connect batteries. Measure bus-bar voltage: expect ~25–26 V (fully charged LiFePO₄).
- Verify the Orin boots to Jetson Linux desktop and can connect to MEGAs using KrabbyMCU (see guides in krabby-research/firmware)
- Monitor for 10 minutes: no hot spots, no smoke, breaker holds, bus-bar voltage stable.
- Test all leg motors using controller and/or test GUI with Orin.
- Measure quiescent current draw (all controllers on, no motors moving). Log the value.

### 4.4 — Documentation and archive

**Deliverable:** Complete build documentation committed to the repo.

Requirements:

- **Assembly photos:** All sides of the completed robot (lid open, lid closed, underside showing yaw motors, each leg close-up).
- **Known issues list:** Any problems discovered during assembly (e.g. tight clearances, wire routing difficulties, bearing fit issues) with recommended fixes for V0.3.
- All committed to the repo (preferred: `krabby-research/hardware/docs/v0.2/`).

---

## Acceptance Criteria

Task 4 is accepted when:

1. Six legs are assembled, QC-inspected, and confirmed interchangeable.
2. Body is fully assembled with all electronics mounted and wired.
3. Power-on smoke test passes (4.3).
6. All documentation (photos, known-issues list) is committed to the repo.
