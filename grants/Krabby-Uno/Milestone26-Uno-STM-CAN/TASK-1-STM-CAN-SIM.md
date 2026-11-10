# Task 1 — CAN firmware and CLI against the sim

Goal: Write the STM32 firmware and a new CLI, and confirm them by flashing a bench board, moving its motors, and moving the simulated robot over virtual CAN.

Outputs
- `firmware/stm/board/`: the STM32 build and the code that touches real pins.
- `firmware/stm/core/`: `can_handler.c` and `telemetry_handler.c`.
- `firmware/stm/dbc/krabby.dbc` and the C code generated from it.
- `firmware/stm/cli/`: the `krabby-firmware-stm` command and the CAN SDK.
- `firmware/stm/sim/` and `firmware/sim/can_robot_core.py`: the simulated pins and the Isaac loop.
- CI publishing of `krabby_uno.elf`.
- Unit tests for the CAN messages and handlers, and one Isaac integration test.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-1---can-firmware-and-cli-against-the-sim).)

---

Note: line numbers below point at the right function at the time of writing and may drift.

---

## 1. Code layout

The handlers in `core/` never touch hardware directly. They call the functions in `core/board.h`, which are implemented once for the real board in `board/` and once for the simulator in `sim/`. This is how the same two C files run in both places.

```c
/* core/board.h */
typedef struct leg leg_t;   /* node ID and per-board state; sim creates six */

void     send_pwm(leg_t *leg, int motor, bool direction, int power);  /* power 0..1000 */
uint16_t read_pot(leg_t *leg, int motor);
int32_t  read_hall(leg_t *leg, int motor);
bool     read_fault(leg_t *leg, int motor);
void     clear_fault(leg_t *leg, int motor);
bool     can_send(leg_t *leg, uint32_t id, const uint8_t *data, uint8_t len);
```

The handlers take a `leg_t *` and keep no global state, so one simulator process can run six boards. The real board's main loop is short: refresh the watchdog, pass any received frame to `can_handler_on_frame()`, and call `telemetry_handler_tick(now_ms)`.

Do not edit `firmware/cli.py`, `firmware/krabby_mcu.py`, or `firmware/arduino/`. Add `firmware.stm`, `firmware.stm.cli`, the `krabby-firmware-stm` script, and an `stm` optional dependency group (`python-can`, `cantools`) to `firmware/pyproject.toml`.

## 2. Flashing and publishing

Copy `firmware/cli.py` to `firmware/stm/cli/cli.py`. Keep `_fetch_index`, `_fetch_builds`, `_page`, `cmd_update`'s branch selection, and the download cache. Delete the Mega port scan and `_probe_version`. Replace `_flash()` (~L250) with this command, and print an install hint if `STM32_Programmer_CLI` is missing, as the current code does for `avrdude`.

```
STM32_Programmer_CLI -c port=SWD -w krabby_uno.elf -v -rst
```

The new CLI has a `flash <path>` command for a local build and an `update <branch>` command for a published one. Flash from a development machine with an ST-LINK on the board's 5-pin SWD header. Flashing from the Orin arrives with Task 4.

Build with CMake, `arm-none-eabi-gcc`, and ST's STM32CubeG4 drivers at a pinned version, through a new `make -C firmware stm-firmware` target. The output is `firmware/build/stm/krabby_uno.elf`. The chip is the STM32G474RBT3, so set the linker script to 128 KB of flash. Run the CPU at 170 MHz from the 8 MHz crystal. The first test firmware blinks `STATUS_LED` and slowly oscillates motor 0 at low duty.

To publish, add an optional `elf_url` next to `hex_url` in the manifest, `latest.json`, `builds.json`, and `index.json` entries that `publish_firmware.py` writes, and upload the file to `<branch>/<build_key>/krabby_uno.elf`. Deployed Mega clients ignore the new key. In `firmware/manifest.py`, read `elf_url` with `.get()` so older builds parse, and have `update` exit with a clear error when a build has no `.elf`. In `publish-firmware.yml`, install `gcc-arm-none-eabi` and run `make -C firmware stm-firmware` before the publish step.

Have `gen_version_h.py` also write `firmware/stm/core/version.h`, adding major, minor, and patch as numbers and the commit as a `uint32_t`, for the BoardInfo message.

## 3. CAN messages

Use classic CAN at 1 Mbit/s with 11-bit IDs. Command IDs are lower than telemetry IDs, so commands win when both try to send at once.

| Message | CAN ID | Sent by | Contents |
|---|---|---|---|
| SetDuty | `0x100 + leg` | Host | yaw, hip, knee: int16 each, scale 0.001, range -1.0 to 1.0 |
| JointTelemetry | `0x200 + leg*16 + motor` | Board | hall: int32, duty: int16 scale 0.001, pot: 12 bits, fault: 1 bit |
| BoardInfo | `0x300 + leg` | Board | major, minor, patch: uint8 each, commit: uint32 |

