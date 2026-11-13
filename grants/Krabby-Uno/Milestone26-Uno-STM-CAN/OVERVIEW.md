# Patina Foundation Grant — Krabby-Uno Milestone 26: Uno STM CAN Firmware

## Grant Overview

Move the Krabby software onto the new Krabby Uno v0.3 leg boards. Each leg gets its own STM32 board, and the Orin talks to the boards over CAN FD. The GUI, gamepad, and AI model control the boards through a new CAN SDK. Denny has already brought up the real board and flashing over CAN. This milestone builds the same path against the simulated robot, so engineers and AI agents can test it without hardware, then spends two support weeks fixing what breaks when it meets the real robot.

Today three Arduino Megas drive the robot. One Mega connects to the Orin over USB and relays every command and telemetry line for the other two, and the boards elect these roles at boot. The v0.3 hardware removes the need for all of this, since each board has its own CAN connection and a fixed ID set by a switch, but the board has no firmware yet.

The new firmware is small and written from scratch. It does three things: it receives a duty cycle for each of the leg's three joints and drives them, it sends back each joint's raw pot reading, signed hall count, and calibrated position from 0.0 to 1.0, and it calibrates a joint on command by driving it to both ends and saving the readings. A leg is identified by its CAN ID, with legs 1, 3, and 5 on the left and 2, 4, and 6 on the right, from front to rear. A joint is identified by its ID within the leg: 1 = yaw, 2 = hip, 3 = knee. Duty runs from -1.0 to 1.0, where the sign is direction and the magnitude is power, and a duty of 0 holds the joint. A DBC file defines every CAN message, and both the firmware and the Python tools build on that one file. Joint names like "front left hip yaw" live only in the GUI and the robot definition. Role election, joint name strings, position targets, and the hold and version commands are all gone from the firmware.

Task 1 writes the firmware handlers and a new CLI, then controls the simulated robot from that CLI over a virtual CAN bus. Task 2 connects the GUI, HAL, gamepad, and AI model to the CAN SDK behind an `--stm` option and tests them against the sim. Tasks 3 and 4 are 20-hour support weeks, working bugs and gaps that Denny and Nick find on the real robot.

At the end, the GUI, gamepad, and AI model drive the simulated robot over CAN with the same SDK the Orin uses, and the problems found on the real robot during the support weeks are fixed or listed with estimates.

## Why is this Important?

- The v0.3 boards cannot run a motor until this firmware exists, so the robot cannot move to the new electronics.
- One serial link and one leader board currently carry all 18 joints, so a leader failure takes down the whole robot. On CAN each board acts on its own commands and sends its own telemetry.
- Replacing the Mega hardware would otherwise break the Milestone 21 simulation. This milestone keeps firmware and the tools above it testable without a robot, for both engineers and AI agents.

## Tasks

Total: ~18 working days, roughly 3-4 weeks part-time at 20-25 hours per week. Implementation detail and exact code locations live in the per-task files linked below; the acceptance criteria are here.

### Task 1 - CAN firmware and CLI against the sim → [TASK-1-STM-CAN-SIM.md](TASK-1-STM-CAN-SIM.md)

#### Narrative

Write the firmware and CLI, then use them to move the simulated robot over virtual CAN. This task needs no hardware. Work runs from the bottom of the stack up, and each step gives something to check before the next one starts. This requires 1) a DBC file that defines the CAN messages, and a new CLI with a command that sends a duty, 2) `can_handler.c`, which receives the duty and drives the joints, 3) `telemetry_handler.c`, which sends pot, hall, and firmware version back, 4) a simulated version of the board's pins, so six copies of these same two files can drive the simulated robot in IsaacSim. The code that touches the real board's pins is Denny's and is not part of this milestone.

The Mega firmware, its CLI, and its simulation stay as they are, because the existing robots still depend on them.

#### Acceptance Criteria

- 1a - `krabby-firmware-stm duty 1 0.5 0 0` sends an FD frame that `candump vcan0` shows with ID `0x100`, a leg 1 joint 1 duty of 0.5, and every other joint at 0.
- 1b - Unit tests show that a board drives its joints in the requested direction and power, ignores the values for other legs, and sends telemetry every 50 ms and its firmware version every 1000 ms.
- 1c - With six simulated boards on `vcan0`, `krabby-firmware-stm duty 1 1.0 0 0` moves only leg 1's yaw joint in IsaacSim. `krabby-firmware-stm telemetry` shows that joint's pot and hall values changing at 20 Hz and the firmware version at 1 Hz. When the duty reverses, the hall count reverses too.
- 1d - An integration test performs the check in 1c without a window and passes under `make -C firmware test-firmware-isaac`.
- 1e - `krabby-firmware-stm calibrate 1 2` drives leg 1's hip to its extended and retracted stops in IsaacSim and saves both ends. Afterwards, telemetry shows the raw pot and hall values and a position of 0.0 at the retracted stop, rising toward 1.0 as the hip extends, and the calibration survives a simulated board restart.
- 1f - `make -C firmware compile-firmware`, `test-native`, and `test-firmware` still pass with no changes to the Mega firmware.

