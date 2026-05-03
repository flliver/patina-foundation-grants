# Task 2 — DIY Linear Actuator Experiment

**Milestone:** M13 – Krabby-Uno V0.2 Hardware Experiments
**Estimated effort:** ~1 week (part-time, bench-only — does not block the robot)
**Dependencies:** Bare Forto / Kstone / Mindong motors on hand. A few Tr12×3 lead screws, POM nuts, and thrust bearings (51201 or similar). One spare Alibaba actuator for head-to-head testing.

---

## Narrative

V0.2 uses off-the-shelf **$36 Alibaba brushed linear actuators** (45 W, 1500 N, 10 mm/s, 100 mm stroke, worm-gear + acme screw internals). They work, they're cheap, and they self-lock — but they're open-loop and they sit at only ~16 % overall electrical-to-mechanical efficiency because the worm gear is a ~45 % efficient stage.

We have three small bare motors on hand that would each make a plausible foundation for a **DIY linear actuator** using a Tr12×3 lead screw and a POM nut — the screw itself is the primary reduction, which means the motor's own gearbox can stay light (~1:10–1:20) and the combined system can hit ~35 % overall efficiency (roughly 2× the Alibaba unit):

- **Forto bare motor** (~115 mN·m base, ~7000 RPM, $40 for motor-only-ish): cordless-drill-class power. Cheap. Light.
- **Kstone KS-57P997** bare motor (~95 mN·m base, ~4200 RPM, $55 with encoder): mid-pack.
- **Mindong SBR3077B-02** BLDC (~72 mN·m, 6600 RPM, 30 mm frame, $140 with 1000 CPT quadrature encoder): precision. Best encoder of anything in the lot. Small.

This task builds **at least one** working DIY actuator (target: Mindong + light planetary + Tr12×3 + POM nut + thrust bearing), characterizes it on a bench test rig, and compares it directly against an Alibaba unit across force, speed, current, efficiency, and noise. If time allows, build a second unit using a different motor for comparison.

Task 2 is **bench-only** — it does not require the assembled robot. It can run in parallel with Tasks 1 / 3 / 4.

---

## Mechanical Architecture (reference)

Per the motor-discussion notes:

```
Motor + small planetary gearbox (~1:10–1:20) ─┐
                                              ├─ rigid coupling
                 Lead screw (Tr12×3) ─────────┘
                      │
                      ├─ thrust bearing (51201) ─ captured with lock-nuts-on-flats or turned shoulder + retaining ring
                      │
                      ├─ POM nut (clamped to moving load / leg link pin)
                      │
                      └─ far-end pillow bearing (optional)
```

The whole actuator pivots at both ends on clevis pins (frame end: pivot pin on the motor bracket; far end: pivot pin on the POM nut / leg link). This matches the hydraulic-cylinder geometry already used by the Alibaba unit on the robot.

---

## Subtasks

### 2.1 — Build the first DIY actuator (Mindong preferred)

**Deliverable:** One fully assembled, working DIY linear actuator.

Requirements:

- Motor: Mindong SBR3077B-02 (preferred — best encoder, smoothest low-speed motion, highest efficiency base motor). Substitute Kstone KS-57P997 or Forto bare if the Mindong proves too fragile during bench-fit.
- Small planetary gearbox matched to the motor face. Target ratio ~1:10 (Mindong + 1:10 → 660 RPM at screw → ~33 mm/s; see discussion notes). Verify with Emily (Mindong) or local supplier. Ratio can be tuned in 2.3.
- Shaft-to-screw coupling: **jaw coupler** (preferred, absorbs misalignment) or rigid coupler with Dremel flat on the Tr12×3 screw for set-screw bite, per motor-discussion notes. Document the choice.
- Thrust bearing retention at the screw end: two Tr12×3 nuts on either side of a 51201 thrust bearing, locked with **red Loctite** (prototype), OR a lathe-turned smooth shoulder with snap rings (production-quality). Prototype with the Loctite approach.
- Clevis mount on the motor end: a bolt-on plywood or 3D-printed bracket with a pivot pin, identical pin diameter to the Alibaba actuator so swap-in testing is simple.
- POM nut: use the same POM nuts already on the robot; make sure the far-end pivot matches the Alibaba spec.
- Document every part with source, price, and qty in a `BOM.md` alongside the CAD.