A DBC message has one fixed ID, so `krabby.dbc` lists 30 messages: 6 SetDuty, 18 JointTelemetry, and 6 BoardInfo. Name them by leg and joint (`SET_DUTY_FL`, `JOINT_FL_YAW`) so `candump` output is readable.

Generate the C code into `firmware/stm/dbc/generated/` with `python -m cantools generate_c_source`, and commit it so the firmware build needs no Python. A `make -C firmware stm-dbc` target regenerates it. All legs share the same layouts, so the firmware decodes every SetDuty with one generated function.

The SDK in `firmware/stm/cli/krabby_mcu.py` loads the DBC with `cantools` and sends with `python-can`. `connect()` opens the bus named by `KRABBY_CAN_BUSES` (default `can0`). `set_duty(leg, yaw, hip, knee)` encodes and sends one frame. A background thread decodes received telemetry into `joints[(leg, motor)]`, with the time each value arrived. The CLI's `duty` and `telemetry` commands call this SDK.

## 4. Driving the motors

These pins come from the EasyEDA project. Motors 0, 1, and 2 are `M1`, `M2`, and `M3` on the schematic.

| Signal | Motor 0 (yaw) | Motor 1 (hip) | Motor 2 (knee) | Peripheral |
|---|---|---|---|---|
| `M*_PWM` | PA8 | PA9 | PA10 | TIM1 CH1, CH2, CH3 |
| `M*_DIR` | PB0 | PB1 | PC1 | GPIO output |
| `M*_SLEEP` | PB10 | PB15 | PB14 | GPIO output |
| `M*_FAULT` | PC14 | PC15 | PB11 | GPIO input |
| `M*_POT_ADC` | PA7 | PC4 | PC5 | ADC2 IN4, IN5, IN11 |
| `M*_POS_A` | PA5 | PC7 | PB2 | TIM2 CH1, TIM3 CH2, TIM5 CH1 |
| `M*_POS_B` | PB3 | PC6 | PC12 | TIM2 CH2, TIM3 CH1, TIM5 CH2 |

The remaining signals are FDCAN2 on PB12 (RX) and PB13 (TX), I2C1 on PA15 (SCL) and PB7 (SDA), the node ID on PC13, PB4, and PB5, `STATUS_LED` on PC8, and `CAN_ACTIVITY_LED` on PC9. The transceiver's standby pin is wired to ground, so firmware does not control it.

Each DRV8262 is strapped as a single H-bridge in PH/EN mode: `M*_PWM` goes to EN and `M*_DIR` goes to PH. `can_handler_on_frame()` ignores any ID other than this board's and calls `send_pwm()` once per motor. A negative duty clears the direction flag, and the magnitude becomes `power` from 0 to 1000. Set TIM1 to 20 kHz, and remember that TIM1 needs its main output enable bit set before any PWM appears. A power of 0 holds EN low, which brakes the motor. The lead screw cannot be back-driven, so the joint holds its position whether the bridge is braked or asleep.

Set the direction polarity per motor in one constant in `board/`, so that positive duty extends the actuator, matching `driveActuator()` in `firmware/arduino/actuator_manager.h` (~L320). Check each motor once on the bench.

Boot in this order, because the motors cannot move until the DAC is programmed:
1. Leave `M*_SLEEP` low. The board's pull-down resistors hold it low through reset.
2. Read the node ID. The switch pulls a line to ground when closed, so a closed switch reads as a 1 bit, with `NODE_ID0` as bit 0. If the ID is not 1 to 6, blink `STATUS_LED` rapidly and keep the motors asleep.
3. Program the DAC7574 over I2C1 at address `0x4C` (both address pins are grounded). Channels A, B, and C set the current limit for motors 0, 1, and 2 through `M*_VREF`. The DAC starts at 0 V, which means no drive current. Use the DRV8262 datasheet's current formula with the board's 1.43 kΩ `IPROPI` resistor. Start at a 2 A limit, keep it in one constant, and record the final value in the task report.
4. Enable the independent watchdog at about 100 ms, so a firmware lockup resets the board and the pull-downs put the motors to sleep.
5. Set `M*_SLEEP` high.

Configure the FDCAN filter to accept only `0x100 + node_id`. In the main loop, check for the bus-off state and clear `INIT` when it occurs, because otherwise a leg that loses its CAN cable never rejoins the bus. Blink `CAN_ACTIVITY_LED` on traffic. A board needs another CAN device on the bus to acknowledge its frames, so bench work needs the Orin or a USB CAN adapter connected, with a 120 Ω terminator at each end of the cable.

The DRV8262 reports a fault on `M*_FAULT`, which is active low. When a motor with a fault receives a duty of 0, call `clear_fault()`, which pulses that motor's `M*_SLEEP` low. Check the pulse width in the datasheet.

## 5. Telemetry

