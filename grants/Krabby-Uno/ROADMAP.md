# Krabby-Uno — Engineering Roadmap

_Last updated: 2026-05-26_

Delivery plan and milestone status across Krabby-Uno: locomotion/AI, EE & hardware, and on-robot + fleet software. The goal is a robot anyone can reproduce by checking out the repo and buying their own parts. Per-milestone scope is authoritative in each `MilestoneN-*/OVERVIEW.md` (this folder) and the matching contract in `krabby-contracts/milestones/MN/`.

---

## 1. Delivery timeline

The headline deliverables, by date. `Mxx` tags link each item to the milestone sections below.

### End of June 2026 — First real-world V0.2 robot, end-to-end
**A V0.2 robot walking in the real world on v2 brushed motors, driven by a real controller, connected end-to-end through the fleet manager + teleop.**
- First V0.2 build assembled and walking — `M12` build executed; v2 brushed motor selection settled (`M22`).
- RL locomotion policy running on hardware — `M15` (Sim2Real).
- Real handheld controller driving the robot — `M6`/`M7` control path.
- End-to-end cloud: device onboarded, telemetry flowing, teleop session from the portal — `M10` (fleet) + `M14` (krabby-launcher) + `M7` (teleop).
- **Critical path:** gated by motor delivery; `M15`, `M10`, and the build all land in parallel — the tightest window on the roadmap.

### End of July 2026 — Tuned, safe, calibrated; fabricate 5–10 bots
**Smooth, usable locomotion; power and safety hardened; small fleet in fabrication.**
- Tuned RL policy; first-pass locomotion/usability kinks resolved — `M15` / `M2`.
- Full BMS settled — `M16`.
- E-stops in place — _(new: safety workstream)_.
- Sensor-configuration cleanup; automatic full-robot calibration — _(new)_.
- Fabricate 5–10 bots — `M22` (BOM + build guide at small-batch scale).

### Aug–Sept 2026 — Field testing, strafing, ops tooling
- Daily real-world field testing of the fleet.
- Strafing / omnidirectional locomotion — `M17`.
- Operational cleanup + regression tooling; per-engineer build/deploy — `M19` + _(new: regression harness)_.

### Oct–Dec 2026 — Navigation; BLDC/48 V; reliability
- Navigation tasks — _(new: autonomous nav workstream)_.
- BLDC + 48 V integration if warranted — `M20`/`M21`/`M23` drive stack (boards designed for ≤72 V; 48 V first integration).
- Reliability improvements driven by field data.

### Jan–Mar 2027 — Nav + voice, arms, model variants, kit production
- Integrated navigation + voice control — _(new)_.
- Arms / manipulators + controllers — _(new)_.
- Larger / smaller model variants — _(new)_.
- Standardize production and ship kits en masse — `M14` installable stack + `M22` productization.

---

## 2. Milestone status (M1–M16)

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
| **M12** | V0.2 Build (frame, legs, yaw motor) | Hardware | krabby-research | 🔄 | Blocked on motor delivery; first build executed in M22. |
| **M13** | V0.2 Hardware Experiments (BLDC yaw, DIY linear actuator, stress, battery) | EE/Hardware | krabby-research | ⬜ | Follow-on to M12; precursor to M23. |
| **M14** | Installable Stack (`krabby-launcher` / `-firmware` / `-bench`) | Fleet/DevOps | krabby-research | ⬜ | On unmerged `m14` branch. Owns OTA pull + `krabby enroll`/`agent` onboarding. |
| **M15** | Sim2Real (prismatic action space, ZED depth pipeline, domain randomization) | AI/Sim | krabby-research | ⬜ | Policy on real hardware. **June target.** |
| **M16** | I2C Sensors & Battery Mgmt (BMI270, Qwiic OLED, 2× INA228, low-power FSM) | EE/Firmware | krabby-research/firmware | ⬜ | Breakouts on existing wiring; shield integration is M18. **BMS = July target.** |

---

## 3. Planned milestones (M17–M24)

Backlog feeding the timeline; target phase in parentheses. Numbering is provisional sequencing.

- **M17 — Omnidirectional / Sideways Locomotion** (Aug–Sept). Add a lateral-velocity command + curriculum to the single locomotion policy (strafe via hip/knee, suppress yaw) so it fits through doors. Feeds M24.
- **M18 — Shield Rev: integrated IMU + status LED + battery mgmt** (after M16). Fold M16's Qwiic breakouts onto the shield PCB as bare chips.
- **M19 — Personal Dev Stacks** (Aug–Sept; high leverage earlier). Username-namespaced package/image build + push so the full build→upload→install path is testable per engineer.
- **M20 — Integrated STM Motor Controller** (Oct–Dec). One STM32-class MCU, ≤72 V, 18–24 actuators. Pairs with M21.
- **M21 — Power Board, TI Full-Bridge** (Oct–Dec). Replace BTN half-bridges; 48 V first integration, 72 V headroom. Pairs with M20.
- **M22 — V2 Production Build + BOM + Install Guide + V1 Motors** (June–July). First build (June) → BOM, build guide, and 5–10-bot fab (July). Gated on motor delivery.
- **M23 — All-BLDC Krab + Custom Linear Actuators** (Oct–Dec+). BLDC + embedded-screw linear actuators, no support frame, higher power tier. After M13 + M20/M21.
- **M24 — Unified Directional Controller + E2E Motion Model** (post-M17). Simplified remote (wheel = strafe, arrows = up/down, buttons = rotate) → e2e model refining strafe/translate/rotate, body held level. After M15 + M17.

**New workstreams surfaced by the timeline, not yet milestone-numbered** — assign numbers as scoped: e-stops/safety, automatic robot calibration, sensor-config cleanup, regression tooling, autonomous navigation, voice control, arms/manipulation, model size variants.

---

## 4. Open technical questions

1. **M23 BLDC drive path** — drive off-the-shelf BLDCs from the existing krabby motor boards (V + PWM), or a custom harness off the Arduinos? Gates the M23 ↔ M20/M21 boundary.
2. **M20/M21 feasibility** — one MCU driving 18–24 actuators at 48–72 V: validate PWM/timer channel count, gate-driver fan-out, and thermal envelope before committing the topology; define the M20/M21 responsibility split.
3. **M17/M24 command surface** — define how translate/strafe/rotate commands compose in a single policy and map to the simplified remote, so M17 and M24 share one schema.
4. **M18 partitioning** — which M16 sensors move to bare-chip on-shield vs. stay modular/Qwiic for serviceability.

---

## 5. Notes

- M17–M24 numbering is provisional sequencing, not a delivery commitment.
