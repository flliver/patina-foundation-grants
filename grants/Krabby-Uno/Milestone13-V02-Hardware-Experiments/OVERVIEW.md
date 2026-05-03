# Milestone 13 – Krabby-Uno V0.2 Hardware Experiments (Overview)

## Milestone Overview

M13 is a set of four **hardware experiments** on top of a completed, standing Krabby-Uno V0.2 robot from M12 (V0.2 build). Each experiment pressure-tests a specific V0.2 design choice with real measurements so that V0.3 can be planned from evidence rather than from supplier data sheets.

The four experiments are **independent** and can run in any order (though Task 3 overlaps with Tasks 1–2 by design, since it mostly consumes wall-clock time). They are:

1. **[Task 1 – Full BLDC yaw motor comparison](TASK-1-BLDC-YAW.md).** Build a full brushless yaw drive on **one leg** (Xingli Z5BLD120 + ZBLD.C20 driver + the existing slot + cam-follower mechanism from M12 Task 2). Instrument it for current, temperature, and encoder tracking. Compare head-to-head against the five brushed-motor legs on the same robot, over walking tests and held-load tests.
2. **[Task 2 – DIY linear actuator from the bare motors on hand](TASK-2-DIY-LINEAR-ACTUATOR.md).** Build a prototype linear actuator from a Forto and/or Kstone and/or Mindong bare motor, a light planetary gearbox, a Tr12×3 lead screw, a POM nut, a thrust bearing, and a pair of retaining nuts or shaft collars. Measure force vs. speed vs. current and compare directly against the $36 Alibaba actuator shipped on the robot.
3. **[Task 3 – Mechanical stress test (weighted single leg)](TASK-3-STRESS-TEST.md).** Fix one leg to a heavy bench. Hang ~50 lb of weight from the foot. Command the leg to walk (cycle through its full yaw + knee + hip range) continuously for several days. Log motor temperature, current, encoder tracking, backlash, and mechanical condition at regular intervals until something fails or a target number of operating hours is reached.
4. **[Task 4 – Battery rundown analysis](TASK-4-BATTERY-RUNDOWN.md).** Run the fully loaded V0.2 robot back and forth along a known-length track for a fixed duration. Log battery voltage, bus current, per-motor current, cumulative Wh consumed, distance traveled, and average speed. Derive effective Wh/m and predicted operational range.

**Outcome:** Four committed write-ups, each with measured data, charts, and a clear recommendation for V0.3.

---

## Why is this Important?

V0.2 (after M12 build acceptance) is a working robot, but it's built on a lot of supplier-data-sheet assumptions — 2000 hr brush life, 20 N·m gearhead rating, 45 W / 1500 N / 10 mm/s Alibaba actuators, 100 Ah battery capacity. None of those numbers are real numbers until we measure them on the assembled robot doing the work it was actually designed to do.

- **M12** proves the robot can be built and stands.  
- **M13** proves that what we built *actually holds up and moves* under sustained real-world load, and gives us hard numbers to design V0.3 against.

The four experiments were explicitly deferred from M12 to keep M12's scope tight. They are the natural next milestone.

---

## Dependencies on M12

Technical scope for the prerequisite V0.2 build is in [Milestone 12 – Krabby-Uno V0.2 Build](../Milestone12-V02-Build/OVERVIEW.md).

| Dependency | Required by | Notes |
|------------|-------------|-------|
| **M12 Task 4 complete** (standing, six-leg robot) | Tasks 1, 3, 4 | Task 2 can start earlier (bench-only) |
| **M12 Task 2 complete** (slot + cam follower yaw drive proven on bench) | Task 1 | BLDC swap is into the same mechanism |
| **Spare brushed yaw motor from M12** (8 ordered, 6 installed, 2 spare) | Task 3 | One spare goes on the stress fixture |
| **Bare motors (Forto, Kstone, Mindong)** | Task 2 | Already on hand |
| **Xingli BLDC Z5BLD120 + ZBLD.C20 driver** | Task 1 | Ordered from Allen (Xingli); arrives with the main M12 motor shipment |
| **Instrumentation hardware** (ACS712 current sensors, K-type thermocouple + MAX6675 or thermistor + logger, data-logging host — can be a Raspberry Pi or laptop) | All tasks | Part of Task setup; enumerated in each task doc |