`telemetry_handler_tick()` sends all three JointTelemetry frames every 50 ms and BoardInfo every 1000 ms, offset by 25 ms from a joint telemetry tick. The G474's FDCAN has only 3 transmit slots, so sending a fourth frame in the same tick would fail. If `can_send()` returns false, drop the frame, because the next tick sends fresher data.

Read the pots on ADC2 with hardware oversampling. The board divides the 5 V pot signal with a 1 kΩ + 15 kΩ series resistance over 22 kΩ, so a pot at full travel reads about 3590 of 4095. Send the raw value.

Each hall pair feeds a timer in encoder mode, counting both edges of both signals, with a small input filter enabled. TIM2 and TIM5 are 32-bit. TIM3 is 16-bit, so on each tick add the signed change in its counter to a 32-bit total. On motor 1, hall A is on channel 2 and hall B on channel 1, the reverse of the other two motors, so invert its count to give every motor the same sign. Check on the bench that extending increases the count.

## 6. Simulation

The Mega simulation records Arduino pin writes in a host library and drives Isaac joints from them. This task follows the plan in `firmware/sim/ARCHITECTURE.md` (~L30): replace the pin layer with bus IDs and keep the same loop shape, with no new framework. Leave the existing `RobotCore` and its host library in place for the Mega.

`firmware/stm/sim/board_sim.c` implements `board.h` using plain arrays. `send_pwm()` stores the signed duty, `read_pot()` and `read_hall()` return values set from Python, and `can_send()` adds the frame to a list. Build it with `can_handler.c` and `telemetry_handler.c` into a `krabby_stm_host` shared library through a `make -C firmware host-stm` target. Load it with a small `ctypes` bridge, `firmware/sim/can_board_bridge.py`, in the style of `host_bridge.py`.

Add `CanRobotCore` in `firmware/sim/can_robot_core.py`, based on `RobotCore.step()` (`robot_core.py` ~L151). It creates six boards with IDs 1 to 6 and finds each motor's joint with `isaac_joint_name_for_firmware_id()` in `simulated_robot_config.py`. Each step does the following:
1. Read all waiting frames from the CAN bus and give them to the boards.
2. Read each joint's position with `PlantReadbackView` and set that motor's pot and hall values.
3. Advance each board's clock by `physics_step_ms(env)`, run its telemetry tick, and send its queued frames onto the bus.
4. Convert each motor's duty to a joint velocity (`duty * max_joint_speed`) and apply it with `set_joint_velocity_target`.
5. Call `advance_sim(env)`, then `apply_cam_hip_slave`, both as `RobotCore.step()` does. `env.step()` would overwrite the velocity targets.

Set the pot with `joint_angle_to_pot_adc()` in `pot_position.py`, passing a `PotCalibrationBand(0, 3590)` so it matches the real board's range. Do not reuse the Mega hall simulator, because `hall_sim.cpp` only counts upward and the new telemetry is signed. Compute the count directly as `round(travel_fraction * counts_per_full_travel)`. Add `stm_counts_per_full_travel` to `hall_encoder_scale.json`, starting at 6400, which is twice the current placeholder because of the change described in FAQ 8. Task 2 replaces it with a measured value.

A new `python -m firmware.sim.run_can_sim --bus vcan0` command opens the Isaac scene and runs this loop. Create the bus once on the Linux machine:

```
sudo modprobe vcan
sudo ip link add dev vcan0 type vcan
sudo ip link set up vcan0
```

Then, in a second terminal with `KRABBY_CAN_BUSES=vcan0`, the CLI talks to the simulated robot with no simulator-specific code.

## 7. Tests

`tests/unit/firmware/test_stm_can_messages.py` encodes SetDuty through the DBC and checks the exact bytes against values worked out by hand. It decodes telemetry frames containing extreme values: pot 0 and 4095, negative duty, and a large negative hall count.

`tests/unit/firmware/test_stm_handlers.py` runs the host library through the `ctypes` bridge and needs no CAN interface. It checks that a SetDuty for this board sets the three duties, a SetDuty for another leg changes nothing, a duty of -0.5 gives direction false and power 500, and a pot value set from Python appears in the next telemetry frame. It also checks that telemetry goes out every 50 ms and BoardInfo every 1000 ms.

`tests/unit/firmware/test_stm_sdk.py` connects the SDK to the host library over python-can's `virtual` bus, which also works on Windows, and checks that `set_duty()` reaches a board and telemetry returns to `joints`.

`tests/integration/firmware/test_isaac_stm_can.py` runs in IsaacSim without a window, next to the other `test_isaac_*` files. It sends `set_duty(1, 1.0, 0, 0)` and checks that FL yaw moves in the positive direction while the other 17 joints stay still, and that its pot and hall values increase. It then sends `set_duty(1, -1.0, 0, 0)` and checks that all of these reverse, and sends zeros and checks that the joint stops.
