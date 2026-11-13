# Task 1 — CAN firmware and CLI against the sim

Goal: Write the STM32 firmware that will ship on the v0.3 boards, with a real and a virtual side for each of its two interfaces, and confirm it by moving the simulated robot from the new CLI. No hardware is needed.

- The CAN interface sits between the Python SDK and a board. The real side is `can0` and `can1` to the STM32 boards. The virtual side is `vcan0` to six simulated boards. The frames are the same on both, and one SDK option picks which side to use.
- The pin interface sits between the firmware core, which turns CAN frames into pin controls, and the hardware. It is one C class, `board_pins_t`, with two implementations: `board_pins_stm32.c` calls the STM32 `HAL_*` functions, and `board_pins_isaac.c` drives the robot in IsaacSim.

Denny's firmware in `hardware/PCB-v0.3/Firmware/` is example code. `board_pins_stm32.c` copies its `HAL_*` calls from it.

Outputs
- `firmware/stm/core/`: `board_pins.h` (the pin control class), `board_can.h`, `app.c`, `can_handler.c`, `telemetry_handler.c`, and `calibration.c`.
- `firmware/stm/board/`: `board_pins_stm32.c`, `can_stm32.c`, `main.c`, and the STM32 build.
- `firmware/stm/sim/`: `board_pins_isaac.c` and `can_sim.c`.
- `firmware/stm/dbc/krabby.dbc` and the C code generated from it.
- `firmware/stm/cli/`: the `krabby-firmware-stm` command with `duty`, `calibrate`, `telemetry`, and a `--virtual` option, and the CAN SDK.
- `firmware/sim/can_robot_core.py` and `run_can_sim`: the Isaac loop that runs six virtual boards.
- Unit tests for the CAN messages and handlers, and one Isaac integration test.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-1---can-firmware-and-cli-against-the-sim).)

---

Note: line numbers below point at the right function at the time of writing and may drift.

---

## 1. The CAN interface

The CAN interface carries the messages in section 4 between the Python SDK and each board. It has two sides, and nothing above or below it knows which one is in use:

| | Real | Virtual |
|---|---|---|
| SDK option | default | `virtual=True`, or `--virtual` on the CLI |
| Bus | `can0` and `can1` | `vcan0` |
| Board at the other end | an STM32 running `board/main.c` | a simulated board in `krabby_stm_host`, run by `run_can_sim` |
| Board end of the bus | `board/can_stm32.c`: FDCAN2, set up as in Denny's `App/Src/can_bridge.c` | `sim/can_sim.c`: a send queue and a receive queue that `CanRobotCore` moves to and from `vcan0` |

The firmware core reaches the bus through three functions in `core/board_can.h`, which both board ends implement:

```c
/* core/board_can.h */
void can_init(leg_t *leg);
bool can_send(leg_t *leg, uint32_t id, const uint8_t *data, uint8_t len);
bool can_receive(leg_t *leg, uint32_t *id, uint8_t *data, uint8_t *len);  /* false if none waiting */
```

On the real board, `can_receive()` takes frames from a queue that the FDCAN2 RX FIFO interrupt fills, as Denny's `HAL_FDCAN_RxFifo0Callback()` does.

## 2. The pin interface

The firmware core in `core/` (`can_handler.c`, `telemetry_handler.c`, `calibration.c`) turns CAN frames into pin controls, and pin readings into telemetry. It never touches hardware directly. All pin control goes through one C class, `board_pins_t`, defined in `core/board_pins.h`. It is a struct of function pointers with two implementations:

- `board/board_pins_stm32.c` controls the real STM32 pins.
- `sim/board_pins_isaac.c` controls the simulated robot in IsaacSim. The simulator creates six instances, one per leg, and all six attach to the same IsaacSim instance and the same robot. Each instance reads and drives only its own leg's three joints.

