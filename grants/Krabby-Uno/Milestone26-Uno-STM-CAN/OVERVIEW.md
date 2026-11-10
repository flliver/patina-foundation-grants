# Patina Foundation Grant — Krabby-Uno Milestone 26: Uno STM CAN Firmware

## Grant Overview

Run the robot on the new Krabby Uno v0.3 leg boards. Each leg gets its own STM32 board, the Orin talks to the boards over CAN, and the GUI, gamepad, and AI model all control the robot through this new path. The same firmware also runs against the simulated robot, and boards can be reflashed over CAN.

Today three Arduino Megas drive the robot. One Mega connects to the Orin over USB and relays every command and telemetry line for the other two, and the boards elect these roles at boot. The v0.3 hardware removes the need for all of this, since each board has its own CAN connection and a fixed ID set by a switch, but the board has no firmware yet.

The new firmware is small and written from scratch. It does two things: it receives a duty cycle for each of the leg's three motors and drives them, and it sends back each motor's pot reading and hall count. A leg is identified by its CAN ID (1 = FL, 2 = FR, 3 = ML, 4 = MR, 5 = RL, 6 = RR) and a motor by its index (0 = yaw, 1 = hip, 2 = knee). Duty runs from -1.0 to 1.0, where the sign is direction and the magnitude is power, and a duty of 0 holds the joint. A DBC file defines every CAN message, and both the firmware and the Python tools build on that one file. Role election, joint name strings, calibration, position targets, and the hold and version commands are all gone from the firmware.

Task 1 writes the firmware and a new CLI, then controls the simulated robot from that CLI over a virtual CAN bus. Task 2 puts all six boards on the real robot. Task 3 connects the GUI, gamepad path, and AI model image behind an `--stm` option. Task 4 adds flashing over CAN.

At the end, a v0.3 robot walks under GUI, gamepad, or AI model control over CAN, and its firmware updates without a cable.

## Why is this Important?

- The v0.3 boards cannot run a motor until this firmware exists, so the robot cannot move to the new electronics.
- One serial link and one leader board currently carry all 18 joints, so a leader failure takes down the whole robot. On CAN each board acts on its own commands and sends its own telemetry.
- Updating firmware today means a USB connection to each board. Flashing over CAN updates all six boards from the Orin, which fleet updates in later milestones will need.
- Replacing the Mega hardware would otherwise break the Milestone 21 simulation. This milestone keeps firmware testable without a robot, for both engineers and AI agents.

## Tasks

Total: ~15 working days, roughly 3-4 weeks part-time at 20-30 hours per week. Implementation detail and exact code locations live in the per-task files linked below; the acceptance criteria are here. Tasks run in order, and each one depends on the task before it.

### Task 1 - CAN firmware and CLI against the sim → [TASK-1-STM-CAN-SIM.md](TASK-1-STM-CAN-SIM.md)

#### Narrative

Write the firmware and CLI, then use them to move the simulated robot over virtual CAN. Work runs from the bottom of the stack up, and each step gives something to check before the next one starts. This requires 1) a new CLI that flashes `.elf` files, along with a CI step that publishes them, 2) a DBC file that defines the CAN messages, and a CLI command that sends a duty, 3) `can_handler.c`, which receives the duty and drives the motors, 4) `telemetry_handler.c`, which sends pot, hall, and firmware version back, 5) a simulated version of the board's pins, so six copies of these same two files can drive the simulated robot in IsaacSim.

The Mega firmware, its CLI, and its simulation stay as they are, because the existing robots still depend on them.

#### Acceptance Criteria

- 1a - The new CLI flashes a test `.elf` over SWD, and the board blinks its status LED and oscillates motor 0. `update <branch>` downloads the `.elf` that CI published to S3 and flashes it.
- 1b - `krabby-firmware-stm duty 1 0.5 0 0` sends a frame that `candump` shows with ID `0x101` and a yaw duty of 0.5.
- 1c - On a bench board, the same command moves each motor in both directions at the requested speed, and a duty of 0 stops it. The board ignores commands sent to a different leg ID.
- 1d - `krabby-firmware-stm telemetry` shows pot, hall count, duty, and fault updating at 20 Hz for each motor, and firmware version at 1 Hz. When a motor reverses, its hall count reverses too.
- 1e - With six simulated boards on `vcan0`, `krabby-firmware-stm duty 1 1.0 0 0` moves only the FL yaw joint in IsaacSim, and the CLI shows that joint's pot and hall values changing.
- 1f - An integration test performs the check in 1e without a window and passes under `make -C firmware test-firmware-isaac`.
- 1g - `make -C firmware compile-firmware`, `test-native`, and `test-firmware` still pass with no changes to the Mega firmware.

