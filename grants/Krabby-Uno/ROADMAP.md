# Krabby-Uno — Engineering Roadmap

_Last updated: 2026-05-30_

Delivery plan and milestone status across Krabby-Uno: locomotion/AI, EE & hardware, and on-robot + fleet software. The goal is a robot anyone can reproduce by checking out the repo and buying their own parts. Per-milestone scope is authoritative in each `MilestoneN-*/OVERVIEW.md` (this folder) and the matching contract in `krabby-contracts/milestones/MN/`.

Numbering convention: **M-numbered** milestones have a folder + contract and are scheduled. **F-numbered** items are roadmap backlog — concept-level, not yet sized for a specific window. An F-item gets promoted to an M-number when it's folder-ized.

---

## 1. Delivery timeline

The headline deliverables, by date. `Mxx` / `Fxx` tags link each item to the milestone sections below.

### End of June 2026 — First real-world V0.2 robot, end-to-end
**A V0.2 robot walking in the real world on v2 brushed motors, driven by a real controller, connected end-to-end through the fleet manager + teleop.**
- First V0.2 build assembled and walking — `M12` build executed; v2 brushed motor selection settled (`F6`).
- Robot calibrated end-to-end (joints + current sense + IMU through HAL) — `M17` (Krab E2E Sensor Validation).
- RL locomotion policy running on hardware — `M15` (Sim2Real).
- Real handheld controller driving the robot — `M6`/`M7` control path.
- End-to-end cloud: device onboarded, telemetry flowing, teleop session from the portal — `M10` (fleet) + `M14` (krabby-launcher) + `M7` (teleop).
- **Critical path:** gated by motor delivery; `M15`, `M17`, `M10`, and the build all land in parallel — the tightest window on the roadmap.

### End of July 2026 — Tuned, safe, calibrated; fabricate 5–10 bots
**Smooth, usable locomotion; power and safety hardened; small fleet in fabrication.**
- Tuned RL policy; first-pass locomotion/usability kinks resolved — `M15` / `M2`.
- Full BMS settled — `M16`.
- E-stops in place — _(new: safety workstream)_.
- Sensor-configuration cleanup; automatic full-robot calibration — covered by `M17`.
- Fabricate 5–10 bots — `F6` (BOM + build guide at small-batch scale).

### Aug–Sept 2026 — Field testing, strafing, ops tooling
- Daily real-world field testing of the fleet.
- Strafing / omnidirectional locomotion — `F1`.
- Operational cleanup + regression tooling; per-engineer build/deploy — `F3` + _(new: regression harness)_.

### Oct–Dec 2026 — Navigation; BLDC/48 V; reliability
- Navigation tasks — _(new: autonomous nav workstream)_.
- BLDC + 48 V integration if warranted — `F4`/`F5`/`F7` drive stack (boards designed for ≤72 V; 48 V first integration).
- Reliability improvements driven by field data.

### Jan–Mar 2027 — Nav + voice, arms, model variants, kit production
- Integrated navigation + voice control — _(new)_.
- Arms / manipulators + controllers — _(new)_.
- Larger / smaller model variants — _(new)_.
- Standardize production and ship kits en masse — `M14` installable stack + `F6` productization.

---

## 2. Milestone status (M1–M17)

