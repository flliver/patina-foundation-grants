# Patina Foundation Grant — Krabby-Uno Milestone 20: v0.3 Per-Leg Motor Controller Board (EE Contract)

## Grant Overview
This milestone is a fixed-price contract with a third-party electrical engineer to design the v0.3 Krabby per-leg controller board — the first major hardware redesign of the robot's motor control electronics. The current architecture uses three Arduino Mega 2560 boards arranged in a leader/follower topology, each running the same firmware and driving six actuators via BTS7960 half-bridge modules on a custom shield; the three boards are wired together over two hardware serial lines and communicate with the Jetson Orin over USB. The v0.3 design eliminates this entire stack and replaces it with six identical STM32G474-based leg boards, one per leg, each driving its three brushed actuators via three onboard STSPIN9P21 full bridges. All six boards communicate with the Jetson Orin over a single CAN-FD bus, daisy-chained leg to leg with the Orin tapping mid-bus. The contractor's scope is schematic capture, PCB layout, and release of manufacturing files to JLCPCB; production firmware, CAN application protocol, and physical integration are out of scope for this contract. The fixed price is $3,000 USD, paid across four payment milestones, with a target of 20 business days to Gerbers. The full RFQ and selected contractor response are in this folder.

## Why is this Important?
- **Eliminates the 3-board Arduino topology.** The leader/follower Mega wiring is the biggest operational constraint on the robot today — a board restart triggers a 3-second role re-election, and wiring complexity grows with every serial-forwarding fix. Six independent identical boards with fixed node IDs are simpler to reason about, replace, and debug.
- **CAN-FD replaces all inter-board wiring.** The ribbon cable, Serial1/Serial3 leader-follower wiring, and all forwarding firmware disappear. The Orin talks directly to each leg board over one bus with standard framing and node IDs — no leader, no relay.
- **STSPIN9P21 full bridges replace BTS7960 modules.** Smaller footprint, better thermal path, built-in current sense and fault diagnosis, and lower BOM.
- **72 V headroom for the 48 V pack.** The existing boards are 24 V rated. This design targets 72 V across all components, covering the 48 V first integration with margin and future-proofing against a voltage tier increase without a board respin.
- **CAN bootloader provisions for OTA.** Hardware provisions (BOOT0, SWD recovery, three-bit node ID) are laid in at schematic time so all six boards can be updated over CAN simultaneously, without physical SWD access on each leg.
- **One design, six boards.** A single schematic and Gerber set produces all six leg boards plus spares — one BOM, one fab order, no variant management.

## Tasks
This milestone is a fixed-price EE contract, not a development sprint. The four items below are the payment milestones from the RFQ; full scope of work, design questions, and acceptance criteria are in the RFQ document. Summaries:

### Payment Milestone 1 — Requirements Freeze and Project Kickoff ($600) → [MCU-v0.3-RFQ.md](MCU-v0.3-RFQ.md)
Confirm any open questions raised during proposal review. Deliverable is a confirmed requirements document (the executed RFQ). Payment released at kickoff.

### Payment Milestone 2 — Schematic Complete and Reviewed ($300) → [MCU-v0.3-RFQ.md](MCU-v0.3-RFQ.md)
Complete schematic and preliminary BOM, reviewed against the RFQ design questions. Covers STM32G474 power tree, clock, reset, SWD/BOOT0, three-bit node ID; CAN-FD transceiver (MCP2562FD) with dual parallel JST-GH CAN ports, ESD/TVS, and selectable 120 Ω termination; three STSPIN9P21 full-bridge channels; current sense, potentiometer, and Hall encoder conditioning per motor; XT30 per motor and XT60 incoming power; fusing; 12–72 V buck for 5 V logic; and all connector pinouts. All 15 RFQ design questions answered in the schematic.

### Payment Milestone 3 — PCB Design and Gerbers Released to Fab ($1,300) → [MCU-v0.3-RFQ.md](MCU-v0.3-RFQ.md)
Four-layer PCB layout with short switching loops, thermal copper and vias under the STSPIN9P21 packages, separated analog and CAN routing, creepage and clearance to 72 V, silkscreen labels, and test points. ERC and DRC clean. DFM reviewed for JLCPCB. Manufacturing package released: BOM, Gerbers, drill files, pick-and-place, and assembly drawings in a JLCPCB-ready archive.

