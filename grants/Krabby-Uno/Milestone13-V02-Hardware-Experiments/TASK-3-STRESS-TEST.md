# Task 3 — Mechanical Stress Test (Weighted Single Leg)

**Milestone:** M13 – Krabby-Uno V0.2 Hardware Experiments
**Estimated effort:** ~2 weeks wall-clock (~1 day build, ~10+ days unattended runtime, ~1 day analysis)
**Dependencies:** M12 Task 4 complete (at least one fully assembled leg in hand). One spare Xingli yaw motor, spare linear actuators, and a spare hip segment (enough to build a complete single-leg fixture without cannibalizing the robot).

---

## Narrative

Every number on a motor data sheet — 2000 hr brush life, 20 N·m gearhead rating, 1500 N actuator force — is derived under conditions that probably don't match krab's actual duty cycle. The only way to know what V0.2 hardware actually survives is to run it under realistic load, continuously, until something breaks or a target number of hours is reached.

This task builds a **single-leg stress fixture**: one complete leg (hip + femur + tibia + slot-drive yaw motor + 2× linear actuators) bolted rigidly to a heavy bench, with a ~50 lb weight hung from the tip of the tibia, and a simple walking program that cycles the leg through its full range of motion continuously. Instrumentation logs motor temperature, current, and encoder tracking over the whole run. Every 24 hours, a human (or a camera) checks mechanical condition: backlash, bearing noise, slot/follower wear, visible smoke, etc.

The goal isn't to "pass" — failure is a perfectly acceptable outcome. The goal is to **measure the failure mode and the hours-to-failure** so V0.3 can design around it.

---

## Subtasks

### 3.1 — Build the stress fixture

**Deliverable:** One complete leg rigidly mounted to a heavy bench or wall, with 50 lb hanging from the foot.

Requirements:

- Pick a location that can tolerate continuous motor noise for 1–2 weeks. A shop corner or a garage is fine; a living space probably isn't.
- **Fixture mount:** Bolt a spare hip body-side mounting plate (identical to the one on the robot, pre-drilled for the yaw motor, the lateral hinge pin, and the slot/cam mechanism) to a **heavy bench or a concrete-wall bracket**. The mount has to take not just 50 lb static but ~2–3× that as dynamic load as the leg cycles. Over-build it — a sheared fixture mid-test is lost time.
- **Leg:** Assemble one complete leg exactly per the M12 Task 1 spec (CNC plywood segments, shoulder-bolt + 6201 joints, 2× linear actuators, wiring harness).
- **Yaw drive:** Install the spare Xingli Z5D120-24 + 1:100 gearhead + 1:100 crank disc + cam follower + slotted hip arm, exactly per the M12 Task 2 spec.
- **Weight:** Hang 50 lb (22.7 kg) from the tibia tip. A kettle-bell through a short loop of chain is the simplest option. Pick a weight-attachment point that can't catch on anything as the leg swings.
- **Swing envelope:** The fixture has to be in clear space — the foot+weight traces a large arc through 3D space as the leg cycles. Measure the full swing envelope on paper before bolting anything down.

### 3.2 — Instrument the rig

**Deliverable:** A logging setup that captures motor current, motor temperature, and encoder position for all three joints (yaw, hip, knee) continuously, plus a camera for visual monitoring.

Requirements:

- **Current sensors:** ACS712 (or equivalent) on each motor's supply line — 1× yaw, 2× linear actuator. Log at ≥ 10 Hz.
- **Temperature sensors:** K-type thermocouple (MAX6675 → Arduino) on each motor case; 1× ambient temperature in the fixture area. Log at 1 Hz.
- **Encoder:** Already on each motor; already connected to the existing Arduino Mega. Log cumulative encoder counts + commanded position for each joint at ≥ 10 Hz.
- **Data logger host:** a Raspberry Pi or spare laptop with USB to the Arduino(s). Log to CSV files, rotated daily.
- **Camera:** a USB webcam (or phone) pointed at the fixture, recording low-frame-rate (e.g. 1 fps) timelapse to disk. This is invaluable for post-mortem on "something happened overnight" failures.
- **Wall-clock annotation:** every log row has a UTC timestamp so cumulative hours can be computed.
- **Safety shutoffs:**
  - Software: if motor current exceeds 1.5× rated for > 500 ms, or motor case temp exceeds 85 °C, cut power to the motor and log the event.
  - Hardware: a 150 A breaker (same as on the robot) or appropriate in-line fuse.
  - Fire-adjacent: no flammables within arm's reach of the fixture.

