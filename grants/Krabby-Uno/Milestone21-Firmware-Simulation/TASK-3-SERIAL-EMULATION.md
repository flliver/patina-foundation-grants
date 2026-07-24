# Task 3 — Serial emulation and three-board firmware

Goal: Extend the simulated firmware to speak the **serial protocol** end to end, so the existing **krabby-SDK** (`firmware/krabby_mcu.py`) can connect to the simulated MCU, send commands, and receive telemetry back — closing the full command → motion → telemetry loop against IsaacSim. Because Tasks 1–2 already give real drive + sensor values (pot, current-sense, hall), wiring serial makes those values flow to the SDK — so this is also where the **`python -m firmware.gui` current-sense demonstration** lands (move the robot, watch the "Cur" column respond). Then emulate the **inter-board serial links** (role election + leader↔follower forwarding) so **three** simulated Arduinos cooperate in one IsaacSim scene, driving all **18 legs**, each board bound to its own set of six joints. This is serial-transport and multi-instance work built on the Task 1 drive + Task 2 sensor interfaces; the SDK is unchanged.

Outputs
- A simulated `Serial`/`HardwareSerial` implementation for the firmware host build, wired to a virtual serial device the SDK can open.
- The full host→MCU command protocol (`T`/`B`/`J`/`C`/`H`/`V`) parsed by the simulated firmware, and role-prefixed telemetry emitted back on the ~50 ms tick.
- krabby-SDK (`KrabbyMCUSDK`) connecting to the simulated MCU via `KRABBY_MCU_PORT`, driving legs, and reading telemetry — demonstrated in tests.
- Inter-board serial emulation: three simulated Arduinos performing role election (FRONT/LEFT/RIGHT), with the leader forwarding follower telemetry and relaying commands.
- Three MCU emulator instances bound to three disjoint sets of 6 joints in one IsaacSim scene, controlling all 18 legs.
- The `python -m firmware.gui` end-to-end current-sense demo: with Task 2's sensors flowing over serial, the GUI "Cur" column responds as legs load the ground.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-3---serial-emulation-and-three-board-firmware).)

---

**NOTE:** Line numbers are pointers to the right function and may drift.

---

## 1. The serial contract the simulator must speak

`firmware/krabby_mcu.py` (`KrabbyMCUSDK`) is the host-side driver. It opens `serial.Serial()` by path (`connect`, ~L80–109), holds DTR false (to avoid resetting a real board), sleeps ~5 s for boot/election, spawns a reader thread (`_reader_loop`, ~L111–155), and immediately sends `H\n`. The simulated firmware must implement both directions of this contract.

**Host → MCU (commands the SDK writes), newline-terminated ASCII:**

| Bytes | Meaning | Firmware parse (`arduino.ino`) |
|---|---|---|
| `T <name> <val> [<name> <val> …]\n` | position targets, val ∈ [0,1] (3dp) | ~L298–307 → `parseCommands` (`command.h` ~L35) → `applyCommands`/`setTarget` |
| `B <name> <pwm> <name> <pwm> … \n` | batch jog, pwm ∈ [−255,255] (trailing space before `\n`) | ~L308–337 → `handleJog` |
| `J<name> <pwm>\n` | single jog (no space after `J`) | ~L338–346 |
| `C\n` | start auto-calibration | ~L347–354 |
| `H\n` | hold / de-energize all | ~L355–362 → `holdAll()` |
| `V\n` | request version | ~L363–404 |

The SDK's emitters: `send_command_joints` (`T …`, ~L177), `send_commands_jog` (`B …`, ~L201), `send_command_jog` (`J…`, ~L217), `send_command_joints_hold` (`H`, ~L246), `send_command_calibrate` (`C`, ~L239), `read_version` (`V`, ~L226).

**MCU → Host (lines the SDK reads):**
- **Telemetry**, every `TELEMETRY_INTERVAL_MS = 50` (`arduino.ino` ~L105, L426–433): `"<ROLE>; " + ActuatorManager::printTelemetry(...)`. `<ROLE>` ∈ {`FRONT`, `UNKWN`, `LEFT `, `RIGHT`} — **note the trailing space in `LEFT `** (from `roleName()`, `arduino.ino` ~L46–56). Each of 6 segments is `name pos pot current enL enR pwmL pwmR saf` (9 tokens); see `LinearActuator::printTelemetry` (~L196–219): both `enL`/`enR` are the same `digitalRead(pinEn)`, `pwmL = currentPwm<0?abs:0`, `pwmR = currentPwm>0?currentPwm:0`, and `saf` is the hall edge count.
- **Version reply**: `VER <v>|<v>|<v> <branch>|<b>|<b> <commit>|<c>|<c>\n` (leader aggregates three boards; a lone board replies `VER <v> <b> <c>`). Parsed by `parse_ver_reply` (`krabby_mcu.py` ~L23–38).
- **Boot/status lines** the SDK loosely recognizes: `ROLE_HINT: …`, `Krabby Ready …`, `--- SYNC ---`, `ROLE: …`, and calibration `…CALIBRATION…`/`Saved`/`CAL`.

The prefix constants the SDK matches on: `_TELEMETRY_LINE_PREFIXES = ("FRONT;", "UNKWN;", "LEFT ;", "RIGHT;")` (`krabby_mcu.py` ~L47–53). Match them **exactly**, trailing space and all.

## 2. Where the simulated serial plugs in

The SDK is **string-port based**, not object-injectable: `connect()` builds `serial.Serial()`, sets `ser.port = self.port`, and calls `ser.open()` (~L86–91). It never takes a pre-opened transport. The clean seams (no SDK change):