If M12 is not yet accepted, Task 2 (which is largely bench-only and doesn't touch the robot) can proceed in parallel; the other three should wait.

---

## Crew / Execution

Unlike M9, M13 is a straightforward engineering milestone (no Gas Town crew, no agent orchestration). One engineer part-time for ~one month, with AI assistance for write-ups and analysis. Tasks 3 and 4 run mostly unattended; Tasks 1 and 2 are the attended engineering time.

---

## Summary of Tasks and Links

| Task | Document | Summary |
|------|----------|---------|
| **Task 1 – BLDC yaw** | [TASK-1-BLDC-YAW.md](TASK-1-BLDC-YAW.md) | Build and benchmark a full BLDC yaw drive (Xingli Z5BLD120 + ZBLD driver) on one leg; compare life, current, heat, backlash, and driver integration against the brushed baseline. |
| **Task 2 – DIY linear actuator** | [TASK-2-DIY-LINEAR-ACTUATOR.md](TASK-2-DIY-LINEAR-ACTUATOR.md) | Build and characterize a prototype linear actuator from the Forto / Kstone / Mindong bare motors + Tr12×3 lead screw + POM nut + thrust bearing. Force/speed/efficiency curves vs. the $36 Alibaba unit. |
| **Task 3 – Mechanical stress test** | [TASK-3-STRESS-TEST.md](TASK-3-STRESS-TEST.md) | 50 lb at the foot, single leg fixed to a bench, walking continuously for days until failure or target hours. Characterize wear points, brush life, gearhead backlash, and bearing condition. |
| **Task 4 – Battery rundown** | [TASK-4-BATTERY-RUNDOWN.md](TASK-4-BATTERY-RUNDOWN.md) | Loaded robot walks a known track for hours. Derive Wh/m, predicted range, per-motor share of consumption, and thermal steady-state under continuous walking. |

---

## Repositories and Artifacts

| Artifact | Preferred location |
|----------|--------------------|
| Task 1 CAD / wiring / test scripts | `krabby-research/hardware/experiments/m13/bldc-yaw/` |
| Task 2 CAD / parts list / test scripts | `krabby-research/hardware/experiments/m13/diy-actuator/` |
| Task 3 stress-test fixture CAD / logs / photos | `krabby-research/hardware/experiments/m13/stress-test/` |
| Task 4 rundown logs / analysis notebook | `krabby-research/hardware/experiments/m13/battery-rundown/` |
| Write-ups (one markdown per task) | same folders (e.g. `.../m13/bldc-yaw/REPORT.md`) |
| Milestone contract (ICA) | [krabby-contracts/milestones/M13/M13.md](https://github.com/flliver/krabby-contracts/blob/main/milestones/M13/M13.md) |
| Grant overview & task specs (this milestone) | [Milestone13-V02-Hardware-Experiments on GitHub](https://github.com/flliver/patina-foundation-grants/tree/main/grants/Krabby-Uno/Milestone13-V02-Hardware-Experiments) |

---

## Acceptance Criteria (Milestone-Level)

M13 is complete when:

1. **Task 1** has a working BLDC yaw leg, a measured comparison against the brushed legs, and a written recommendation (keep brushed for V0.3, or switch to BLDC).
2. **Task 2** has at least one working DIY linear actuator, measured force/speed/current curves, and a written recommendation (keep Alibaba for V0.3, or switch to DIY — and if DIY, which motor).
3. **Task 3** has either a failed motor/gearbox with a root-cause analysis, or a target-hours "passed" report, including logged data and photos at regular intervals through the run.
4. **Task 4** has a full rundown log (voltage, current, distance, time) and a derived Wh/m + predicted range figure with uncertainty.
5. All four write-ups are committed to `krabby-research` under the `hardware/experiments/m13/` tree.
