# Task 3 — Serial emulation and three-board firmware

Goal: extend the simulated firmware to speak the serial protocol end to end, so the existing krabby-SDK (`firmware/krabby_mcu.py`) can connect to the simulated MCU, send commands, and receive telemetry back, closing the command-to-motion-to-telemetry loop against IsaacSim. Tasks 1 and 2 already produce real drive and sensor values, so wiring serial makes them flow to the SDK, and the `python -m firmware.gui` current-sense demonstration lands here. The task then emulates the inter-board serial links, with role election and leader-to-follower forwarding, so three simulated Arduinos cooperate in one IsaacSim scene and drive all 18 legs, each board bound to its own six joints. The SDK stays unchanged throughout.

Outputs
- A simulated `Serial`/`HardwareSerial` implementation for the firmware host build, wired to a virtual serial device the SDK can open.
- The full host-to-MCU command protocol (`T`, `B`, `J`, `C`, `H`, `V`) parsed by the simulated firmware, with role-prefixed telemetry emitted back on the ~50 ms tick.
- krabby-SDK (`KrabbyMCUSDK`) connecting to the simulated MCU via `KRABBY_MCU_PORT`, driving legs, and reading telemetry, demonstrated in tests.
- Inter-board serial emulation: three simulated Arduinos performing role election (FRONT, LEFT, RIGHT), with the leader forwarding follower telemetry and relaying commands.
- Three MCU emulator instances bound to three disjoint sets of 6 joints in one IsaacSim scene, controlling all 18 legs.
- The `python -m firmware.gui` end-to-end current-sense demo, where the GUI "Cur" column responds as legs load the ground.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-3---serial-emulation-and-three-board-firmware).)

---

Note: line numbers are pointers to the right function and may drift.

---

## 1. The serial contract the simulator must speak

`firmware/krabby_mcu.py` (`KrabbyMCUSDK`) is the host-side driver. It opens `serial.Serial()` by path (`connect`, ~L80-109), holds DTR false to avoid resetting a real board, sleeps around 5 s for boot and election, spawns a reader thread (`_reader_loop`, ~L111-155), and immediately sends `H\n`. The simulated firmware implements both directions of this contract.

Host to MCU, newline-terminated ASCII:

| Bytes | Meaning | Firmware parse (`arduino.ino`) |
|---|---|---|
| `T <name> <val> [<name> <val> …]\n` | position targets, val in [0,1] to 3dp | ~L298-307, `parseCommands` (`command.h` ~L35), `applyCommands`/`setTarget` |
| `B <name> <pwm> <name> <pwm> … \n` | batch jog, pwm in [−255,255], trailing space before `\n` | ~L308-337, `handleJog` |
| `J<name> <pwm>\n` | single jog, with no space after `J` | ~L338-346 |
| `C\n` | start auto-calibration | ~L347-354 |
| `H\n` | hold and de-energize all | ~L355-362, `holdAll()` |
| `V\n` | request version | ~L363-404 |

The SDK's emitters are `send_command_joints` (`T …`, ~L177), `send_commands_jog` (`B …`, ~L201), `send_command_jog` (`J…`, ~L217), `send_command_joints_hold` (`H`, ~L246), `send_command_calibrate` (`C`, ~L239), and `read_version` (`V`, ~L226).

MCU to host:

Telemetry goes out every `TELEMETRY_INTERVAL_MS = 50` (`arduino.ino` ~L105, L426-433) as `"<ROLE>; " + ActuatorManager::printTelemetry(...)`, where `<ROLE>` is one of `FRONT`, `UNKWN`, `LEFT `, or `RIGHT`. Note the trailing space in `LEFT `, which comes from `roleName()` (`arduino.ino` ~L46-56). Each of the 6 segments is `name pos pot current enL enR pwmL pwmR saf`, nine tokens; see `LinearActuator::printTelemetry` (~L196-219), where `enL` and `enR` both carry the same `digitalRead(pinEn)`, `pwmL = currentPwm<0?abs:0`, `pwmR = currentPwm>0?currentPwm:0`, and `saf` is the hall edge count.

The version reply is `VER <v>|<v>|<v> <branch>|<b>|<b> <commit>|<c>|<c>\n`, with the leader aggregating three boards and a lone board replying `VER <v> <b> <c>`. It is parsed by `parse_ver_reply` (`krabby_mcu.py` ~L23-38).

The SDK also loosely recognizes boot and status lines: `ROLE_HINT: …`, `Krabby Ready …`, `--- SYNC ---`, `ROLE: …`, and calibration lines matching `…CALIBRATION…`, `Saved`, or `CAL`.

The prefix constants the SDK matches on are `_TELEMETRY_LINE_PREFIXES = ("FRONT;", "UNKWN;", "LEFT ;", "RIGHT;")` (`krabby_mcu.py` ~L47-53). Match them exactly, trailing space included.

## 2. Where the simulated serial plugs in

The SDK takes a port path rather than a transport object: `connect()` builds `serial.Serial()`, sets `ser.port = self.port`, and calls `ser.open()` (~L86-91). Two seams follow from that, and both leave the SDK alone.

The first is a virtual serial device selected through `KRABBY_MCU_PORT` or the `port=` argument. `firmware/mcu_port.py:default_port()` (~L16-61) honors `KRABBY_MCU_PORT` first, so point it at a PTY on Linux or macOS via `os.openpty()`, or a virtual COM pair on Windows via com0com or socat, with the simulated MCU driving the other end. pyserial opens it like any device.