| # | Milestone | Track | Repo | Status | Notes |
|---|-----------|-------|------|--------|-------|
| **M1** | Base Training (Isaac Lab + RSL-RL, eval harness) | AI/Sim | krabby-research | ✅ | Foundational RL pipeline. |
| **M2** | Hexapod Training (teacher/student parkour, `crab_hex.usd`) | AI/Sim | krabby-research / holosoma | 🔄 | Policies trained; gait tuning ongoing. |
| **M3** | HAL Runtime (ZMQ HAL contract, Isaac + Jetson HAL servers) | AI/Sim | krabby-research | ✅ | Shared IPC consumed by M7/M15. |
| **M4** | MCU Firmware (Mega, MCUSDK — 18-actuator drive + telemetry) | EE/Firmware | krabby-research/firmware | ✅ | Leader/follower Mega topology. |
| **M5** | Holosoma Hexapod (training env / embodiment) | AI/Sim | holosoma | ✅ | — |
| **M6** | Joystick Krab (gamepad locomotion path) | Control | krabby-research | ✅ | Controller→policy path reused by M7/M15. |
| **M7** | Vision + Teleop (WebRTC agent, `krabby-control-v1`) | Vision/Control | krabby-research | 🔄 | Built in `teleop/`; being finished. |
| **M8** | MCU + H-Bridge PCB (krabbyuno v0.1) | EE/Hardware | krabby-research/hardware | ✅ | v0.1 schematics, gerbers, power + shield boards. |
| **M10** | Fleet Manager | Fleet/DevOps | krabby-home/fleet | ⬜ | AWS IoT Core MQTT (one channel shared w/ teleop signaling), per-device X.509, pull-based staged OTA. Spec rewritten 2026-05; build not started. **June target.** |
| **M11** | Scene Reconstruction (nuRec) | AI/Sim | krabby-research | 🔄 | SLAM3R → nuRec. |
| **M12** | V0.2 Build (frame, legs, yaw motor) | Hardware | krabby-research | 🔄 | Blocked on motor delivery; chassis hand-off feeds `M17` Task 4. |
| **M13** | V0.2 Hardware Experiments (BLDC yaw, DIY linear actuator, stress, battery) | EE/Hardware | krabby-research | ⬜ | Follow-on to M12; precursor to F7. |
| **M14** | Installable Stack (`krabby-launcher` / `-firmware` / `-bench`) | Fleet/DevOps | krabby-research | ⬜ | On unmerged `m14` branch. Owns OTA pull + `krabby enroll`/`agent` onboarding. |
| **M15** | Sim2Real (prismatic action space, ZED depth pipeline, domain randomization) | AI/Sim | krabby-research | ⬜ | Policy on real hardware. **June target.** Depends on `M17`. |
| **M16** | I2C Sensors & Battery Mgmt (BMI270, Qwiic OLED, 2× INA228, low-power FSM) | EE/Firmware | krabby-research/firmware | ⬜ | Breakouts on existing wiring; shield integration is F2. **BMS = July target.** |
| **M17** | Krab End-to-End Sensor Validation (V0.3 bring-up, joint cal, current sense, ZED IMU, model E2E) | EE/Firmware + AI/Sim | krabby-research | ⬜ | **Prerequisite for `M15`.** Folder created 2026-05-30. |

---

## 3. Future backlog (F1–F8)

Backlog feeding the timeline. F-series items don't have folders yet — they get promoted to M-numbers when scoped. Target phase in parentheses.

- **F1 — Omnidirectional / Sideways Locomotion** (Aug–Sept). Add a lateral-velocity command + curriculum to the single locomotion policy (strafe via hip/knee, suppress yaw) so it fits through doors. Feeds F8.
- **F2 — Shield Rev: integrated IMU + status LED + battery mgmt** (after M16). Fold M16's Qwiic breakouts onto the shield PCB as bare chips.
- **F3 — Personal Dev Stacks** (Aug–Sept; high leverage earlier). Username-namespaced package/image build + push so the full build→upload→install path is testable per engineer.
- **F4 — Integrated STM Motor Controller** (Oct–Dec). One STM32-class MCU, ≤72 V, 18–24 actuators. Pairs with F5.
- **F5 — Power Board, TI Full-Bridge** (Oct–Dec). Replace BTN half-bridges; 48 V first integration, 72 V headroom. Pairs with F4.
- **F6 — V2 Production Build + BOM + Install Guide + V1 Motors** (June–July). First build (June) → BOM, build guide, and 5–10-bot fab (July). Gated on motor delivery. Consumes the calibrated robot from `M17` (any final motor exchanges happen here; calibration should already be correct).
- **F7 — All-BLDC Krab + Custom Linear Actuators** (Oct–Dec+). BLDC + embedded-screw linear actuators, no support frame, higher power tier. After M13 + F4/F5.
- **F8 — Unified Directional Controller + E2E Motion Model** (post-F1). Simplified remote (wheel = strafe, arrows = up/down, buttons = rotate) → e2e model refining strafe/translate/rotate, body held level. After M15 + F1.

**New workstreams surfaced by the timeline, not yet F-numbered** — assign as scoped: e-stops/safety, sensor-config cleanup, regression tooling, autonomous navigation, voice control, arms/manipulation, model size variants.

---

## 4. Open technical questions

1. **F7 BLDC drive path** — drive off-the-shelf BLDCs from the existing krabby motor boards (V + PWM), or a custom harness off the Arduinos? Gates the F7 ↔ F4/F5 boundary.
2. **F4/F5 feasibility** — one MCU driving 18–24 actuators at 48–72 V: validate PWM/timer channel count, gate-driver fan-out, and thermal envelope before committing the topology; define the F4/F5 responsibility split.
3. **F1/F8 command surface** — define how translate/strafe/rotate commands compose in a single policy and map to the simplified remote, so F1 and F8 share one schema.
4. **F2 partitioning** — which M16 sensors move to bare-chip on-shield vs. stay modular/Qwiic for serviceability.

---

## 5. Notes

- F-series numbering is provisional and stable across a backlog item's lifetime — an F1 promoted to a milestone keeps the "was F1" reference for traceability and gets a new M-number based on schedule order.
- M-numbered milestones have a folder under `grants/Krabby-Uno/MilestoneN-*` and a matching contract in `krabby-contracts/milestones/MN/`.
