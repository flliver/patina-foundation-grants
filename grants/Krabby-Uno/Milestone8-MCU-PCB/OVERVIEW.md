# Patina Foundation Grant  
## Krabby-Uno Milestone 8: Modular Wiring and Combined H-Bridge Hardware

---

## Grant Overview

This milestone covers the design of two simple, tightly scoped hardware PCBs to clean up and modularize the actuator wiring for the Krabby-Uno hexapod robot.

The robot has 18 linear actuators, each with a 5-wire interface (2 motor wires, 3 rotary potentiometer wires). The current design uses one BTS7960-class H-bridge per actuator with ~10 individual wires routed directly to an Arduino Mega. This creates excessive wiring clutter, makes enclosure design impractical, and increases the risk of noise and miswiring.

The goal of this milestone is to replace that wiring with a repeatable, enclosure-friendly pattern using:

- a passive Arduino Mega shield that routes control signals to DB-19 connectors, and
- a combined H-bridge board that integrates three BTS7960-class drivers and consolidates all actuator connections.

---

## Why is this Important?

- Eliminates large bundles of loose point-to-point wires between Arduino Megas and H-bridges  
- Replaces Dupont wiring with locking, strain-relieved connectors  
- Makes it possible to design and print protective enclosures  
- Establishes a modular pattern:  
  18 actuators -> 6 identical driver boards -> 6 identical cables  
- Reduces wiring errors and improves serviceability  
- Creates a hardware foundation that does not block future migration to CAN or other buses  

---

## Tasks

### Task 1 - Arduino Mega DB-19 Control Shield

#### Narrative

Design a passive Arduino Mega shield that reroutes existing Mega pins to DB-19 connectors used only for control and feedback signals. Each DB-19 connector serves one group of three actuators. The shield performs no signal processing and contains no active electronics. It exists solely to replace fragile wiring with a clean, repeatable connector interface.

The DB-19 connector is explicitly **not** used for motor power. Motor power will be handled entirely on the H-bridge board via a separate connector.

The shield must be mechanically suitable for enclosure mounting and compatible with a 3D-printed case with a screwless, clip-on cover.

#### Acceptance Criteria

The shield must satisfy the following:

- PCB uses standard Arduino Mega shield footprint
- Two DB-19 connectors per shield, each serving three actuators
- DB-19 is used exclusively for control and feedback signals
- Per DB-19 connector, the following signals are routed:
  - **6x PWM lines** (R/L PWM, dual-PWM style)
  - **3x EN** (one per motor, not directional)
  - **3x IS** (one per motor, not directional)
  - **3x rotary potentiometer signal lines**
  - **1x logic VCC**
  - **1x logic GND**
  - **2x spare pins**
- No motor power routed through the shield
- Clear silkscreen labeling for signals and connector orientation
- Complete schematic, PCB layout, Gerbers, and pin-mapping documentation delivered
- Matching 3D-printable enclosure provided with all connectors exposed and a screwless clip-on cover

---

### Task 2 - Combined 3-Actuator H-Bridge PCB

#### Narrative

Design a combined H-bridge PCB that integrates three BTS7960-class motor drivers into a single board.

Each board drives exactly three actuators and interfaces to the Arduino Mega shield through a single DB-19 control connector. Motor power enters the board through a dedicated 2-pin locking Molex connector rated for worst-case current at 12V operation.

Each actuator connects to the board via a **1x5-pin connector** carrying motor and rotary potentiometer signals. The board internally splits these signals so that motor wires route to the H-bridges and potentiometer signals route to the DB-19 control connector.

Thermal management, power conditioning, and noise isolation are part of the PCB design and must be compatible with a 3D-printed enclosure and an inexpensive, standard aluminum fin heatsink.

#### Acceptance Criteria

The H-bridge PCB must satisfy the following:

- PCB integrates exactly **3 BTS7960-class H-bridges**
- Board has the following external connectors:
  - **1x DB-19** (control signals only)
  - **3x 1x5-pin actuator connectors**
  - **1x 2-pin locking Molex battery input**