1. **Virtual serial device via `KRABBY_MCU_PORT` or `port=`.** `firmware/mcu_port.py:default_port()` (~L16–61) honors `KRABBY_MCU_PORT` first. Point it at a PTY (Linux/macOS: `os.openpty()`) or a virtual COM pair (Windows: com0com/socat), with the simulated MCU driving the other end. pyserial opens it like any device.
2. **Monkeypatch `serial.Serial`** for pure unit tests — this is what the existing tests do (`tests/unit/firmware/test_krabby_mcu.py` injects `sdk.ser = Mock()`).

Prefer (1) for the integration loop and (2) for fast unit tests. Two behaviors to tolerate: the ~5 s boot sleep (`krabby_mcu.py` ~L93) and the auto-`H\n` right after open. On the firmware host build, the `Serial`/`HardwareSerial` objects the C++ writes to (`arduino.ino` uses `Serial`, `Serial1`, `Serial2`) must be backed by this transport — `Serial` ↔ the host/USB link, `Serial1`/`Serial2` ↔ the inter-board links (§4).

## 3. Emitting telemetry from the simulated firmware

`ActuatorManager::printTelemetry(Print& out)` (~L316–324) joins the 6 `LinearActuator::printTelemetry` segments with `;` and `println()`s. Tasks 1–2 already make every telemetry field real (pot tracks the joint, current-sense from physics, hall edge counts), so the fields fall out for free **once the `Print`/`Serial` stream is wired to the transport**:
- `pos` = `getPos()` (from simulated pot), `pot` = `(int)avgPot`, `current` = `(int)avgIS` (physics current-sense, Task 2), `enL`/`enR` = `digitalRead(pinEn)` (Task 2), `pwmL`/`pwmR` from `currentPwm`, `saf` = `hallHwGetEdgeCount(hallSlot)` (real edge counts, Task 2).

So Task 3's telemetry work is mostly transport plumbing plus emitting on the `millis()`-based 50 ms cadence.

## 4. Three boards and inter-board serial (`determineRole` + forwarding)

On real hardware, three Megas elect roles over `Serial1`/`Serial2` and the FRONT leader aggregates everything to USB. Reproduce this with three MCU emulator instances and emulated inter-board links.

- **Role election** — `determineRole()` (`arduino.ino` ~L149–234): a 3-second `millis()` loop broadcasting `"SYNC"` on `Serial1`/`Serial2` every 10 ms; `readStringUntil('\n')` reads peers; `ASSIGN_LEFT`/`ASSIGN_RIGHT` make a board a follower; SYNC from both sides makes a board FRONT (leader, assigns followers, sets `leftSerial`/`rightSerial`); timeout → ROLE_UNKNOWN (front actuators). To emulate three boards, wire each instance's `Serial1`/`Serial2` to the appropriate peer instance's stream (a pair of in-memory byte queues per link), and let the real election code run.
- **Command relay** — in `loop()`, the leader forwards `T `/`B `/`J`/`C`/`H`/`V` payloads to followers (~L305–306, 313–336, 344–345, 352–361, 384–402).
- **Telemetry forwarding** — `forwardFullLines()` (~L117–147) drains a follower `HardwareSerial` and forwards only complete `\n`-terminated lines to `mainSerial`. The leader calls it around `updateAll()` (~L416–424), so the host receives `FRONT;…`, `LEFT ;…`, `RIGHT;…` each cycle.

**Simplification path:** for the first round trip you can run a **single** leader emulator that already owns all 18 joints and emits all three role-prefixed telemetry lines itself, skipping inter-board emulation. Then add the true three-instance election + forwarding to exercise the real multi-board code paths. Do both — the single-board shortcut is a stepping stone, not the deliverable.

## 5. Binding three boards to 18 joints in one scene

The 18 joints split leg-major across three boards (see `hal/server/robot_definition_krabby_hex.py` for the canonical `{leg}_{joint_type}` ordering, and the firmware's per-board `ACT_LIST_FRONT/LEFT/RIGHT` in `arduino.ino` ~L84–86). Each emulator instance gets its own `SimulatedRobotConfig` (Task 1) bound to that board's 6 joints of the shared crab Articulation. `RobotCore` (or a multi-board coordinator) advances all three firmware instances, collects their pin writes, sets all 18 joint targets, steps physics once, and feeds each board's 6 pots back. Resolve each board's joint indices via `robot.find_joints(...)` regex (e.g. the FRONT board's legs) rather than assuming index order.

## 6. Verify
- **Connect:** start the simulated MCU on a PTY, set `KRABBY_MCU_PORT`, `KrabbyMCUSDK().connect()`; assert no error and telemetry begins flowing.
- **Round trip:** `send_command_jog("FLHY", 200)`; assert the FL hip-yaw joint moves and that `mcu.joints["FLHY"].pos`/`.pwm` update via parsed telemetry.
- **Protocol coverage:** `read_version()` returns a parseable `VER`; `send_command_joints_hold()` de-energizes (EN low in telemetry); `send_command_calibrate()` runs the cal state machine to completion against simulated limits.
- **Election:** boot three instances; assert exactly one FRONT + one LEFT + one RIGHT, the leader owns the host port, and all three role-prefixed telemetry lines arrive per cycle.
- **All 18:** iterate the 18 joint names, jog each, assert only the corresponding Isaac joint moves.
- **GUI current-sense (end-to-end):** run `python -m firmware.gui` against the simulated MCU; press a foot into the floor and watch the "Cur" column rise, lift the body and watch it reach the high band (~30), rest and watch it fall to ~0. This is the end-to-end payoff of Task 2's current-sense plus this task's serial, and confirms telemetry stays nominal (all fields parse).