```c
/* core/board_pins.h */
typedef struct { uint16_t pot_retracted, pot_extended; int32_t hall_retracted, hall_extended; } joint_cal_t;

/* joint is 1 (yaw), 2 (hip), or 3 (knee) */
typedef struct board_pins board_pins_t;
struct board_pins {
    void     (*init)(board_pins_t *self);
    uint8_t  (*node_id)(board_pins_t *self);
    uint32_t (*now_ms)(board_pins_t *self);
    void     (*send_pwm)(board_pins_t *self, int joint, bool direction, int power);  /* power 0..1000 */
    uint16_t (*read_pot)(board_pins_t *self, int joint);
    int32_t  (*read_hall)(board_pins_t *self, int joint);
    bool     (*read_fault)(board_pins_t *self, int joint);
    void     (*clear_fault)(board_pins_t *self, int joint);
    bool     (*save_calibration)(board_pins_t *self, int joint, const joint_cal_t *cal);
    bool     (*load_calibration)(board_pins_t *self, int joint, joint_cal_t *cal);  /* false if none saved */
};

board_pins_t *board_pins_stm32(void);                  /* the one real board */
board_pins_t *board_pins_isaac_create(uint8_t node_id); /* one per simulated board */
```

The firmware core only ever holds a `board_pins_t *` and calls through it, for example `pins->send_pwm(pins, 2, true, 500)`. It cannot tell which implementation it has.

Each STM32 method copies the `HAL_*` calls Denny's example uses for the same job (`App/Src/app_service.c` unless noted). Each Isaac method stores or returns a value for its own leg, and `CanRobotCore` (section 8) exchanges all six boards' values with the one IsaacSim env each step.

| Method | `board_pins_stm32.c`, from Denny's example | `board_pins_isaac.c` |
|---|---|---|
| `init` | start the TIM1 PWM and encoder timers, set the DAC7574 current limit over I2C1 (`dac_setup()`) | clear the board's state |
| `node_id` | the ID switch (`Board_GetNodeId()` in `App/Src/board.c`) | the ID given to `board_pins_isaac_create()`, 1 to 6 |
| `now_ms` | `HAL_GetTick()` | a clock advanced by `physics_step_ms(env)` |
| `send_pwm` | the motor's DIR and SLEEP pins and TIM1 channel, and its left or right LED (`set_applied()`) | store the signed duty, which becomes the Isaac joint's velocity target |
| `read_pot` | the filtered ADC1 and ADC2 readings | the pot value set from the Isaac joint's position |
| `read_hall` | the motor's encoder timer (TIM2, TIM8, TIM5), extended to a signed 32-bit total | the hall count set from the Isaac joint's position |
| `read_fault`, `clear_fault` | the motor's FAULT pin, and a sleep toggle to clear it | a fault flag that tests can set |
| `save_calibration`, `load_calibration` | a reserved flash page, written the way Denny's firmware saves its configuration (the STM32G474 has no EEPROM) | an array that survives a simulated board restart |

## 3. Main loop and builds

`core/app.c` holds the main loop that both builds share. `app_init(leg, pins)` stores the `board_pins_t *` in the leg, calls its `init` and `can_init()`, and loads each joint's calibration. `app_tick(leg)` passes every waiting frame from `can_receive()` to `can_handler_on_frame()`, then runs `calibration_tick()` and `telemetry_handler_tick()`. The firmware takes a `leg_t *` and keeps no global state, so one simulator process can run six boards, each with its own `board_pins_isaac_create()` instance.

The same `core/` files build two ways. This matches the Mega simulation, where the shipping firmware runs unchanged against a simulated `Arduino.h` (`firmware/sim/ARCHITECTURE.md`).
- `make -C firmware stm-build` builds the real board firmware with `board/board_pins_stm32.c` and `board/can_stm32.c`, using `arm-none-eabi-gcc` for the STM32G474. Start from Denny's CubeMX project for the clock, pin, and peripheral setup, and replace his `App/` folder with a `board/main.c` that calls `app_init(leg, board_pins_stm32())` once and `app_tick()` in a loop. Leave OpenBLT and the flash layout as Denny set them, so the result flashes over CAN with his tools. Running this build on a real board happens in the support weeks.
- `make -C firmware host-stm` builds the `krabby_stm_host` shared library with `sim/board_pins_isaac.c` and `sim/can_sim.c`, for the simulator and the unit tests.