The second is monkeypatching `serial.Serial` for pure unit tests, which is what the existing tests do: `tests/unit/firmware/test_krabby_mcu.py` injects `sdk.ser = Mock()`.

Use the virtual device for the integration loop and the monkeypatch for fast unit tests. Two behaviors need tolerating: the roughly 5 s boot sleep (`krabby_mcu.py` ~L93) and the automatic `H\n` right after open. On the firmware host build, the `Serial`, `Serial1`, and `Serial2` objects the C++ writes to must be backed by this transport, with `Serial` carrying the host and USB link and `Serial1` and `Serial2` carrying the inter-board links described in section 4.

## 3. Emitting telemetry from the simulated firmware

`ActuatorManager::printTelemetry(Print& out)` (~L316-324) joins the 6 `LinearActuator::printTelemetry` segments with `;` and calls `println()`. Tasks 1 and 2 already make every telemetry field real — the pot tracks the joint, current-sense comes from physics, and hall edge counts accumulate — so the fields fall out once the `Print` and `Serial` stream is wired to the transport:

`pos` is `getPos()` from the simulated pot, `pot` is `(int)avgPot`, `current` is `(int)avgIS` from Task 2's physics current-sense, `enL` and `enR` are `digitalRead(pinEn)` from Task 2, `pwmL` and `pwmR` come from `currentPwm`, and `saf` is `hallHwGetEdgeCount(hallSlot)` with Task 2's real edge counts.

Task 3's telemetry work is therefore transport plumbing plus emitting on the `millis()`-based 50 ms cadence.

## 4. Three boards and inter-board serial

On real hardware, three Megas elect roles over `Serial1` and `Serial2`, and the FRONT leader aggregates everything to USB. Reproduce this with three MCU emulator instances and emulated inter-board links.

Role election runs in `determineRole()` (`arduino.ino` ~L149-234): a three-second `millis()` loop broadcasting `"SYNC"` on `Serial1` and `Serial2` every 10 ms, with `readStringUntil('\n')` reading peers. `ASSIGN_LEFT` and `ASSIGN_RIGHT` make a board a follower, SYNC arriving from both sides makes a board FRONT so it assigns followers and sets `leftSerial` and `rightSerial`, and a timeout leaves the board ROLE_UNKNOWN driving front actuators. To emulate three boards, wire each instance's `Serial1` and `Serial2` to the appropriate peer instance's stream using a pair of in-memory byte queues per link, and let the real election code run.

Command relay happens in `loop()`, where the leader forwards `T `, `B `, `J`, `C`, `H`, and `V` payloads to followers (~L305-306, 313-336, 344-345, 352-361, 384-402).

Telemetry forwarding happens in `forwardFullLines()` (~L117-147), which drains a follower `HardwareSerial` and forwards complete newline-terminated lines to `mainSerial`. The leader calls it around `updateAll()` (~L416-424), so the host receives `FRONT;…`, `LEFT ;…`, and `RIGHT;…` each cycle.

A single-leader emulator that owns all 18 joints and emits all three role-prefixed telemetry lines itself is a useful stepping stone for the first round trip. The deliverable is the true three-instance election and forwarding, which exercises the real multi-board code paths, so build both in that order.

## 5. Binding three boards to 18 joints in one scene

The 18 joints split leg-major across three boards. The canonical `{leg}_{joint_type}` ordering lives in `hal/server/robot_definition_krabby_hex.py`, and the firmware's per-board lists are `ACT_LIST_FRONT`, `ACT_LIST_LEFT`, and `ACT_LIST_RIGHT` (`arduino.ino` ~L84-86).

Each emulator instance gets its own Task 1 `SimulatedRobotConfig` bound to that board's 6 joints of the shared crab Articulation. `RobotCore`, or a multi-board coordinator above it, advances all three firmware instances, collects their pin writes, sets all 18 joint targets, steps physics once, and feeds each board's 6 pots back. Resolve each board's joint indices via `robot.find_joints(...)` regex rather than assuming index order.

## 6. Verify

- Connect: start the simulated MCU on a PTY, set `KRABBY_MCU_PORT`, call `KrabbyMCUSDK().connect()`, and assert it succeeds and telemetry begins flowing.
- Round trip: `send_command_jog("FLHY", 200)`, then assert the FL hip-yaw joint moves and `mcu.joints["FLHY"].pos` and `.pwm` update via parsed telemetry.
- Protocol coverage: `read_version()` returns a parseable `VER`; `send_command_joints_hold()` de-energizes, with EN low in telemetry; `send_command_calibrate()` runs the calibration state machine to completion against simulated limits.
- Election: boot three instances and assert exactly one FRONT, one LEFT, and one RIGHT, that the leader owns the host port, and that all three role-prefixed telemetry lines arrive per cycle.
- All 18: iterate the 18 joint names, jog each, and assert only the corresponding Isaac joint moves.
- GUI current-sense end to end: run `python -m firmware.gui` against the simulated MCU, press a foot into the floor and watch the "Cur" column rise, lift the body and watch it reach the high band near 30, then rest and watch it fall toward 0. This is the payoff of Task 2's current-sense over this task's serial, and it confirms telemetry stays nominal with all fields parsing.