#### Time estimate (~5 days)

| Days | Sub-task title |
|------|----------------|
| 0.5 | DBC file, `connect()`, and the `duty` command |
| 1 | `can_handler.c` and its unit tests |
| 1 | `telemetry_handler.c` and its unit tests |
| 1 | `calibration.c`, the `calibrate` command, and its unit tests |
| 1.5 | Simulated pins, virtual CAN, and the Isaac test |

### Task 2 - GUI, HAL, gamepad, and model on --stm → [TASK-2-CLIENT-REWIRE.md](TASK-2-CLIENT-REWIRE.md)

#### Narrative

Control the simulated robot from the GUI, the gamepad, and the AI model. Each of these reaches the firmware through the serial SDK, so `--stm` swaps in the CAN SDK at the point where they create it, and everything above that point stays the same. On this path the HAL sends duty straight through and does no radian conversion. This requires 1) an `--stm` option on the GUI, 2) a CAN version of the HAL's motor class, 3) an `--stm` option on the HAL server and the gamepad and model mappers, passed through from `krabby run`, 4) a CAN transport for the Isaac HAL's fullsim backend, 5) the CAN Python packages in the release image.

Without `--stm`, every tool behaves as it does today, so one image serves both Mega and v0.3 robots.

#### Acceptance Criteria

- 2a - `python -m firmware.gui --stm` jogs joints on the simulated robot over `vcan0` and shows live pot and hall values. Without the flag, the GUI still uses the serial SDK and its existing tests pass.
- 2b - With the Isaac HAL running `--backend fullsim --firmware-transport can`, the gamepad moves the simulated robot's joints, and `candump vcan0` shows the SetDuty frames.
- 2c - In the same setup, the AI model's commands move the simulated robot's joints.
- 2d - Unit tests show that a full `JointCommand` reaches the right leg and joint as duty, with no radian conversion, and that the gamepad and model mappers keep today's output without `--stm`.

#### Time estimate (~3 days)

| Days | Sub-task title |
|------|----------------|
| 0.5 | GUI `--stm` option |
| 1 | HAL CAN motor class, `--stm` options, and mapper changes |
| 0.5 | Isaac fullsim CAN transport |
| 0.5 | Release image and `krabby run` changes |
| 0.5 | Testing the GUI, gamepad, and AI model against the sim |

### Task 3 - Support week 1 → [TASK-3-SUPPORT-WEEK-1.md](TASK-3-SUPPORT-WEEK-1.md)

#### Narrative

Spend 20 hours fixing what breaks when the Task 1 and Task 2 work meets Denny's real boards and Nick's robot work. Work comes from one list in `SUPPORT-LOG.md`, which Denny and Nick prioritize at the start of the week. The list starts with testing CAN on the Orin's own ports, moving Denny's control GUI into `krabby-firmware`, and publishing it to PyPI, so anyone can install it on an Orin and drive a board.

#### Acceptance Criteria

- 3a - On an Orin, `pip install krabby-firmware` from PyPI installs the CAN tools. With a board plugged into `can0` or `can1`, Denny's control GUI lists the board, shows its telemetry, and drives its motors.
- 3b - `SUPPORT-LOG.md` lists every reported item with its reporter, problem, estimate, and whether it needs the robot.
- 3c - 20 hours are logged against the list, worked in the priority order Denny and Nick set, with actual hours recorded per item.
- 3d - Each finished item is shown fixed by a test or by repeating the step that failed, with the result recorded.
- 3e - A summary of finished items, hours, and the top of the remaining list is posted in Discord at the end of the week.

#### Time estimate (~5 days)

| Days | Sub-task title |
|------|----------------|
| 5 | 20 hours of support work from the list |

### Task 4 - Support week 2 → [TASK-4-SUPPORT-WEEK-2.md](TASK-4-SUPPORT-WEEK-2.md)

#### Narrative

Spend a second 20 hours on the same list, re-prioritized by Denny and Nick at the start of the week.

#### Acceptance Criteria

- 4a - 20 hours are logged against the list, worked in priority order, with actual hours recorded per item.
- 4b - Each finished item is shown fixed by a test or by repeating the step that failed, with the result recorded.
- 4c - The list is left current, with finished, returned, and remaining items each clear, and the remaining items carry estimates for the next milestone.

