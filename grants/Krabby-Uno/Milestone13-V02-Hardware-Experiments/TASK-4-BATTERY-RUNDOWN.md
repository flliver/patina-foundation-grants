# Task 4 — Battery Rundown Analysis (Loaded Robot)

**Milestone:** M13 – Krabby-Uno V0.2 Hardware Experiments
**Estimated effort:** ~3–5 days wall-clock (~1 day build/instrument, ~1–2 days run, ~1 day analysis)
**Dependencies:** M12 Task 4 complete — the fully assembled V0.2 robot is standing and walks under its own power on a flat surface. Batteries fully charged. A safe indoor or garage area with enough room for a back-and-forth walking path.

---

## Narrative

V0.2 carries 2× 12 V 100 Ah LiFePO₄ batteries in series (24 V nominal, 2.4 kWh usable). Nobody on the team has yet measured how long the robot can actually walk on that. Spec-sheet math (24 V × 100 Ah × 80 % DoD ≈ 1.92 kWh, divided by an *assumed* average draw of ~500 W ≈ 4 hours) is guess-on-guess.

This task runs the **fully loaded, fully assembled** robot on a known-length indoor track for as long as the batteries last, logging bus voltage, bus current, per-motor current (or per-leg if grouped), distance traveled, and average speed. The output is a **Wh/m figure** and a **predicted range** with uncertainty, plus a breakdown of where the energy goes (yaw motors vs. linear actuators vs. electronics/Orin idle).

That measurement drives a bunch of downstream decisions for V0.3: whether to go to a bigger battery bank, whether to prioritize efficiency improvements on yaw (one of the biggest continuous consumers) or on the linear actuators (intermittent but high-current), and whether the Orin's idle draw is meaningful vs. motor draw at walking speed.

---

## Subtasks

### 4.1 — Set up the walking track

**Deliverable:** A clear, indoor or garage-grade walking path long enough for meaningful data collection, with a known length marked.

Requirements:

- **Path length:** at least 10 m straight-line, preferably 20 m or more. A long hallway, a garage side-to-side run, or a driveway under cover all work. Flat surface strongly preferred — slopes introduce an unknown gravity load that changes the measurement.
- **Path marking:** tape or paint at 1 m intervals along the full length. Simplest distance logger is a camera + paint-mark counter.
- **Turnaround zones:** at each end, enough floor space for the robot to stop, execute its turn-in-place routine, and start walking back. If no in-place turn routine exists yet, just reverse direction (the slot + cam-follower yaw drive supports this natively by reversing motor direction — see [M12 build OVERVIEW](../Milestone12-V02-Build/OVERVIEW.md)).
- **Ambient conditions:** record temperature (thermal loads change battery behavior), note if indoor/outdoor and the time of day. Don't run this outdoors in direct sun or in sub-freezing conditions for the first pass — keep the variables down.

### 4.2 — Instrument the robot

**Deliverable:** Data logging for the full rundown.

Required channels, all at ≥ 1 Hz:

- **Bus voltage** at the marine bus bar (directly tapped, through a voltage divider into an Arduino analog input or a dedicated voltmeter module).
- **Bus current** into the bus bar (from the battery side). Hall-effect (e.g. ACS758-100A) recommended — the continuous draw is likely 10–30 A. Log instantaneous current and, separately, accumulate it into Ah consumed.
- **Per-leg current** — 6 channels, one per H-bridge power-input line. Reuse the ACS712 approach from M13 Tasks 1 & 3.
- **Motor case temperatures** — at least on the 6 yaw motors, ideally on the 12 linear actuators too. K-type thermocouple + MAX6675 each, or thermistors + ADC if simpler.
- **Distance / position** — either:
  - A wheel-style encoder trailed behind the robot on a cable (simplest),
  - A ceiling-mounted camera tracking a fiducial on top of the robot (cleanest),
  - Or frame-count from a fixed video camera + the marked floor (cheapest; post-process manually).