Do not edit `firmware/cli.py`, `firmware/krabby_mcu.py`, or `firmware/arduino/`. Add `firmware.stm`, `firmware.stm.cli`, the `krabby-firmware-stm` script, and an `stm` optional dependency group (`python-can`, `cantools`) to `firmware/pyproject.toml`.

## 4. CAN messages

Use CAN FD with 11-bit IDs, at 1 Mbit/s for arbitration and 5 Mbit/s for data, with bit rate switching on. Every message is an FD frame. Command IDs are lower than telemetry IDs, so commands win when both try to send at once.

| Message | CAN ID | Sent by | Length | Contents |
|---|---|---|---|---|
| SetDuty | `0x100` | Host | 36 bytes | for each of legs 1 to 6, joints 1, 2, 3: int16, scale 0.001, range -1.0 to 1.0 |
| Calibrate | `0x180 + leg` | Host | 1 byte | joint: uint8, 2 or 3 |
| LegTelemetry | `0x200 + leg` | Board | 48 bytes | for each of joints 1, 2, 3: hall int32, duty int16 scale 0.001, pot 12 bits, fault 1 bit, calibrating 1 bit, calibrated 1 bit, position float64 |
| BoardInfo | `0x300 + leg` | Board | 8 bytes | major, minor, patch: uint8 each, commit: uint32 |

Each joint's telemetry takes 16 bytes with padding, so one FD frame carries the whole leg. One SetDuty frame commands the whole robot, so all 18 joints update together and each board reads its own leg's 6 bytes. Telemetry and BoardInfo need an ID per leg, because no two boards may send the same ID, and a DBC message has one fixed ID. `krabby.dbc` therefore lists 19 messages: 1 SetDuty, 6 Calibrate, 6 LegTelemetry, and 6 BoardInfo. Mark each as an FD frame with the `VFrameFormat` attribute. Name them by leg and joint ID only (`SET_DUTY` with signals `L1_J1` to `L6_J3`, `TELEMETRY_1` with signal `J1_HALL`). Names like "front left yaw" belong in the GUI, not in the firmware or the DBC.

Generate the C code into `firmware/stm/dbc/generated/` with `python -m cantools generate_c_source`, and commit it so the firmware build needs no Python. A `make -C firmware stm-dbc` target regenerates it. All legs share the same telemetry layout, so one generated function encodes every leg's telemetry.

Have `gen_version_h.py` also write `firmware/stm/core/version.h`, adding major, minor, and patch as numbers and the commit as a `uint32_t`, for the BoardInfo message.

The SDK in `firmware/stm/cli/krabby_mcu.py` loads the DBC with `cantools` and sends with `python-can`. `connect()` opens `can0` and `can1` with `fd=True`. `connect(virtual=True)` opens `vcan0` instead, where the simulated boards run, and nothing else in the SDK changes. Tests can pass in their own python-can bus. The SDK keeps the last duty for all 18 joints. `set_duties(values)` replaces all 18 and `set_duty(leg, j1, j2, j3)` replaces one leg's three, and each sends one SetDuty frame with `is_fd=True` and `bitrate_switch=True`. `send_calibration(leg, joint)` sends one Calibrate frame. A background thread decodes received telemetry into `joints[(leg, joint)]`, with the time each value arrived. The CLI in `firmware/stm/cli/cli.py` has `duty`, `calibrate <leg> <joint>`, and `telemetry` commands that call this SDK, and a `--virtual` option that passes `virtual=True` to `connect()`.

## 5. Driving the joints