#### Time estimate (~5 days)

| Days | Sub-task title |
|------|----------------|
| 5 | 20 hours of support work from the list |

## Information

- Repositories: `krabby-research` (firmware, HAL, simulation, images), `patina-foundation-grants` (this grant), `krabby-contracts` (the milestone ICA, M26 folder not yet created).
- Hardware: Tasks 1 and 2 need only a Linux machine with IsaacSim. The support weeks use Denny's v0.3 boards and an Orin. `hardware/Uno-v0.3/` holds the board's schematic and EasyEDA project.
- Mega firmware that this replaces: `firmware/arduino/arduino.ino`, `actuator_manager.h`, and `hall_hw.cpp`.
- Host tools that this replaces: `firmware/cli.py`, `firmware/krabby_mcu.py`, and `firmware/mcu_port.py`.
- Simulation: `firmware/sim/`. Start with `ARCHITECTURE.md`, `robot_core.py`, and `simulated_actuator.py`.
- Code that creates the SDK: `firmware/gui/app.py`, `hal/server/jetson/hal_server.py`, and `hal/server/isaac/hal_server.py`. Code that builds `JointCommand`: `controller/mappers/gamepad_to_krabby_hal_mapper.py` and `compute/parkour/mappers/model_to_hardware.py`.
- Previous milestone for the support-week format: Milestone 23, Task 2.
- Not part of this milestone: the real-board firmware and CAN flashing (done by Denny), installing the other four boards, making the robot walk, the Qwiic IMU, the Qwiic screen, battery monitoring, motor current telemetry, and position control.

## FAQ

1. Why write the firmware from scratch instead of porting the Mega code? Most of the Mega code exists for features this firmware drops: the serial parser, role election, telemetry relay, and the position controller. Only the rule for driving a motor and the auto-calibration sequence carry over, and both are short.

2. Can the robot walk with duty commands alone? Yes, because it already does. The HAL's position command path is commented out in `hal/server/jetson/krabby_mcusdk.py`, since not every joint has a pot connected, and production sends open-loop jog PWM. Duty is that same command on a -1.0 to 1.0 scale. Position control can return on the Orin using the calibrated position in telemetry.

3. What happens if the Orin stops sending commands? Each joint keeps running at its last duty, the same way a jog does on the Mega today. The firmware has no command timeout. Its watchdog only covers a firmware lockup, which resets the board and puts the motors to sleep.

4. The DBC is meant to generate code for both sides, so why is there no generated Python? `cantools` generates C only. The firmware compiles that generated C, and the Python tools load `krabby.dbc` with `cantools` when they start. Both sides still read the message layout from the one file.

5. Why does one SetDuty frame carry all 18 joints? The HAL already sends a full 18-joint command each tick, so one frame updates every joint at the same moment, and the DBC needs one SetDuty message instead of six. Telemetry still needs an ID per leg, because a CAN bus requires that no two boards send the same ID.

6. Why CAN FD instead of classic CAN? Later milestones will add messages, such as motor current, IMU, and battery state, that need more room, and moving every board and tool to FD later would mean touching all of them again. FD frames hold up to 64 bytes, so one frame already carries a whole leg's telemetry. Arbitration runs at 1 Mbit/s and data at 5 Mbit/s. The Orin, the board's transceiver, `vcan`, and python-can all support FD.

7. Why send every command on both buses? The CLI then needs no record of which leg is on which bus, and each board reads only its own leg's values. Wiring a leg to the other bus requires no software change.

8. Why do hall counts differ from the Mega's? The Mega counts both edges of hall A and uses hall B only for direction. The STM32 timers count both edges of both signals, so the same motion produces twice as many counts. The two numbers should never be compared directly.

9. Can this be developed on Windows? Partly. The unit tests use python-can's in-process virtual bus and run on Windows. `vcan0` and SocketCAN exist only on Linux, so the IsaacSim tests need a Linux machine.

10. Why does the HAL no longer convert radians? The values the HAL calls radians already act as duty. It multiplies them by 510 to get jog PWM, and the gamepad mapper scales full stick to 0.5 for that reason. On `--stm`, `JointCommand` carries duty directly. How a trained model's actions become duty for the linear actuators is decided with the next model release.

11. How will Qwiic devices work? The primary IMU comes from the ZED camera and does not touch CAN. A Qwiic IMU, the Qwiic screen, or the battery monitor can plug into the Qwiic connector on any of the six boards. The board detects the device at startup, then forwards its readings onto the bus, or for the screen, reads the robot's state off the bus. None of this is in this milestone, but the Qwiic connector shares I2C1 with the DAC7574 at `0x4C`, so the firmware's I2C code must not assume the DAC is the only device on the bus.
