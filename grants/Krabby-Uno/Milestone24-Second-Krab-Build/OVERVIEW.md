# Patina Foundation Grant — Krabby-Uno Milestone 24: Second Krab Build

## Grant Overview
Get two Krabby robots working. Finish robot 1 — mount the six hip yaw motors, install the Orin, 24 V battery, and Bluetooth controller, and fix the two firmware bugs (only one actuator moves at a time; an encoder path locks up the Arduino) — until it walks stably under a gamepad. Then build out robot 2: complete the partly assembled leg modules, build the body, validate every joint using robot 1's known-good motor controllers, and stand it up on its own Jetson with a second set of motor controllers using the older v0.1 boards and their `KRABBY_PIN_REV` 2 pin map. Finish by installing cameras on both robots and enrolling both into the fleet manager, with simultaneous telemetry and a teleop session against each.

## Why is this Important?

- Robot 1 is close and needs to get finished. Once it walks stably, Nick can put the locomotion model on it and we can see it working. M15, M18, and teleop are all waiting on that.
- Robot 2 is mostly parts we already have, and enables Nick to have a robot and our EE to have a robot.
- The fleet manager has only ever run with one robot connected. Two robots is the first real test of enrollment, simultaneous telemetry, and teleop reaching the right device.

## Tasks
Total: ~1 month (1 week per task, ~5 part-time working days each). Tasks run in order: Task 2 borrows Task 1's repaired controllers, Task 3 needs Task 2's assembled robot, and Task 4 needs both robots running.

### Task 1 - Fix robot 1 and get it running stably -> [TASK-1-ROBOT-1-BRINGUP.md](TASK-1-ROBOT-1-BRINGUP.md)
#### Narrative
Finish robot 1 and get it stable. Fabricate and mount the six hip yaw motors on their slot boards and lever arms, test all motor crimps (power and POT), and install the Orin, 24 V battery, and Bluetooth controller and validate all 18 legs can be controlled via the bluetooth controller. If not, do some basic triage/debugging and let Fletcher know if any help is needed root causing any controller or other issues, and soak some of the debugging time that was set aside to root cause the issue and fix it. Then fix a regression in the firmware, only one actuator moves at a time, and a second issue with the yaw encoder wiring that sends too many interrupts and locks up the Arduino. The task document lists the four most likely causes of the single-actuator bug — a stuck calibration state machine, a blocking batch-jog parser, an all-or-nothing target parser, and a failed role election — with file and line references for each, plus the blocking serial calls behind the lockup.

#### Acceptance Criteria

- 1a - All six hip yaw motors are mounted on their slot boards with lever arms fitted, and every yaw joint sweeps its full range by hand with clearance from the chassis.
- 1b - Every motor lead is crimped, tug-tested, and labeled, and the Orin, 24 V battery, and controller are installed on the robot and run without a bench supply.
- 1c - The root cause of the one-motor-at-a-time regression is identified in the firmware, named by file and function, fixed, and covered by a test that fails against the unfixed code.
- 1d - The encoder lockup is reproduced, root-caused, and fixed (likely rewiring the encoder to use the correct slower tick encoder wire)

#### Time estimate (~5 days)

| Days | Sub-task title |
|------|----------------|
| 1 | Mount six hip yaw motors, fit lever arms, crimp and label leads |
| 1.5 | Install the Orin, 24 V battery, and controller, and run the robot off its own power, debug setup issues |
| 1.5 | Root-cause and fix the one-motor-at-a-time regression, with a regression test |
| 1 | Root-cause and fix the encoder lockup |


### Task 2 - Finish robot 2's legs and body -> [TASK-2-ROBOT-2-MECHANICAL.md](TASK-2-ROBOT-2-MECHANICAL.md)
#### Narrative
Finish robot 2 mechanically. Complete the six leg modules (bearings, pivots, femur, knee, actuators, yaw gearmotor), build the body, join the legs to it, and run the standing check: all six feet flat under the robot's own weight, every joint sweeping its full range by hand. Then validate every joint by borrowing robot 1's motor controllers one board at a time and driving each leg through its range. If a leg doesn't move under a controller that already works on robot 1, the problem is the leg or the harness, and it gets found now instead of in Task 3 when the untested rev 2 boards are also in play.