`can_handler_on_frame()` reads this board's three values from SetDuty and calls `send_pwm()` once per joint. It ignores any other ID except its own Calibrate ID. A negative duty clears the direction flag, and the magnitude becomes `power` from 0 to 1000. A power of 0 holds the joint. Positive duty extends the actuator.

When a joint with a fault receives a duty of 0, call `clear_fault()`.

## 6. Calibration

This ports the Mega's auto-calibration (`updateCalibration()` in `firmware/arduino/actuator_manager.h` ~L473) to one joint at a time. When `can_handler_on_frame()` receives a Calibrate frame for this board, it starts a small state machine in `calibration.c` for that joint. `calibration_tick(leg, now_ms)` runs it from the main loop, so the board keeps sending telemetry while it runs.

1. Drive the joint to extend at a duty of 0.6 until the hall count has not changed for 250 ms, which is the Mega's stall rule (`isStalled()` ~L239). Record the pot reading and hall count.
2. Drive it to retract at a duty of -0.6 until it stalls the same way. Record the pot reading and hall count.
3. Stop the joint and save the four values with `save_calibration()`.

Keep the duty and stall time in constants. Stop and keep the old calibration if a joint has not stalled after 10 seconds, if the driver reports a fault, or if a SetDuty gives the joint being calibrated a non-zero duty. Until then the board ignores SetDuty values for that joint. Telemetry sets the joint's `calibrating` bit while this runs. Yaw (joint 1) rotates freely and has no stops, so the board ignores a Calibrate frame for joint 1, and yaw's `position` stays NaN.

At startup `app_init()` loads each joint's calibration with `load_calibration()`.

## 7. Telemetry

`telemetry_handler_tick(now_ms)` sends LegTelemetry every 50 ms and BoardInfo every 1000 ms. If `can_send()` returns false, drop the frame, because the next tick sends fresher data.

Telemetry always carries the raw values. The pot value is the raw 12-bit reading, and a pot at full travel reads about 3590 of 4095 on the real board. The hall count is the signed running total of hall ticks the board has counted since it started, and it goes down when the motor reverses.

`position` runs from 0.0 at the retracted stop to 1.0 at the extended stop, clamped to that range. It comes from the pot when the calibrated pot range is at least 100 counts, and from the hall count otherwise, since not every joint has a pot connected. The hall total restarts at 0 on every power-up, so a hall-only joint needs a new calibration after a restart. `position` is NaN and the `calibrated` bit is clear until the joint has a usable calibration.

## 8. Simulation

The Mega simulation has the same two virtual sides: a host `Arduino.h` that records pin writes and drives Isaac joints from them, and a virtual serial port the SDK talks to (`firmware/sim/ARCHITECTURE.md`, `VIRTUAL_SERIAL.md`). The STM simulation swaps these for `board_pins_isaac.c` and `vcan0`, and keeps the same loop shape with no new framework. Leave the existing `RobotCore` and its host library in place for the Mega.

Sections 1 and 2 describe `can_sim.c` and `board_pins_isaac.c`. Load the `krabby_stm_host` library with a small `ctypes` bridge, `firmware/sim/can_board_bridge.py`, in the style of `host_bridge.py`. Because `sim/can_sim.c` only fills and empties queues, the same library works on `vcan0` and on python-can's in-process virtual bus.

Add `CanRobotCore` in `firmware/sim/can_robot_core.py`, based on `RobotCore.step()` (`robot_core.py` ~L151). It runs in one process with one IsaacSim env. It creates six boards with IDs 1 to 6, each with its own `board_pins_isaac_create()` instance and its own CAN queues, and maps each leg and joint ID to its Isaac joint in one table, built with `isaac_joint_name_for_firmware_id()` in `simulated_robot_config.py`. Each step does the following:
1. Read all waiting frames from the CAN bus and add them to each board's receive queue.
2. Read each joint's position with `PlantReadbackView` and set that joint's pot and hall values.
3. Advance each board's clock by `physics_step_ms(env)`, call `app_tick()`, and send the frames in its send queue onto the bus.
4. Convert each joint's duty to a joint velocity (`duty * max_joint_speed`) and apply it with `set_joint_velocity_target`.
5. Call `advance_sim(env)`, then `apply_cam_hip_slave`, both as `RobotCore.step()` does. `env.step()` would overwrite the velocity targets.

