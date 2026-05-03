# Task 1 — Full BLDC Yaw Drive Experiment

**Milestone:** M13 – Krabby-Uno V0.2 Hardware Experiments
**Estimated effort:** ~1 week (part-time)
**Dependencies:** M12 Task 2 and Task 4 complete (standing robot with brushed-motor yaw drive working). Xingli BLDC motor + matched driver on hand.

---

## Narrative

M12 (V0.2 build) intentionally ships V0.2 with **brushed** yaw motors (Xingli Z5D120-24 + 1:100 gearhead, ~2000 hr rated brush life) driving the slot + cam-follower mechanism. That was the right choice for V0.2 because it drops into the existing BTS7960 H-bridge + Arduino Mega architecture with zero electrical or control-software change.

But over years of fleet operation, brushed motors on the yaw axis are the highest-wear electro-mechanical element in the robot. The Xingli **brushless** motor (Z5BLD120: 90 mm, 458 mN·m base torque, 2500 RPM, built-in hall sensors, nominal ~10,000 hr bearing-limited life) would nominally be a 4–5× lifetime improvement. The cost is a full-3-phase driver (Xingli's ZBLD.C20-120L2R or equivalent) and a different wiring / control scheme.

This task swaps the brushed yaw motor on **one leg** of the assembled V0.2 robot for a full BLDC drive — motor + driver + control-signal integration — and runs a head-to-head comparison with the five brushed legs. The output is enough evidence to decide whether V0.3 goes brushless on yaw.

---

## Subtasks

### 1.1 — BLDC driver integration

**Deliverable:** A working Xingli Z5BLD120 + ZBLD.C20 driver mounted to one leg-attachment point, wired into the existing Arduino Mega control architecture, and controllable by a simple test sketch.

Requirements:

- Swap one of the brushed yaw motors on the robot (pick a corner leg to minimize gait-side-effects during A/B testing) for the BLDC + driver assembly. Keep the existing slot + cam-follower hardware — the mechanism doesn't change, just the motor.
- Wire the driver to 24 V main bus (not through the BTS7960 — the BLDC driver has its own MOSFET power stage).
- Pick one of the two control paths (document the choice):
  - **Path A — PWM/analog:** Arduino PWM → driver VAR/AI2 speed input, digital pin → FWD/DI1 direction input, digital pin → BRK/DI5 brake. Closest analog to the existing H-bridge control. Set SW2 ON for AI2 speed control per the driver manual.
  - **Path B — RS-485 Modbus RTU:** Arduino Serial1 → MAX485 transceiver → driver RS-485 terminal. Set SW3 ON, use Modbus RTU (19200 8N1, default slave address 01) via ModbusMaster Arduino library. This path is more invasive but the cleaner long-term story.
- Route driver hall-sensor feedback: the driver handles commutation internally, but tap the hall lines to the Arduino as well so the Mega can compute position independently (same as the brushed encoder lines).
- Install an **ACS712-30A** (or equivalent) current sensor inline on the driver's 24 V supply to get a proportional current reading back to the Arduino (the ZBLD driver has internal overcurrent protection but does not expose a proportional current output).
- Confirm the driver's locked-rotor / stall protection triggers correctly on a bench test (manually stall the motor; verify the driver cuts current and sets its fault code).

### 1.2 — Control parity on the robot

**Deliverable:** The BLDC leg's yaw axis performs functionally identically to the other five legs from the main controller's perspective.

Requirements:

- Update the leg's control code so that the same trajectory command (e.g. "sweep to +30°" or "run continuous at 30 RPM motor output") produces the same observable leg motion on the BLDC leg as on the other five.
- If Path A (PWM/analog): confirm PWM duty cycle → motor RPM mapping by running the motor at 25 %, 50 %, 75 %, 100 % duty and measuring encoder-based output RPM. Produce a lookup table.
- If Path B (Modbus RTU): confirm speed commands and status reads work at the intended polling rate (~30 Hz or faster).
- Run the robot's standing test from M12 Task 4.5 with all six legs including the BLDC leg. Confirm the robot stands and holds position for ≥ 60 seconds just like in M12.

### 1.3 — Head-to-head comparison

**Deliverable:** A written report (`REPORT.md`) with measured comparisons between the BLDC leg and the five brushed legs across the test matrix below.

Test matrix (all tests run **on the standing / walking robot**, not on a bench):

| Test | Metric | Brushed legs | BLDC leg |
|------|--------|--------------|----------|
| **Quiescent hold** (robot standing, all legs holding position 5 minutes) | Bus current per leg | Measure | Measure |
| **Single stride** (one full ±30° yaw sweep at rated speed) | Motor case temperature rise, peak current, time-to-complete | Measure | Measure |
| **1-minute continuous walk** (tripod gait) | Motor case steady-state temperature, average current, encoder tracking error | Measure | Measure |
| **Stall test** (leg pressed against a fixed stop; drive commanded forward 10 s) | Peak current, time-to-protection-trip, driver behavior | Measure | Measure |
| **Efficiency** (commanded 30 RPM output, known-mass leg, 5 oscillations) | Electrical Wh in, estimated mechanical Wh out, derived efficiency | Measure | Measure |

Requirements:

- Instrument **all six legs** with ACS712 (or equivalent) inline current sensors and K-type thermocouples on the motor cases. Log at ≥ 10 Hz to an SD card or host laptop.
- Run each test at least 3 times for repeatability. Report mean ± range.
- Include at least one **thermal image** (phone thermal adapter is fine) of the robot underside after the 1-minute walk test, showing relative heating of the brushed and BLDC motors side by side.
- Include **one full-length video** of the robot walking with all six legs (5 brushed + 1 BLDC) so subjective smoothness / noise can be reviewed.

### 1.4 — Recommendation and V0.3 path

**Deliverable:** Section in the `REPORT.md` with a clear recommendation.

Required content:

- **Decision:** "V0.3 yaw should be brushed / BLDC" — with the single biggest driver of the decision called out (lifetime, cost, efficiency, noise, or control complexity).
- **Cost delta per robot** if going BLDC: 6× (motor + driver) vs. current (motor only). Include shipping.
- **Control-software implications:** What changes in the Mega sketch if BLDC is adopted fleet-wide.
- **Open questions / follow-ups:** Anything the test couldn't settle (e.g. true long-term brush-life requires M13 Task 3).

---

## Acceptance Criteria

Task 1 is accepted when:

1. One leg on the assembled V0.2 robot runs on a full BLDC yaw drive (motor + driver + integrated control).
2. The robot passes the M12 Task 4.5 standing test with the BLDC leg in place.
3. `REPORT.md` is committed under `krabby-research/hardware/experiments/m13/bldc-yaw/` with the full test matrix measured and tabulated.
4. Thermal image and walking video are included as attachments in the same folder.
5. Recommendation section is present and clearly written.