### Payment Milestone 4 — Final Bug Fixes and Closeout ($800) → [MCU-v0.3-RFQ.md](MCU-v0.3-RFQ.md)
One review-driven revision and one agreed first-article hardware ECO incorporated. Final BOM, final Gerbers, and a hardware test checklist (power-on sequence, SWD programming check, CAN loopback, per-channel motor drive verify) delivered. Likely requires one board respin for fixes. Project closed when the board is functioning and in the robot. 

## Information
- **Repositories:** `krabby-research` (v0.2 baseline, future firmware and integration), `patina-foundation-grants` (this grant), `krabby-contracts` (milestone ICA).
- **RFQ document:** [`MCU-v0.3-RFQ.md`](MCU-v0.3-RFQ.md) — full scope of work, design questions, proposal requirements, and payment structure.
- **Contractor response:** [`krab_v03_leg_board_RFQ_Response_v1_1.md`](krab_v03_leg_board_RFQ_Response_v1_1.md) — contractor's proposal, answers to all 15 design questions, and proposed schedule.
- **Client design notes:** [`DESIGN-NOTES.md`](DESIGN-NOTES.md) — 21-item watch list of hardware decisions, lessons from the v0.2 boards, and open questions to resolve during schematic and layout reviews.
- **v0.2 baseline design files:** `hardware/Uno-v0.2/power-board` (power board being redesigned), `hardware/Uno-v0.2/shield` (shield being eliminated), `hardware/Uno-v0.2/motors`.
- **Key components:** STM32G474RET6 (64-pin MCU, FDCAN, 18 timers); STSPIN9P21 (full-bridge motor driver, ×3 per board); MCP2562FD (CAN-FD transceiver); XT30 (per-motor power), XT60 (incoming power); 4-pin JST-GH (CAN daisy-chain).
- **Bus leader:** Seeed reComputer Robotics J4012 (Jetson Orin NX), tapping CAN bus mid-chain.
- **Dependency:** No dependencies, this is designed to be a low-effort, independent investigation to get us ahead on our options for more powerful boards/higher voltage/options for BLDC motors. F4/F5 (Integrated STM Motor Controller, TI Full-Bridge) are the longer-term successors if this design warrants further iteration.

## FAQ 
- **Why STM32G474 over ATmega?** The G474 has a hardware FDCAN peripheral, enough PWM timer channels for six actuators, ADC for current sense and position, and runs at 3.3 V. The ATmega has no CAN peripheral, can't be updated over CAN, and tops out at 24 V system operation. The G474 is the right class of MCU for a CAN-connected motor controller at this power level.
- **Why CAN-FD over the existing serial leader/follower topology?** One bus, standard framing, hardware-level error detection, 64-byte payloads, and fixed node IDs replace all serial forwarding firmware and inter-board ribbon wiring. The Orin can address each leg board directly without routing through a leader. Any single board can be restarted or replaced without affecting the other five. The big reason though is it gets rid of the intermediate shield boards entirely, and opens up a whole class of options once we have a high quality micro controller on the MCUs. 
- **Why STSPIN9P21 over BTS7960 half-bridge modules?** The STSPIN9P21 is a single-package full bridge (vs. two BTS7960 half-bridge modules per motor), soldered on-board, with built-in current sense output and fault diagnosis pins. Fewer connectors, smaller footprint, better thermal path, and no carrier PCB for the half bridges.
- **Why one board per leg instead of a centralized controller?** A failed board takes down one leg, not a third of the robot. Each board has a fixed node ID (three-bit solder jumper), eliminating role election entirely. Six identical boards mean one spare covers any position.
- **Why 72 V headroom when the pack is 48 V?** Component ratings must exceed the worst-case transient, not just the nominal rail. 72 V-rated parts give margin on a 48 V pack and avoid a full board respin if the pack voltage changes. We can also choose to boost to 60V if we want at some point.
- **What is out of scope for this contract?** Production firmware, CAN application protocol definition, motor-control algorithms, CAN bootloader code, Jetson/ROS software, harness and enclosure design, certification, fabrication, assembly, and shipping. These are follow-on work in `krabby-research`.
- **Where are the full task details?** In this folder: [`MCU-v0.3-RFQ.md`](MCU-v0.3-RFQ.md) (scope, design questions, payment structure) and [`krab_v03_leg_board_RFQ_Response_v1_1.md`](krab_v03_leg_board_RFQ_Response_v1_1.md) (contractor proposal and answers to all design questions).