#### Time estimate (~5 days)

| Days | Sub-task title |
|------|----------------|
| 1 | STM32 build, flash CLI, and `.elf` publishing |
| 0.5 | DBC file, `connect()`, and the `duty` command |
| 1 | `can_handler.c` running on a bench board |
| 1 | `telemetry_handler.c` running on a bench board |
| 1.5 | Simulated pins, virtual CAN, and the Isaac test |

### Task 2 - Real robot on two buses → [TASK-2-REAL-ROBOT.md](TASK-2-REAL-ROBOT.md)

#### Narrative

Move all 18 joints on the real robot from the CLI. This requires 1) setting each board's ID switch, flashing it, and wiring three legs to each of the Orin's two CAN ports, 2) a CLI that sends every command on both buses and reads telemetry from both, 3) a `show` command that lists the boards it hears, 4) a sweep script that jogs each joint in turn and checks its telemetry.

#### Acceptance Criteria

- 2a - `show` lists all six boards with their leg name, firmware version, and commit.
- 2b - The sweep script moves each of the 18 joints in both directions and reports a pass for each one, and the correct joint is seen to move each time.
- 2c - After one leg's CAN cable is unplugged, the other five legs still respond and the CLI marks the unplugged leg as stale within 1 second. Plugging the cable back in restores the leg without a restart.
- 2d - The measured hall counts for a full stop-to-stop move of each joint type are recorded in the task report and used in the simulation.

#### Time estimate (~3 days)

| Days | Sub-task title |
|------|----------------|
| 0.5 | Send on both buses, read from both buses, `show` |
| 1 | Orin CAN setup, wiring, and flashing six boards |
| 1 | Sweep script and checking all 18 joints |
| 0.5 | Unplug recovery and hall count measurement |

### Task 3 - GUI, controller, and model image on --stm → [TASK-3-CLIENT-REWIRE.md](TASK-3-CLIENT-REWIRE.md)

#### Narrative

Control the v0.3 robot from the GUI, the gamepad, and the AI model. Each of these reaches the firmware through the serial SDK, so `--stm` swaps in the CAN SDK at the point where they create it, and everything above that point stays the same. This requires 1) an `--stm` option on the GUI, 2) a CAN version of the HAL's motor class that converts radians to duty, 3) an `--stm` option on the HAL server, passed through from `krabby run`, 4) the CAN Python packages in the release image.

Without `--stm`, every tool behaves as it does today, so one image serves both Mega and v0.3 robots.

#### Acceptance Criteria

- 3a - `python -m firmware.gui --stm` jogs joints on the v0.3 robot and shows live pot and hall values. Without the flag, the GUI still controls a Mega robot.
- 3b - With the HAL server running `--stm`, the gamepad walks the v0.3 robot.
- 3c - With the release image running `--stm`, the AI model walks the v0.3 robot.
- 3d - The same release image, run without `--stm`, still controls a Mega robot.

#### Time estimate (~3 days)

| Days | Sub-task title |
|------|----------------|
| 0.5 | GUI `--stm` option |
| 1 | HAL CAN motor class and `--stm` option |
| 0.5 | Release image and `krabby run` changes |
| 1 | Testing the GUI, gamepad, and AI model on the robot |

### Task 4 - Flash over CAN → [TASK-4-CAN-FLASH.md](TASK-4-CAN-FLASH.md)

#### Narrative

Update firmware over CAN so routine updates need no cable. This requires 1) the OpenBLT bootloader on each board, installed once over SWD, 2) a check in `can_handler.c` that resets the board into the bootloader when an XCP connect frame arrives, 3) an `update` command that sends the new firmware over CAN to one leg or to all six.

#### Acceptance Criteria

- 4a - A board with OpenBLT installed boots the application normally, and the application behaves as it did before the bootloader was added.
- 4b - `update --leg 1 <branch>` flashes that leg over CAN from a build in S3, and the board then reports the new version. Other legs keep running.
- 4c - `update --all <branch>` flashes all six boards across both buses, and `show` lists the new version on all six.
- 4d - If power is cut during a flash, the board stays in the bootloader on the next power-up, and running `update` again completes the flash.

#### Time estimate (~4 days)

| Days | Sub-task title |
|------|----------------|
| 1.5 | OpenBLT port for this board |
| 0.5 | Moving the application above the bootloader, and the reset on XCP connect |
| 1 | `update` over CAN for one leg and all legs |
| 1 | Power-cut testing and the six-board update |

## Information