- Battery connector and copper sizing support:
  - ~8A per motor at 12V
  - ~3A per motor at 24V
  - Lower current operation at 48V
- IS outputs include:
  - ~10kohm series resistor
  - RC filtering capacitor
  - Zener diode clamp for ADC over-voltage protection
- Board includes appropriate bulk and local capacitors to handle motor current spikes and smooth battery draw
- EN and IS signals are per-motor, not directional (no left/right duplication)
- Potentiometer inputs are explicitly treated as rotary potentiometers
- PCB layout separates high-current motor paths from low-level control and feedback signals to avoid noise coupling
- Thermal design supports mounting a standard off-the-shelf aluminum fin heatsink compatible with the enclosure
- Matching 3D-printable enclosure provided with all connectors external and a screwless clip-on cover
- Complete schematic, PCB layout, Gerbers, connector pinouts, and assembly notes delivered
- Design is manufacturable and intended to be printed and validated by the grant owner

---

### Task 3 - Fabricate, Assemble, and Test Arduino Mega DB-19 Control Shields

#### Narrative

This task covers fabrication, assembly, and electrical verification of the shield design from Task 1. Before committing to production, a finalized fab/assembly package (Gerbers, BOM, pick-and-place, panelization if used, assembly notes) is prepared and reviewed/approved as part of the grant work.

Interaction with a PCB fabrication/assembly vendor (for example JLCPCB, PCBWay, or a similar house) is in scope for this milestone. The goal is to demonstrate an end-to-end path from design files to working, assembled hardware using commodity PCB services; the process and vendor-specific notes should be documented so others can reproduce the build.

The shields must be delivered as fully assembled, ready‑to‑use Arduino Mega shields that can be plugged directly into an Arduino Mega without any rework. All through-hole and surface-mount parts are installed by the chosen assembly house or by the grantee. Units should be electrically tested before being considered complete. Directions for ordering new printed boards, and/or how to port the design to a different fab facility, should be committed to the `krabby-research/assets/` directory so that any individual can order and/or fabricate their own Krabby-Uno boards.

Because each robot requires **3x shields** and the milestone targets **two complete Krabby-Uno robots**, the total production quantity under this task is **6 shields**. Shields should be produced as a single manufacturing lot so that all units are identical.

#### Acceptance Criteria

- Pre-build review step completed and documented before ordering PCBs/assembly
- Fabricated, assembled, and electrically tested shields delivered (quantity: **6 units**, providing two full sets of 3 per robot)
- Tests include power/continuity checks and pin-mapping verification against the design
- PCB fabrication and assembly is managed with a reputable vendor and results in fully assembled, ready-to-plug shields
- Each shield passes basic electrical bring-up:
  - Visual inspection for obvious assembly defects (solder bridges, missing parts, reversed connectors)
  - Continuity checks for all DB-19 pins back to the correct Arduino Mega pins
  - Verification that no DB-19 pins are shorted to adjacent pins, logic VCC, or GND unless intended by the schematic
- A simple test report (spreadsheet or PDF) is provided listing each shield serial/label and pass/fail results for continuity/pin-mapping checks
- Any substitutes are pre-approved and documented; as-built lot/substitute records provided
- Boards are packaged for shipment with basic ESD protection and labeling

---

### Task 4 - Fabricate, Assemble, and Test Combined 3-Actuator H-Bridge PCBs

#### Narrative

This task covers fabrication, assembly, and electrical verification of the H-bridge design from Task 2. A pre-build review and approval of the manufacturing package is required before placing the order.

As with the shields, selecting and managing a PCB fab/assembly vendor, ensuring parts availability, and documenting any proposed substitutes are all in scope. The intent is that this milestone demonstrates a realistic path for the community to obtain working boards from common PCB services, with enough documentation that others can reproduce the process.

Each robot requires **6 H-bridge boards** (one per group of 3 actuators), and the milestone targets **two complete Krabby-Uno robots**, for a total of **12 assembled H-bridge boards** delivered under this task.

Bench testing is required on every assembled board before it is considered complete. If physical actuators are not available, testing may be done with representative resistive or electronic loads that approximate expected current draw. The outcome should be that the boards can be plugged into a Krabby-Uno robot and immediately begin driving actuators without obvious manufacturing or assembly issues.