Set the pot with `joint_angle_to_pot_adc()` in `pot_position.py`, passing a `PotCalibrationBand(0, 3590)` so it matches the real board's range. Do not reuse the Mega hall simulator, because `hall_sim.cpp` only counts upward and the new telemetry is signed. Compute the count directly as `round(travel_fraction * counts_per_full_travel)`. Add `stm_counts_per_full_travel` to `hall_encoder_scale.json`, starting at 6400, which is twice the current placeholder because of the change described in FAQ 8. Replace it with Denny's measured value from the real boards when it is available.

A new `python -m firmware.sim.run_can_sim --bus vcan0` command opens the Isaac scene and runs this loop. Create the bus once on the Linux machine. The MTU of 72 lets `vcan0` carry FD frames:

```
sudo modprobe vcan
sudo ip link add dev vcan0 type vcan
sudo ip link set vcan0 mtu 72
sudo ip link set up vcan0
```

Then, in a second terminal, `krabby-firmware-stm --virtual` talks to the simulated robot. The CLI and SDK have no other simulator-specific code.

Each Isaac joint's position limits act as its mechanical stops. A joint driven into a limit stops moving, so its hall count stops changing and calibration detects the stall the same way it does on the real board.

## 9. Tests

`tests/unit/firmware/test_stm_can_messages.py` encodes SetDuty through the DBC and checks the exact bytes against values worked out by hand. It decodes telemetry frames containing extreme values: pot 0 and 4095, negative duty, a large negative hall count, and a NaN position.

`tests/unit/firmware/test_stm_handlers.py` runs the host library through the `ctypes` bridge and needs no CAN interface. It checks that SetDuty sets this board's three duties from its own leg's values, that changing only other legs' values changes nothing on this board, a duty of -0.5 gives direction false and power 500, and a pot value set from Python appears in the next telemetry frame. It also checks that LegTelemetry goes out every 50 ms and BoardInfo every 1000 ms, and that a frame `can_send()` refuses is dropped. For calibration, it drives a joint whose pot and hall values Python holds still at each end, and checks that the joint extends, then retracts, saves the four values, and reports position 0.0 and 1.0 at the two ends. It also checks that a non-zero SetDuty for the joint being calibrated stops it and keeps the old values, that a joint that never stalls stops after 10 seconds, that a joint with a flat pot reports position from the hall count, and that a Calibrate frame for joint 1 changes nothing.

`make -C firmware stm-build` compiles `board_pins_stm32.c` and `can_stm32.c` with the same `core/` files, so a change to `core/` that breaks the board build is caught without hardware.

`tests/unit/firmware/test_stm_sdk.py` connects the SDK to the host library over python-can's `virtual` bus, which also works on Windows, and checks that `set_duty()` reaches a board, that a second `set_duty()` for another leg keeps the first leg's values in the next frame, and that telemetry returns to `joints`.

`tests/integration/firmware/test_isaac_stm_can.py` runs in IsaacSim without a window, next to the other `test_isaac_*` files. It sends `set_duty(1, 1.0, 0, 0)` and checks that leg 1 joint 1 moves in the positive direction while the other 17 joints stay still, and that its pot and hall values increase. It then sends `set_duty(1, -1.0, 0, 0)` and checks that all of these reverse, and sends zeros and checks that the joint stops. Finally it sends `send_calibration(1, 2)`, waits for the `calibrating` bit to clear, and checks that leg 1's hip reports position 0.0 at its retracted limit. It drives the hip partway out and checks that position rises between 0.0 and 1.0, then restarts the simulated board and checks that the saved calibration is still used.