- Repositories: `krabby-research` (firmware, HAL, simulation, images), `patina-foundation-grants` (this grant), `krabby-contracts` (the milestone ICA, M26 folder not yet created).
- Hardware: `hardware/Uno-v0.3/` holds the schematic, BOM, assembly PDF, and EasyEDA project. Where the Milestone 20 documents disagree with these files, these files are correct. The board uses an STM32G474RBT3, three DRV8262 motor drivers, a DAC7574 that sets each motor's current limit, and an MCP2562FD CAN transceiver. Datasheet: https://www.ti.com/product/DRV8262.
- Mega firmware that this replaces: `firmware/arduino/arduino.ino`, `actuator_manager.h`, and `hall_hw.cpp`.
- Host tools that this replaces: `firmware/cli.py`, `firmware/krabby_mcu.py`, and `firmware/mcu_port.py`.
- Simulation: `firmware/sim/`. Start with `ARCHITECTURE.md`, `robot_core.py`, and `simulated_actuator.py`.
- Code that creates the SDK: `firmware/gui/app.py`, `hal/server/jetson/krabby_mcusdk.py`, and `hal/server/jetson/hal_server.py`.
- Publishing: `firmware/scripts/publish_firmware.py`, `firmware/manifest.py`, `.github/workflows/publish-firmware.yml`, and the `krabby-firmware-public` S3 bucket.
- Bootloader: OpenBLT, https://www.feaser.com/openblt/. Its Nucleo-G474RE demo is the starting point for Task 4.
- Not part of this milestone: the Qwiic OLED, battery monitoring, motor current telemetry, calibration, and position control.

## FAQ

1. Why write the firmware from scratch instead of porting the Mega code? Most of the Mega code exists for features this firmware drops: the serial parser, role election, telemetry relay, calibration, and the position controller. Only the rule for driving a motor carries over, and that is a few lines.

2. Can the robot walk with duty commands alone? Yes, because it already does. The HAL's position command path is commented out in `hal/server/jetson/krabby_mcusdk.py`, since not every joint has a pot connected, and production sends open-loop jog PWM. Duty is that same command on a -1.0 to 1.0 scale. Position control can return on the Orin once calibration moves there.

3. What happens if the Orin stops sending commands? Each motor keeps running at its last duty, the same way a jog does on the Mega today. The firmware has no command timeout. Its watchdog only covers a firmware lockup, which resets the board and puts the motors to sleep.

4. The DBC is meant to generate code for both sides, so why is there no generated Python? `cantools` generates C only. The firmware compiles that generated C, and the Python tools load `krabby.dbc` with `cantools` when they start. Both sides still read the message layout from the one file.

5. Why does each leg have its own CAN IDs? A CAN bus requires that no two boards send the same ID, so telemetry needs per-leg IDs in any case. Per-leg command IDs let the CAN hardware discard other legs' commands before the firmware sees them.

6. Why classic CAN when the hardware supports CAN FD? Every message fits in the classic 8 bytes, and classic CAN at 1 Mbit/s leaves plenty of room for this traffic. It also works with every CAN tool and adapter. Moving to FD later only requires a change to the bus settings and the DBC.

7. Why send every command on both buses? The CLI then needs no record of which leg is on which bus, and each board already ignores commands for other legs. Wiring a leg to the other bus requires no software change.

8. Why do hall counts differ from the Mega's? The Mega counts both edges of hall A and uses hall B only for direction. The STM32 timers count both edges of both signals, so the same motion produces twice as many counts. The two numbers should never be compared directly.

9. Why does this grant disagree with the Milestone 20 documents? The board changed during design. It uses DRV8262 drivers where those documents name the STSPIN9P21, an STM32G474RB with 128 KB of flash where they name the 512 KB RE, FDCAN2 where they name FDCAN1, JST-PH CAN connectors where they name JST-GH, and a node ID switch where they name solder jumpers. This milestone also splits the single six-leg bus into two buses of three legs.

10. Do the firmware and bootloader fit in 128 KB? They should, since OpenBLT usually takes under 32 KB and this application is small. The linker script enforces the limit, so a build that outgrows its space fails instead of overwriting the bootloader.

11. Can this be developed on Windows? Partly. The unit tests use python-can's in-process virtual bus and run on Windows. `vcan0` and SocketCAN exist only on Linux, so the IsaacSim test and all work on the real bus need a Linux machine.

12. Does the Isaac `--backend fullsim` HAL use the new firmware? No. It keeps running the Mega firmware over serial. This milestone tests the new firmware in simulation through the CLI, and the AI model path is tested on the real robot in Task 3.