#### Acceptance Criteria

- Pre-build review step completed and documented before ordering PCBs/assembly
- Fabricated, assembled, and electrically tested H-bridge boards delivered (target: **12 units**, providing two full sets of 6 boards for 18 actuators each)
- Smoke/load test performed on each board to validate the three BTS7960-class channels and IS/EN/PWM signal paths
- Any substitutes are pre-approved and documented; as-built lot/substitute records provided
- Boards are packaged for shipment with basic ESD protection, labeling, and test/QA evidence
  - At minimum, per-board test notes indicate the supply voltage, load used, and confirmation that all three channels respond correctly to EN/PWM and report IS as expected
  - Clear labeling on each board so physical boards can be correlated with test/QA records

#### Expected fabrication cost (non-binding)

- For the quantities above (6 shields and 12 H-bridge boards) using low-cost PCB fab + assembly services (e.g., JLCPCB, PCBWay), total fabrication + assembly + parts cost is expected to be on the order of **$400–700 USD**, depending on final BOM, board size, and vendor choices.
- Actual quotes and final costs should be documented alongside the manufacturing notes so future builders have a realistic reference.

---

## FAQ

**Q: Does DB-19 carry motor power?**  
A: No. DB-19 is strictly for control and feedback signals. Motor power enters the H-bridge board via a dedicated 2-pin locking connector.

**Q: How do actuators connect to the H-bridge board?**  
A: Each actuator provides a 1x5-pin connection carrying two motor wires and three rotary potentiometer wires.

**Q: Is firmware or Arduino code included in this milestone?**  
A: No. This milestone is hardware only.

**Q: Who assembles and tests the boards?**  
A: This milestone includes fabrication, assembly, and electrical verification of the PCBs. The design files, manufacturing notes, and test/QA evidence should be committed to the repository so that both the original grantee and future community members can reproduce the process and validate boards.

**Q: Is CAN bus part of this work?**  
A: No. CAN is not part of this milestone and should not be assumed in the design.

**Q: What about other Arduino Mega pins that aren't used here?**  
A: As much as reasonably possible, the shield design should avoid permanently blocking or repurposing unused Arduino Mega pins. Any header positions that are not needed for the DB-19 or serial mappings above should remain accessible (or be re-exposed via through-headers or test pads) so builders can still attach Dupont leads or additional hardware for debugging, expansion, or future milestones.

---

## Appendix A - Arduino Mega to DB-19 Pin Mapping

This appendix documents a concrete mapping from the existing Arduino Mega pinout in the leg controller firmware to the DB-19 connectors on the shield. It is written so that a PCB designer can drop it straight into a schematic and silkscreen, and so that firmware contributors can quickly see how hardware pins line up with logical joints.

The current firmware (`krabby-research/firmware/arduino/controller.ino`) instantiates six linear actuators with the following Arduino pins:

- `LHY`: PWM_R = 2, PWM_L = 3, EN_R = 22, EN_L = 23, IS = A6, POT = A0  
- `LHL`: PWM_R = 4, PWM_L = 5, EN_R = 24, EN_L = 25, IS = A7, POT = A1  
- `LKL`: PWM_R = 6, PWM_L = 7, EN_R = 26, EN_L = 27, IS = A8, POT = A2  
- `RHY`: PWM_R = 8, PWM_L = 9, EN_R = 28, EN_L = 29, IS = A9, POT = A3  
- `RHL`: PWM_R = 10, PWM_L = 11, EN_R = 30, EN_L = 31, IS = A10, POT = A4  
- `RKL`: PWM_R = 12, PWM_L = 13, EN_R = 32, EN_L = 33, IS = A11, POT = A5  

**Note:** As of this document, the firmware still exposes direction-specific enable pins (`EN_R`, `EN_L`) per motor. The hardware mapping below is defined in terms of a *single* per-motor `EN` signal (e.g., EN1, EN2, EN3) brought out on its own Arduino pin (sequentially D22–D27 for the six motors). Firmware will be updated to match this single-EN-per-motor convention; until that change lands, this appendix should be treated as the target/authoritative mapping for new hardware.