### 2.2 — Build the bench test rig

**Deliverable:** A simple bench fixture that applies a known axial load to the actuator and measures extension, speed, and motor current simultaneously.

Requirements:

- Fixed base (scrap plywood or 2020 extrusion) holds the actuator's motor-end clevis pin.
- Moving end of the actuator pulls against a **calibrated spring or a hanging weight stack**. Options:
  - Weight stack via pulley: hang 10, 20, 50, 100, 200, 500, 1000, 1500 N in sequence.
  - S-type load cell inline: cleaner, but needs a logger.
  - Gym cable-crossover or similar with a scale.
- Linear-travel measurement: a simple **linear potentiometer or dial indicator** over the stroke, or frame-by-frame video analysis if finer instrumentation isn't available. Target ≥ 1 mm position resolution.
- Current measurement: ACS712 inline on the motor supply. Log at ≥ 100 Hz.
- Motor case thermocouple on the motor body.
- Data logger: Raspberry Pi or laptop with a USB ADC (e.g. the Arduino Mega from the M8 stack), logging to CSV.

### 2.3 — Characterize the DIY actuator

**Deliverable:** Force / speed / current / efficiency curves for the DIY actuator at two or three gearbox ratios.

Test points (run each 3× for repeatability):

| Load | Extend cmd | Retract cmd | Metrics logged |
|------|-----------|-------------|----------------|
| No load | full-speed | full-speed | Top speed, no-load current |
| 100 N | full-speed | full-speed | Speed under light load, current |
| 500 N | full-speed | full-speed | Speed, current, temperature rise |
| 1000 N | full-speed | full-speed | Speed, current, temperature rise |
| 1500 N | full-speed | full-speed | Speed, current, temperature rise, stall behavior |
| "Alibaba match" (1500 N target) | full-speed | full-speed | Head-to-head with the Alibaba point |

Requirements:

- Run the test at **at least two gearbox ratios** (e.g. 1:10 and 1:20) if spare gearheads are available. Otherwise document that only one was measured.
- Produce **one chart** (force vs. speed) overlaying the DIY actuator and the Alibaba actuator.
- Produce **one chart** (efficiency vs. load) overlaying both.
- Report stall current and stall force for each.

### 2.4 — Backdrive / self-locking check

**Deliverable:** Short section in the report quantifying backdrive behavior.

The discussion notes flag that Tr12×3 + POM is **right at the self-locking threshold** — it probably holds static loads, but shocks may cause it to slip. The Alibaba actuator (worm gear + acme screw) is definitively self-locking. This matters for the robot's stance phase.

Test:

- Extend the DIY actuator to mid-stroke, with motor unpowered.
- Hang 100 N, 500 N, 1000 N, 1500 N sequentially from the nut end.
- After each load is applied, tap the actuator body firmly with a soft mallet 5×. Measure any change in actuator position with a dial indicator.
- Repeat the Alibaba actuator under identical conditions.
- Report any slippage (or confirm zero slip up to test load).

### 2.5 — Write-up and recommendation

**Deliverable:** `REPORT.md` committed under `krabby-research/hardware/experiments/m13/diy-actuator/`.

Required content:

- BOM (parts, prices, total per-unit cost) for the DIY actuator. Compare to the $36 Alibaba unit.
- All charts and tables from 2.3 and 2.4.
- Subjective notes (noise, smoothness, heat, wiring hassle).
- **Decision section:** "V0.3 should stay with Alibaba / move to DIY" and why. If DIY, which motor and which gear ratio.
- If DIY is recommended: a design iteration list for a V2 DIY actuator (integrated housing, ball-screw vs. trapezoidal, lock-pin for self-locking, encoder integration).

---

## Acceptance Criteria

Task 2 is accepted when:

1. At least one DIY linear actuator is built, runs under its own power, and completes at least one load cycle.
2. The bench test rig is documented (photos + CAD/description) so it can be rebuilt.
3. Force/speed/current/efficiency curves are measured and committed as data + charts.
4. Backdrive test is completed for both the DIY and Alibaba units.
5. `REPORT.md` with measurements, BOM comparison, and a clear decision section is committed to the repo.