- **Orin power draw** — one separate ACS712 or INA219 on the Orin's 12 V supply / barrel jack. Separating compute-load from motor-load is one of the main outputs of this experiment.

Log everything to a host computer (Raspberry Pi, laptop riding along on the robot, or a wireless telemetry link back to a nearby laptop).

### 4.3 — Run the test

**Deliverable:** Continuous telemetry from fully charged batteries to a defined low-voltage cutoff.

Pre-run:

- Charge both 12 V 100 Ah batteries to full (record resting voltage + charger stats).
- Let them rest 30 minutes so the surface charge dissipates; record resting voltage again as the true starting point.
- Power on the robot, boot Orin, confirm all 6 legs respond, confirm instrumentation is logging, set the host clock.

Run:

- Command the robot to walk continuously along the track: forward to the end, reverse (or in-place turn), back to the start, repeat.
- Target duration: run until **bus voltage under load drops to 22.0 V** (a conservative LiFePO₄ low-voltage cutoff that preserves cell life), **or** until 8 hours of continuous walking has elapsed — whichever comes first.
- Keep a human observer in the room at all times during the first run. Unsupervised runs are only OK once the first run has shown no unexpected failure modes.
- Every 15 minutes of wall-clock time, have the observer verbally note into a handheld recorder / phone: ambient temp, robot motion quality (smooth vs. stuttering), any unusual noise.

Post-run:

- Measure total distance walked (cumulative, counted from the 1 m floor marks).
- Measure final resting voltage (after 30-minute settle).
- Measure Ah consumed (integrated current) and Wh consumed (integrated power).
- Visually inspect the robot: any hot components, any loose hardware, any scuffed plywood.

### 4.4 — Analysis and write-up

**Deliverable:** `REPORT.md` committed under `krabby-research/hardware/experiments/m13/battery-rundown/`.

Required analysis:

- **Total distance / total time / average speed.** Distance in meters, time in seconds, speed in m/s and mph.
- **Wh/m figure.** Total Wh consumed divided by total distance. Include uncertainty (95 % CI or min/max bounds) based on instrument accuracy.
- **Predicted range** at full charge: (usable battery Wh) / (Wh/m). Use 2.0 kWh as an upper bound and 1.6 kWh as a conservative bound (LiFePO₄ 80 % DoD). Report both.
- **Power breakdown chart:** average power draw during walking, decomposed into:
  - Orin + electronics idle
  - Yaw motors (6 channels summed)
  - Linear actuators (12 channels summed)
  - "Other" / error term
- **Speed vs. draw:** if the robot walks at different cadences during the run (e.g. slow during start/stop, faster in straightaways), plot instantaneous Wh/m vs. instantaneous speed.
- **Thermal curves:** motor case temp vs. time for at least one representative yaw motor and one representative linear actuator. Does anything approach 70 °C?
- **Battery voltage sag:** plot bus voltage vs. Ah consumed. Identify the knee point if any.

Required write-up sections:

- **Headline number:** predicted range (X ± Y meters on a full charge).
- **Where the Wh go:** percentage breakdown across the four buckets above.
- **Single biggest surprise** from the data.
- **Implications for V0.3:**
  - Is the Orin idle draw large enough to warrant a lower-power compute board?
  - Is one leg disproportionately drawing current (suggesting mechanical binding)?
  - Are the yaw motors hotter than the linear actuators (suggesting the slot/cam follower is the main duty-cycle consumer)?
  - Does the robot need more battery, or should efficiency be attacked first?

---

## Acceptance Criteria

Task 4 is accepted when:

1. The robot has walked continuously on the instrumented track from full charge to low-voltage cutoff (or 8 hours, whichever first).
2. Full telemetry is logged (bus voltage + current, per-leg current, Orin current, motor temperatures, distance).
3. `REPORT.md` is committed with the analysis, headline range figure, power-breakdown chart, and V0.3 implication section.
4. Raw data (CSV or similar) and at least one timelapse or walking video of the run are committed alongside the report.