The shield provides **two DB-19 connectors**, one per leg (three actuators per connector). For each DB-19:

- 6x PWM lines (two per motor: R/L)
- 3x EN lines (one per motor, not directional)
- 3x IS lines (one per motor)
- 3x POT lines (one per motor)
- 1x logic VCC
- 1x logic GND
- 2x spare pins

Each motor has a dedicated EN pin on the Arduino (no separate EN_R/EN_L on the shield). The downstream 3‑actuator H-bridge PCB is responsible for using that single EN to gate both BTS7960 enable inputs as needed.

### DB-19 Pin Numbering Convention

For both connectors, the DB-19 pins are assigned as follows (per connector):

- Pin 1:  PWM1_R  
- Pin 2:  PWM1_L  
- Pin 3:  PWM2_R  
- Pin 4:  PWM2_L  
- Pin 5:  PWM3_R  
- Pin 6:  PWM3_L  
- Pin 7:  EN1  
- Pin 8:  EN2  
- Pin 9:  EN3  
- Pin 10: IS1  
- Pin 11: IS2  
- Pin 12: IS3  
- Pin 13: POT1  
- Pin 14: POT2  
- Pin 15: POT3  
- Pin 16: logic VCC  
- Pin 17: logic GND  
- Pin 18: spare (reserved for future use)  
- Pin 19: spare (reserved for future use)  

### Left Leg DB-19 Connector (LHY, LHL, LKL)

Mapping for the **left leg** DB-19 connector (`LHY`, `LHL`, `LKL`):

- Motor 1 (LHY)  
  - DB-19 Pin 1 (PWM1_R)  → Arduino D2  
  - DB-19 Pin 2 (PWM1_L)  → Arduino D3  
  - DB-19 Pin 7 (EN1)     → Arduino D22  
  - DB-19 Pin 10 (IS1)    → Arduino A6  
  - DB-19 Pin 13 (POT1)   → Arduino A0  

- Motor 2 (LHL)  
  - DB-19 Pin 3 (PWM2_R)  → Arduino D4  
  - DB-19 Pin 4 (PWM2_L)  → Arduino D5  
  - DB-19 Pin 8 (EN2)     → Arduino D23  
  - DB-19 Pin 11 (IS2)    → Arduino A7  
  - DB-19 Pin 14 (POT2)   → Arduino A1  

- Motor 3 (LKL)  
  - DB-19 Pin 5 (PWM3_R)  → Arduino D6  
  - DB-19 Pin 6 (PWM3_L)  → Arduino D7  
  - DB-19 Pin 9 (EN3)     → Arduino D24  
  - DB-19 Pin 12 (IS3)    → Arduino A8  
  - DB-19 Pin 15 (POT3)   → Arduino A2  

- Common pins  
  - DB-19 Pin 16 (VCC)    → 5V logic supply (shield net shared with Arduino 5V)  
  - DB-19 Pin 17 (GND)    → logic GND (shield net shared with Arduino GND)  
  - DB-19 Pins 18–19      → left unconnected on the initial revision; reserved for future signals  

### Right Leg DB-19 Connector (RHY, RHL, RKL)

Mapping for the **right leg** DB-19 connector (`RHY`, `RHL`, `RKL`):

- Motor 1 (RHY)  
  - DB-19 Pin 1 (PWM1_R)  → Arduino D8  
  - DB-19 Pin 2 (PWM1_L)  → Arduino D9  
  - DB-19 Pin 7 (EN1)     → Arduino D28  
  - DB-19 Pin 10 (IS1)    → Arduino A9  
  - DB-19 Pin 13 (POT1)   → Arduino A3  

- Motor 2 (RHL)  
  - DB-19 Pin 3 (PWM2_R)  → Arduino D10  
  - DB-19 Pin 4 (PWM2_L)  → Arduino D11  
  - DB-19 Pin 8 (EN2)     → Arduino D26  
  - DB-19 Pin 11 (IS2)    → Arduino A10  
  - DB-19 Pin 14 (POT2)   → Arduino A4  