#### Acceptance Criteria

- 2a - All six leg modules for robot 2 are assembled.
- 2b - The body is built and all six legs are attached, with legs on the correct sides and each yaw sweeping clear of the chassis.
- 2c - Robot 2 passes the standing check: all six feet touch flat under its own weight and every joint sweeps its full range.
- 2d - Every one of robot 2's 18 joints is driven through its range using motor controllers borrowed from robot 1, in both directions, with the leg identified for each test.
- 2e - Robot 1 is reassembled with its own controllers and still passes the Task 1 walk test after the borrowed hardware is returned.

#### Time estimate (~4 days)

| Days | Sub-task title |
|------|----------------|
| 1 | Finish the six leg modules: bearings, pivots, femur, knee, actuators, yaw gearmotor |
| 1 | Build the body and mount the hinges |
| 1 | Join the legs to the body and run the standing check |
| 1 | Validate all 18 joints with robot 1's controllers, then restore robot 1 |

### Task 3 - Second Jetson and second controller set -> [TASK-3-ROBOT-2-ELECTRONICS.md](TASK-3-ROBOT-2-ELECTRONICS.md)
#### Narrative
Set up robot 2's electronics: the second Jetson and a second set of motor controllers, using the older v0.1 boards already on hand. The Jetson follows the same install path as robot 1 (base image, `krabby-launcher`, `krabby install`, `krabby run`). The v0.1 boards use the older pin map, which the firmware already supports as `KRABBY_PIN_REV` 2, so build and flash with `PIN_REV=2` — the published firmware artifact is a rev 3 build and will drive the wrong actuators on these boards. Rev 2 also has no hall wiring, so anything that assumes hall feedback needs to handle the missing sensor.

This task includes: second Jetson setup from base image to running stack, installing the three v0.1 boards with enclosures, harness, and 24 V power, building and flashing rev 2 firmware to all three boards, confirming role election and checking all 18 channels against Task 2's validation table, and fixing whatever rev-2-specific problems turn up.
#### Acceptance Criteria

- 3a - The second Jetson is set up from its base image through to a running stack, with every deviation from the documented path recorded.
- 3b - Three v0.1 motor controller boards are installed on robot 2 in their enclosures, wired to all 18 actuators through labeled connectors, and powered from a 24V AC->DC converter.
- 3c - Firmware built for `KRABBY_PIN_REV` 2 is flashed to all three boards and motors are controllable/correct. Identify and fix (or report if unable to fix) any regressions in current firmware that's preventing the older pin configs from working.
- 3d - Robot 2 walks under gamepad control and its own Jetson.

#### Time estimate (~5 days)

| Days | Sub-task title |
|------|----------------|
| 1 | Set up the second Jetson from base image to running stack |
| 1 | Install the three v0.1 boards, enclosures, harness, and 24 V power |
| 1.5 | Build and flash `KRABBY_PIN_REV` 2 firmware, fix rev-2 drift |
| 0.5 | Whole-robot channel check on all 18 joints |
| 1 | Gamepad walk test and any fixes/regressions/debugging |

### Task 4 - Cameras and fleet manager on both robots -> [TASK-4-CAMERAS-AND-FLEET.md](TASK-4-CAMERAS-AND-FLEET.md)
#### Narrative
Install and test the cameras and the fleet manager on both robots. Mount each robot's front depth camera and side cameras (cutting chassis openings where needed), bring them up through the Jetson HAL sensor catalog, and commit the configuration so it survives a fresh install. Then enroll both robots into the fleet manager and test them together: both showing live telemetry in the portal at the same time, telemetry attributed to the right robot, and a teleop session against each with video and control. The fleet path has never run with two devices, so expect to fix things here.

#### Acceptance Criteria