### 3.3 — Write the test program

**Deliverable:** An Arduino + Pi program that cycles the leg through realistic walking motion indefinitely.

Required behavior per cycle (~1 cycle ≈ 1 stride ≈ 2 s at 30 RPM yaw output):

1. Yaw: motor rotates one full revolution (= one ±30° oscillation of the hip arm).
2. Knee + hip linear actuators follow a "lift-swing-plant-push" trajectory that roughly mimics stance / swing phases, with extension and retraction spanning **at least 50 % of stroke** and with at least one extension-under-load phase per cycle. Actuators should see repeated direction changes within each stride, matching the robot's real duty cycle for them.
3. Log one row per joint at 10 Hz.
4. Between cycles: no pause. Run continuously until software cutoff triggers, a human halts, or a mechanical fault occurs.

Additionally:

- Every 3600 cycles (~2 hours), write a **condition-check banner** into the log so analysis is easy to align.
- At software startup, record and log BOM info (motor serials, encoder types, firmware hash).

### 3.4 — Run + daily inspection

**Deliverable:** Continuous log data and daily inspection notes, through at least one of: **failure, 240 operating hours, or 500,000 yaw cycles** (whichever comes first).

Daily (every 24 hours or one nominal run-day):

- Walk to the rig. Listen for unusual noise (grinding, clunking, whining).
- Visually inspect: any oil/grease migration, any cracks in the plywood, any loose hardware, any smoking insulation. Photograph anything changed.
- Grab backlash numbers: with the motor off, push each joint through its play by hand and record the angular slop. Compare to day-zero numbers.
- Check slot and cam-follower: visible wear marks, follower-pin shininess (indicating metal-to-metal scrub), any play that wasn't there yesterday.
- Thermal: touch-check the motor case (or thermal-image it). Record ambient temp.
- Append a dated markdown entry to `inspection-log.md`.

End-of-run:

- If **failure occurred:** photograph the failure mode, disassemble the failed component(s), and write a root-cause section in the report. Save the broken parts (don't throw them out — they are future design data).
- If **target hours reached without failure:** photograph current condition in the same framing as day-zero photos, disassemble and inspect each joint, record backlash / wear / bearing-smoothness for each. The target hours are **not** a final verdict — M13 or later can resume the fixture with the same parts if we want more data.

### 3.5 — Write-up

**Deliverable:** `REPORT.md` committed under `krabby-research/hardware/experiments/m13/stress-test/`.

Required content:

- Fixture description with photos.
- Full CSV data attached (or linked via LFS / external storage if large).
- Cumulative hours run, cumulative cycles run, ambient temperature range.
- Charts: motor temperature vs. time, motor current vs. time, encoder drift vs. time (cumulative tracking error across the run).
- Inspection log (chronological markdown entries).
- **Failure analysis or endurance report** depending on outcome.
- **Design implications for V0.3:** concrete list of the ≥ 3 changes that would most improve life (e.g. "upgrade cam follower to sealed needle bearing", "use steel hip pin rather than shoulder bolt", "add grease fitting to slot insert").

---

## Acceptance Criteria

Task 3 is accepted when:

1. The fixture is built and documented, with photos.
2. Instrumentation logs continuously for the full run.
3. The run is terminated either by a logged failure, by reaching 240 hours / 500,000 cycles, or by a mutually-agreed early stop (documented).
4. Daily inspection log has entries for every run-day.
5. `REPORT.md` is committed with data, charts, failure or endurance analysis, and a concrete V0.3 recommendations section.