- Motor 3 (RKL)  
  - DB-19 Pin 5 (PWM3_R)  → Arduino D12  
  - DB-19 Pin 6 (PWM3_L)  → Arduino D13  
  - DB-19 Pin 9 (EN3)     → Arduino D27  
  - DB-19 Pin 12 (IS3)    → Arduino A11  
  - DB-19 Pin 15 (POT3)   → Arduino A5  

- Common pins  
  - DB-19 Pin 16 (VCC)    → 5V logic supply (shield net shared with Arduino 5V)  
  - DB-19 Pin 17 (GND)    → logic GND (shield net shared with Arduino GND)  
  - DB-19 Pins 18–19      → left unconnected on the initial revision; reserved for future signals  

This mapping keeps the current `controller.ino` joint ordering, satisfies the DB-19 signal counts defined in Task 1, and sets a clear precedent for future firmware refactors that may move to single-EN control per motor in software as well.

---

## Appendix B - Serial Communication Breakouts Between Arduinos

Milestone 4 defines a three-Mega topology (one leader, two followers) where the leader communicates with followers over serial links. To support that wiring cleanly, the shield should break out two hardware serial ports from the Arduino Mega to locking connectors.

### Reserved Arduino Serial Pins

Use the Arduino Mega's additional hardware UARTs for inter‑MCU links:

- `Serial1`: TX1 = D18, RX1 = D19  
- `Serial2`: TX2 = D16, RX2 = D17  

These four signals are reserved for leg‑pair communication in the three‑Mega configuration described in **Milestone 4 - MCU Firmware**. `Serial` (USB) remains reserved for host (Jetson Orin) communication.

### Shield Connectors

The shield exposes these serial lines on two locking 3-pin headers (e.g., latching 3-pin Molex/KF series) so that each inter‑MCU cable carries TX, RX, and a shared ground:

- `SERIAL_A` connector (for one follower Mega)  
  - Pin 1: TX1 (Arduino D18)  
  - Pin 2: RX1 (Arduino D19)  
  - Pin 3: GND (logic ground, common between boards)  

- `SERIAL_B` connector (for the other follower Mega)  
  - Pin 1: TX2 (Arduino D16)  
  - Pin 2: RX2 (Arduino D17)  
  - Pin 3: GND (logic ground, common between boards)  

The mating cable between boards must **cross TX and RX**:

- Leader TX1 → Follower RX1, Leader RX1 → Follower TX1 (with GND–GND straight‑through)  
- Leader TX2 → Follower RX2, Leader RX2 → Follower TX2 (with GND–GND straight‑through)  

All Megas share a common ground through these connectors and/or their power wiring. This arrangement matches the intent of Milestone 4 (a leader Mega with dedicated serial channels to two followers), keeps the USB/`Serial` port free for host communication, and makes the inter‑MCU wiring repeatable and enclosure‑friendly.


-

I have an 18 actuator hexapod robot that uses 5 wire linear actuators for each joint (2 wire for motor, 3 wire for potentiometer), anywhere from 12v to 48v motors. I currently use BTS9670 hbridge, one per actuator, w/ 10 wire coming off h-bridge going to Arduino mega (each mega controls 6 actuator). the 10 wire are: R/L_IS, R/L_EN, R/L_PWM, VCC, GND, M+/- (24v). The wiring is super annoying between the h-bridge and mega, and there is no easy way to make a case around it because there is wires all over the place. Instead, I want to hire some one to make a 2x simple PCB:
#1) DB-19 - Arduino-mega shield that reroutes pins from existing arduino mega to 2x DB-19, 1 serving each set of 3x limbs. There will be ~17 pin from arduino to each leg (3x actuator), L1/2/3 IS (3x), L1/2/3 EN (3x), L1/2/3 R/LPWM (6x), L1/2/3 POT (3x), VCC (1x), GND (1x), leaving 2 empty pins for future use. 
#2) Combined h-bridge - Combine 3x BTS79760 into one, and reroute pin out so they all connect via single DB-19 connector + 3x latching molex via pinout provided above. 