- 4a - Each robot has its front depth camera mounted and streaming RGB and depth through the Jetson HAL, surviving a reboot without a person re-plugging anything.
- 4b - Side cameras are mounted on both robots and stream through the same HAL sensor catalog path.
- 4c - Followed the guide for setting up camera in `krabby-research` and updated any additional steps so the configuration is reproducible on a fresh install.
- 4d - Both robots enroll with distinct identities and report live telemetry in the fleet portal at the same time, with each robot's telemetry attributed to the right robot.
- 4e - Test whether a teleop session runs against each robot with video and control, reaching the robot it was aimed at, and a session against one robot while both are reporting telemetry does not disturb the other. Report list of issues to Bruce/Fletcher.

#### Time estimate (~5 days)

| Days | Sub-task title |
|------|----------------|
| 1.5 | Mount and bring up the front depth camera on both robots |
| 1 | Mount and bring up the side cameras and commit the sensor catalog config |
| 1 | Enroll both robots and confirm simultaneous live telemetry |
| 1 | Teleop against each robot, including concurrent operation |
| 0.5 | Reboot and reconnect checks on both robots |

## Information

- Repositories: `krabby-research` (firmware, HAL, controller, the `krabby` CLI, fleet client, and hardware design files), `patina-foundation-grants` (this grant), `krabby-contracts` (the milestone ICA).
- Firmware: `firmware/arduino/arduino.ino` (main loop, role election, command fan-out), `actuator_manager.h` (per-actuator drive, calibration state machine, telemetry), `command.h` (target parsing), `hall_hw.cpp` (pin-change interrupt handlers), `board_pins.h` (the three `KRABBY_PIN_REV` pin maps), and `firmware/Makefile` (`PIN_REV`).
- Host side: `firmware/krabby_mcu.py`, `firmware/mcu_port.py`, `firmware/SETUP.md`, and `firmware/MOTOR_HEADER_PINOUT.md` for the rev 3 header map and the UART assignment.
- HAL and control: `hal/server/jetson/hal_server.py`, `krabby_mcusdk.py`, `front_camera_factory.py`, `zed_camera.py`, and the gamepad path in `controller/scripts/jetson/E2E_GAMEPAD_KRABBY.md` and `CONNECT_PRO_CONTROLLER.md`.
- Fleet: `fleet/cli/`, `fleet/service/`, `fleet/portal/`, `fleet/ENROLL.md`, and `fleet/SETUP-FLEET.md`, with the on-robot side in `krabby/enroll.py`, `krabby/agent.py`, and `krabby/telemetry.py`.
- Hardware: `hardware/Uno-v0.1/` holds the older shield and power board design files whose pin map is `KRABBY_PIN_REV` 2, and `hardware/Uno-v0.2/` holds the current boards and the BOM.
- Bring-up path and known issues: Milestone 23 owns the cold-start command sequence and the friction log format, in [Milestone23-FleetReliability-QOL](../Milestone23-FleetReliability-QOL/TASK-1-BENCH-TEST-AND-BRINGUP.md) and that folder's `KNOWN-ISSUES.md`.

## Looking ahead

With two robots, work stops queuing on one machine. One can sit on the M23 CI bench while the other stays free for locomotion and vision work, and a hardware failure stops one workstream instead of everything.

The v0.1 boards on robot 2 put shelf hardware to use, and both robots stay candidates for the Milestone 20 STM32G474 per-leg boards once that design is fabbed and has firmware. Running two pin revisions in one fleet now also shakes out the mixed-revision problems before that retrofit starts.

## FAQ

1. Why put the older v0.1 boards on robot 2 instead of ordering more v0.2 boards? They are already on the shelf, the firmware already has their pin map (`KRABBY_PIN_REV` 2), and a fab order adds weeks of lead time.

2. Why test robot 2's legs with robot 1's controllers first? It separates leg problems from board problems. A leg that works under a proven controller is mechanically good, so when the rev 2 boards go in and something fails, the board revision is the suspect.

3. Why do cameras and the fleet manager come last? They need robots that already move reliably — debugging a camera driver on a robot that hangs mid-run means debugging both at once. Doing it last also means the fleet test runs against two robots, which is the test that matters.

4. What counts as stable? 5-10 minutes commanded motion on the robot's own battery with no board hang, no lost joint, and no manual restart. Any reliability issues outside of cables/controller should be logged for later fixes.
