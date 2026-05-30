# Task 1 — V0.3 MCU bring-up, primary↔follower comms fix, board config

**Time estimate: ~4 dev days (range 3–6).** The variance is the comms-bug depth; signal-integrity or buffer-timing issues can push it to a full week. Sub-table:

| Days | Sub-task |
|------|----------|
| 0.5 | Verify Task 0 hand-off (18 motors driveable on FRONT board; LEFT/RIGHT known-broken state) and reproduce the comms failure with serial logging |
| 1–2 | Root-cause the primary↔follower comms failure (SYNC vs. forward vs. follower-receive vs. RX buffer) and ship the fix + repro test |
| 1 | Refactor role election: drop SYNC; EEPROM-only role load on boot; role set explicitly via `SET role …` |
| 0.5 | `SET` / `SET_LEFT` / `SET_RIGHT` / `GET` / `GET_LEFT` / `GET_RIGHT` command path on firmware + SDK + CLI |
| 0.5 | SETUP.md / Makefile updates and end-to-end power-cycle verification |

Goal: Building on Task 0's wired-and-labeled bench, validate that all 18 motors (12 linear + 6 hip-yaw) drive correctly through `KrabbyMCUSDK` from the GUI, fix the known bug where the primary (USB-connected) shield does not reliably drive the LEFT/RIGHT follower shields, and replace the SYNC-based role election with a simple EEPROM load plus generic `SET` / `GET` config commands the operator uses to assign roles and serials. The commands take a list of `key value` pairs in the same style as the existing `T` (target joints) command, so future config keys (M16's IMU mount transform, calibration metadata, etc.) plug in without inventing new commands. Per-board serial numbers can be written by `SET serial …` directly or by `calibrate-all --serials …` (Task 3) — same EEPROM slot, either path works.

Outputs
- All 18 motors driveable via the GUI and `KrabbyMCUSDK` end-to-end (FRONT and both followers, post-fix).
- Live pot **and** Hall encoder values visible per joint in telemetry / GUI (depending on the joint's KRABBY_PIN_REV configuration).
- Root cause of the primary↔follower comms failure identified, fixed, and a repro/regression test committed.
- Firmware refactor: SYNC-based role election replaced by a plain EEPROM load on boot; any board with unset EEPROM defaults to `ROLE_UNKNOWN` and waits for an explicit `SET role <PRIMARY|LEFT|RIGHT>`.
- New `SET` / `GET` command path with side-suffixed variants (`SET_LEFT`/`SET_RIGHT`/`GET_LEFT`/`GET_RIGHT`) that route directly to the corresponding follower over Serial1/Serial2. JOINTS-style key-value lists. All config persists in EEPROM.
- README/Makefile updates documenting the new bring-up flow.

Acceptance Criteria
- **1a** — All 18 motors (12 linear + 6 hip-yaw) drive from the GUI (`python -m firmware.gui`) and via `KrabbyMCUSDK.send_command_jog` / `send_command_joints` — not just the FRONT board.
- **1b** — Pot value (`avgPot` / raw) **and** Hall edge count visible in telemetry for every joint that has a Hall sensor wired (per `KRABBY_PIN_REV` mapping in `board_pins.h`).
- **1c** — Primary↔follower comms failure root-caused: which stage fails (SYNC, leader→follower forward, follower receive) is documented with captured serial logs at the failure point.
- **1d** — Comms failure fixed with a repro test (an integration test or a documented manual repro) that demonstrates the bug pre-fix and passes post-fix.
- **1e** — Role is loaded from EEPROM on boot; if EEPROM is unset, the board comes up `ROLE_UNKNOWN` and waits. **Role is set explicitly via `SET role <PRIMARY|LEFT|RIGHT>`**. The operator brings up a board, sends `SET role PRIMARY`, and the board persists that to EEPROM for next time. (The existing SYNC-based `determineRole()` handshake is removed.)
- **1f** — `SET` / `GET` command path implemented (firmware + SDK) with side-suffixed variants. JOINTS-style key-value lists. Serial is **async fire-and-forget for SET; sync request-reply for GET** (matching the existing `V` → `VER` reply pattern). There is **no `ERR` reply** — the SDK validates commands client-side, and the firmware silently ignores anything malformed.
  - `SET <key> <val> [<key> <val> …]` — writes one or more config keys to the board receiving the command. No reply. Allowed keys: `role`, `serial`.
  - `GET <key> [<key> …]` — reads one or more config keys from the receiving board; replies `GET <key> <val> [<key> <val> …]` (tagged reply, like `VER`). Allowed keys: `role`, `serial`.
  - `SET_LEFT <key> <val> …` / `GET_LEFT <key> …` — primary-only. The primary writes the bare `SET …` / `GET …` out **Serial1** (`leftSerial`) to the LEFT follower. For `GET_LEFT`, the primary reads the follower's `GET …` reply and rewrites it to `GET_LEFT …` on USB so the host can identify the source. On a follower (i.e. a non-primary that somehow received this on USB), the command is silently ignored.
  - `SET_RIGHT <key> <val> …` / `GET_RIGHT <key> …` — same shape over **Serial2** (`rightSerial`); reply rewritten to `GET_RIGHT …` on USB.
- **1g** — Config state (role, serial) persists in EEPROM and is reloaded on boot; survives a power cycle.
- **1h** — `ERR <token> <errorcode>` telemetry channel implemented (firmware + SDK) per §5. Wire format is a single ASCII line; emitted asynchronously by any command during execution; throttled to one ERR per `(token, code)` per active fault event. The SDK's `_reader_loop` parses ERR lines as tagged telemetry events. Tasks 2, 3, and 4 emit ERR lines through this channel from the canonical vocabulary in §5.
- **1i** — Firmware README updated with the new bring-up procedure: plug USB into the chosen primary, send `SET role PRIMARY`, then from that board `SET_LEFT role LEFT` and `SET_RIGHT role RIGHT`; verify with `GET_LEFT role` / `GET_RIGHT role`. Roles persist across reboot.

---

**NOTE:** Line numbers below are pointers and may drift.

## 1. Plug the motors in, verify both sensor paths

Three Arduino Mega + Krabby-Uno **v0.3** shields. The active board pin revision is selected at compile time via `KRABBY_PIN_REV` in `firmware/arduino/board_pins.h` — V0.3 is **Rev 3** (EN interleaved D22/D24/D26 + D23/D25/D27; Hall on D50–D52 + A12–A14; PWM D2–D13 unchanged; analog IS A6–A11 + POT A0–A5). Build with:

```bash
arduino-cli compile --fqbn arduino:avr:mega \
  --build-property "build.extra_flags=-DKRABBY_PIN_REV=3" \
  firmware/arduino
```

(or whatever the existing `firmware/Makefile` target is; update the Makefile if it doesn't already select rev 3 for the V0.3 build).

Task 0 already plugged the motors and ran a manual smoke test on the FRONT board. Task 1's bring-up verification is end-to-end through `KrabbyMCUSDK`, across all 18 joints. For each joint, verify in the GUI:
- **Position (pot)** — the `Pot` column in `firmware/gui/app.py` (`JointRow.update_from_telemetry` ~L59) shows the raw ADC value; it changes when the motor is moved by hand or jogged.
- **Current sense** — the `Cur` column updates when the motor is driven; ~0 idle, non-zero under load.
- **Hall edges** — the `Hall` column tracks cumulative edge count from `hallHwGetEdgeCount()` (`hall_hw.h`); increments when the motor moves on a joint that has a Hall sensor wired (Rev 3 only, slots 0–5 → D50/D51/D52 + A12/A13/A14).

If a pot, current, or Hall channel reads obviously wrong (always zero, always max, noise that doesn't track motion), document which slot/wire and triage before proceeding. Some basic changes to the firmware to properly forward hall signals may be required at this stage, as hall has not been implemented (hallB shares the POT wire, so you may see something, but unsure what you'd see).

## 2. Debug the primary↔follower comms failure

The known symptom: the USB-connected primary does not reliably drive LEFT and RIGHT followers, and the host logs "can't keep up" errors. The **most likely root cause is a 64-byte default serial RX buffer instead of the required 256-byte buffer in the locally-built firmware** — start the debug there.

### 2.1 Prime suspect — `SERIAL_RX_BUFFER_SIZE` not bumped to 256 in the local build

The leader Mega's RX buffer needs to hold ~200-byte telemetry lines from each of Serial1 (left follower) and Serial2 (right follower) while the main loop services USB and the actuator update. The Arduino AVR core defaults this to **64 bytes**, which drops the middle of forwarded lines. The fix is a build define: `-DSERIAL_RX_BUFFER_SIZE=256`. This is well-known, documented in [`firmware/SETUP.md`](../../krabby-research/firmware/SETUP.md) (§2.1 "Serial RX buffer"), and applied in two places **except the one that matters most for this debug**:

- ✅ `.github/workflows/publish-firmware.yml` (~L52–53) passes `--build-property compiler.cpp.extra_flags=-DSERIAL_RX_BUFFER_SIZE=256` (and `compiler.c.extra_flags=…`) to `arduino-cli compile`. So **S3-hosted release builds** (used by `krabby-firmware update`) are correct.
- ✅ `firmware/install.py` (~L15–16) writes a `platform.local.txt` to `~/.arduino15/packages/arduino/hardware/avr/1.8.7/platform.local.txt` containing the same defines. So **Linux/macOS dev machines that ran `krabby-firmware install`** have working IDE/CLI builds.
- ❌ **`firmware/Makefile` (`compile-firmware` / `upload-firmware` targets) does NOT pass the flag.** It sets `BUILD_PROPS` only for non-default `KRABBY_PIN_REV` (~L23–28). On any dev machine that didn't run `install.py` — or whose Arduino15 path doesn't match the hardcoded Linux/Mac path (Windows uses `%LOCALAPPDATA%\Arduino15`, not `~/.arduino15`), or whose AVR core version isn't `1.8.7` — `make upload-firmware` silently produces a 64-byte-buffer binary and the comms fail.

**The fix in this task:** bake `-DSERIAL_RX_BUFFER_SIZE=256` into the Makefile's `BUILD_PROPS` unconditionally, the same way CI passes it. Concretely:

```make
BUILD_PROPS := --build-property "compiler.cpp.extra_flags=-DSERIAL_RX_BUFFER_SIZE=256 -DKRABBY_PIN_REV=$(PIN_REV)" \
               --build-property "compiler.c.extra_flags=-DSERIAL_RX_BUFFER_SIZE=256 -DKRABBY_PIN_REV=$(PIN_REV)"
```

(Adjust quoting per make/shell conventions on Windows vs POSIX; arduino-cli accepts multiple defines in one `extra_flags` string.) Once the Makefile passes the flag, every build path — CI, `make upload-firmware`, an IDE build using `platform.local.txt` — produces an identical binary, regardless of whether `install.py` ran or what AVR core version is installed.

**Verify the binary actually has the bumped buffer** before declaring this fixed: compile, then `avr-objdump -h firmware/build/arduino.ino.elf` (or the equivalent), check the `.bss` size against a known-good CI build; or instrument the firmware to print `SERIAL_RX_BUFFER_SIZE` at boot and assert it equals 256 in a power-on log.

Once the Makefile flag lands, retest the primary→follower path. If "can't keep up" goes away and telemetry forwarding is clean, that was the bug. If symptoms persist, move on to the secondary suspects below.

### 2.2 Secondary suspects (only if §2.1 doesn't resolve it)

1. **SYNC election fails:** the primary never reaches `ROLE_FRONT` because it doesn't see SYNC tokens from both followers within the 3 s window (`determineRole()` in `arduino.ino` ~L149–234). Instrument: print `syncFromLeft`/`syncFromRight` state every 100 ms during election; capture follower serial output during the same window. (Note: §3 of this task removes SYNC election entirely, so don't sink hours into debugging the old path — if §2.1 doesn't fix it, jump straight to §3's refactor and re-test.)
2. **Leader forwards commands but followers don't see them:** the primary reaches `ROLE_FRONT`, accepts a `T` / `B` / `J` / `H` command from USB, and writes to `leftSerial`/`rightSerial` (~L305, ~L321, ~L344, ~L360), but the follower's `mainSerial` never has bytes to read. Instrument: log every byte the primary writes to `leftSerial` and every byte the follower reads from its `mainSerial`.
3. **Followers receive but don't apply:** the follower's `loop()` reads bytes but the command parser drops them. Instrument: log `cmdType` on the follower's side and confirm whether `parseCommands` returns the expected count.
4. **Hardware/wiring:** Rev 3 shield pinout for Serial1/Serial2 connectors (defined as `SERIAL_LEFT = Serial1` on D18/19 and `SERIAL_RIGHT = Serial2` on D16/17 — see `arduino.ino` ~L14–17); confirm the shield silkscreen matches and that the inter-board cables are correctly oriented.

Triage tool: loopback test — short Serial1 TX to Serial1 RX on a single board, write bytes, and confirm read-back. Then probe each leg of the actual path with the smallest possible payload (single `J` jog command) before scaling to batched `B` commands. **Capture serial logs at the failure point** for the acceptance criterion (1c).

### 2.3 Deliverable for §2

Once root-caused (1d), commit:
- The Makefile change so the local build matches the CI build (likely the actual fix, per §2.1).
- An integration test under `tests/integration/` or a documented manual repro (firmware unit + procedure) that demonstrates the bug pre-fix and passes post-fix.
- A SETUP.md update noting that the Makefile is now self-sufficient and `install.py`'s platform.local.txt write is a belt-and-suspenders backup, not a requirement.

## 3. Replace SYNC election with EEPROM-only role + explicit `SET role`

After Task 1's refactor, `determineRole()` in `arduino.ino` (~L149–234) is just an EEPROM load — the previous SYNC handshake on every boot is removed.

- **Boot:** load role from EEPROM (existing magic `0xAB` at addr 32–33). If EEPROM is uninitialized, role = `ROLE_UNKNOWN`.
- **Setting role:** operator sends `SET role <PRIMARY|LEFT|RIGHT>` over whatever serial the board is reachable on (USB for the board you're about to designate as primary; the primary's `SET_LEFT` / `SET_RIGHT` for the followers). The board writes the new role to EEPROM and triggers a re-init.
- **`ROLE_UNKNOWN` behavior:** the board doesn't drive any actuators — `setup()` skips `actuatorManager->initAll()` until role is known. It still accepts `SET` so the operator can assign a role.

Code changes:
- Replace `determineRole()` body with a simple EEPROM load (default `ROLE_UNKNOWN` if magic absent).
- Delete the SYNC token send/receive loop and the `ASSIGN_LEFT` / `ASSIGN_RIGHT` paths (~L19–20, ~L184–223). Keep the `roleName()` helper and the EEPROM helpers.
- Extend the command dispatcher in `loop()` (~L295–411) to recognize the new multi-letter tokens `SET`, `SET_LEFT`, `SET_RIGHT`, `GET`, `GET_LEFT`, `GET_RIGHT` alongside the existing single-letter `T`/`B`/`J`/`C`/`H`/`V`. On the primary, `_LEFT`/`_RIGHT` variants strip the suffix and write the bare command to the hard-wired follower serial (`leftSerial` for `_LEFT`, `rightSerial` for `_RIGHT`); for the GET variants, the primary reads the follower's `GET …` reply (using the existing `readVerLine` pattern in `arduino.ino` ~L256–291) and rewrites the leading tag to `GET_LEFT …` / `GET_RIGHT …` on USB so the host can identify the source. On a non-primary, `_LEFT`/`_RIGHT` are silently dropped (no reply) — the SDK won't send them to a non-primary anyway.

## 4. `SET` / `SET_LEFT` / `SET_RIGHT` / `GET` / `GET_LEFT` / `GET_RIGHT` command schema

Single-line ASCII, mirroring the existing `T` (target joints) command's "list of name-value pairs" shape so the parser logic is uniform and new config keys plug in without inventing new commands. The side-suffixed variants encode the target board in the command name itself — no nested commands; the primary just looks at the suffix and routes to the corresponding serial port.

**Serial is async.** SET is fire-and-forget (matching how `T`/`B`/`J`/`C`/`H` already work). GET is request/reply, tagged like the existing `VER` reply pattern. The MCU SDK validates commands client-side, so the firmware can silently drop anything it doesn't recognize.

```
SET       <key> <val> [<key> <val> ...]   → (no reply)
SET_LEFT  <key> <val> [<key> <val> ...]   → (no reply; primary forwards bare "SET …" out Serial1)
SET_RIGHT <key> <val> [<key> <val> ...]   → (no reply; primary forwards out Serial2)
GET       <key> [<key> ...]               → "GET <key> <val> [<key> <val> ...]"
GET_LEFT  <key> [<key> ...]               → "GET_LEFT <key> <val> ..."   (primary reads follower's "GET …" reply, rewrites the tag to "GET_LEFT" on USB)
GET_RIGHT <key> [<key> ...]               → "GET_RIGHT <key> <val> ..."  (same over Serial2)
```

**Examples** (host talking to the primary over USB):

```
> SET role PRIMARY
> GET role serial
< GET role PRIMARY serial PRI-0042
> SET_LEFT role LEFT
> GET_LEFT role serial
< GET_LEFT role LEFT serial LEF-0007
> SET_RIGHT role RIGHT
> GET_RIGHT role serial
< GET_RIGHT role RIGHT serial RGT-0019
> SET serial PRI-0099
> GET serial
< GET serial PRI-0099
```

**Allowed keys** for M17:

| Key | `SET` | `GET` | Notes |
|---|---|---|---|
| `role` | yes (`PRIMARY` / `LEFT` / `RIGHT` / `UNKNOWN`) | yes | Persisted to EEPROM at addr 32–33 (existing magic `0xAB`) |
| `serial` | yes | yes | Persisted to EEPROM at addr 35–50. May also be written by `calibrate-all --serials …` (Task 3) — same EEPROM slot, last-write-wins |

Firmware version / branch / commit are **not** part of this command — the existing `V` → `VER <version> <branch> <commit>` reply already provides that and stays unchanged. Per-joint `calibration_state` is also outside the scope of `SET`/`GET`; if Task 2 wants to surface it on the wire, it adds its own tagged command (e.g. `CAL_STATE` reply). New config keys added by later milestones (e.g. M16's IMU mount transform) extend the table above without introducing new wire commands.

**Notes:**
- Multi-letter command parsing: extend the dispatcher in `arduino.ino` `loop()` (~L295–411) to read a leading token (chars-until-whitespace) instead of a single peeked char. Single-letter `T`/`B`/`J`/`C`/`H`/`V` paths stay backward-compatible — they're just tokens of length 1.
- `SET_LEFT` / `SET_RIGHT` / `GET_LEFT` / `GET_RIGHT` are primary-only on the wire. A non-primary that receives one silently drops it; there's no error reply (the SDK won't send these to a non-primary in normal use).
- The MCU SDK is the validation layer: it rejects unknown keys, invalid role values, and non-primary destinations client-side before the bytes ever hit the serial port.

**Parsing on the firmware side** can reuse the existing `parseCommands` shape from the `T`-command path (`arduino.ino` ~L302 — `parseCommands(payload, cmdBuf, CMD_BUF_SIZE)`): both `SET` and `GET` are name-value lists, the same token walker handles both. Unknown tokens are silently dropped — no reply.

**EEPROM layout lives in a single struct in a new header `firmware/arduino/eeprom_layout.h`** — not in scattered constants across `arduino.ino` and `actuator_manager.h`. Read/write uses Arduino's standard struct serializer (`EEPROM.put(addr, struct)` / `EEPROM.get(addr, struct)`), so adding/removing fields is a struct edit + schema-version bump, not a hand-rolled byte layout. Sketch:

```cpp
// firmware/arduino/eeprom_layout.h
#pragma once
#include <stdint.h>

constexpr uint16_t EEPROM_MAGIC      = 0xK17C;   // pick a unique sentinel
constexpr uint8_t  EEPROM_SCHEMA_VER = 1;

enum BoardRole : uint8_t { ROLE_UNKNOWN = 0, ROLE_PRIMARY = 1, ROLE_LEFT = 2, ROLE_RIGHT = 3 };

struct EepromLayout {
    uint16_t  magic;            // EEPROM_MAGIC if valid; anything else → uninitialized → default UNKNOWN
    uint8_t   schema_version;   // EEPROM_SCHEMA_VER; bumped when struct changes
    BoardRole role;             // PRIMARY / LEFT / RIGHT / UNKNOWN
    char      serial[16];       // zero-padded ASCII; "" if unset
    // Task 2 extends with JointCalBlock joints[6];  ← reserve the layout now
    uint32_t  crc32;            // computed over all prior bytes
};

constexpr int EEPROM_BASE_ADDR = 0;  // single struct lives at addr 0; replaces the old ad-hoc map
```

`saveConfig()` / `loadConfig()` are one-liners around `EEPROM.put` / `EEPROM.get` on this struct, with the magic + CRC checked on load. The legacy `CalData` at the old addr 0 with magic `0xDEADBEEF` is deprecated by this rewrite — Task 2's `JointCalBlock` is added as a struct field above, in the same header, persisted by the same `EEPROM.put` call.

Document the schema in a comment block in `eeprom_layout.h` itself (single source of truth) so Tasks 2/3 + M16 extend the struct without bookkeeping address ranges.

## 5. `ERR` telemetry channel — asynchronous error output

`SET` / `GET` are command-response. Joint telemetry is a continuous stream. **`ERR` is a third, separate channel: asynchronous error lines the firmware emits during command execution whenever something goes wrong.** This is the wire format every other task in M17 leans on for its error reporting; defining it here keeps the protocol surface in one place.

**Wire format** (single line, ASCII, matching the existing `T`/`B`/`J`/`V`/telemetry style):

```
ERR <token> <errorcode>
```

- `<token>` is the subsystem the error is scoped to. For joint-level errors that's the joint name (`RLHL`, `FRKL`, …). Future non-joint subsystems can use their own tokens (e.g. `ERR system <code>`); each consuming task documents the tokens it emits.
- `<errorcode>` is a string drawn from the vocabulary owned by the consuming task. Strings, not numeric IDs, so the wire is self-describing. (Task 4 owns the calibration vocabulary — `motor_did_not_move`, `current_sense_no_signal`, `not_calibrated`, etc.)
- Always exactly two tokens after `ERR`.

**Semantics:**

- **Fire-and-forget.** ERR is not a response to a command; it's telemetry. Any command (`T`, `B`, `J`, `C`/`calibrate-all`, the future `validate-current-sense`, anything else) can emit ERR lines while executing. The firmware never blocks waiting for the host to read them.
- **Throttled.** No flooding — the firmware emits at most one `ERR <joint> <code>` per joint per active stall/fault event (e.g. one `motor_did_not_move` per motion attempt, not one per loop iteration). The mechanism: keep a per-(joint, code) "emitted this event" flag, clear it when the fault condition clears (motor moves, command ends, joint state changes).
- A command completing without firing an `ERR` line is considered success. `V` and the telemetry stream are the only commands that produce affirmative output.
- **Forwarded by the leader.** Followers emit ERR on their UART uplink; the leader's `forwardFullLines()` (~`arduino.ino` L117) relays them to USB unchanged — so the host sees all 18 joints' ERR lines through one serial port.

**SDK behavior** (`firmware/krabby_mcu.py`):

- `_reader_loop` (~L135–141) dispatches lines by prefix. Add `elif line.startswith("ERR ")` next to the existing telemetry / `VER` dispatch.
- Parse `ERR <token> <code>` into a structured event and surface it via a registered callback (or a simple in-memory ring of recent errors the GUI / `explain_failures` can read).
- The SDK does **not** treat ERR as a response to the most recent command; it treats it as a tagged telemetry event the host can choose to act on (or display).
- For operator-facing display, the SDK looks up the code in Task 4's `_FIX_INSTRUCTIONS` table and prints a human-readable fix-instruction.

**Used by:** Task 2 §2f (motor health), Task 2 §6.5 (`not_calibrated` rejection), Task 3 §6 (cal failures), Task 4 (current-sense validation failures).

**Canonical error-code vocabulary** (the only valid `<errorcode>` strings; the firmware emits these as inline string literals where the fault is detected):

| Code | Emitted when |
|---|---|
| `motor_did_not_move` | Motor commanded with PWM applied, but the wired sensor reading didn't change within the stall timeout (Task 2 §2f) |
| `motor_jammed` | Same as above but current is high — true mechanical stall, not an end-stop (per the `isStalled` current-aware split in the [OVERVIEW FAQ](OVERVIEW.md)) |
| `pot_value_invalid` | Pot reading floating or stuck outside `[0, 1023]` while motion was commanded (Task 2) |
| `hall_no_edges` | Hall channel not incrementing on commanded motion of a HALL-typed joint (Task 2) |
| `hall_drift` | Per-joint cal's two repeat sweeps disagree beyond tolerance (Task 2 §2c) |
| `not_calibrated` | Position target sent to a `PARTIALLY_CALIBRATED` Hall joint (Task 2 §6.5) |
| `current_sense_no_signal` | Current-sense validation: unloaded and loaded readings essentially equal — no change under load (Task 4) |
| `current_sense_no_spike` | Current-sense validation: loaded reading didn't rise enough above unloaded, or rose to an anomalous extreme (Task 4) |

New codes are added to this table when a task starts emitting them — the table is the single source of truth. The SDK's `_FIX_INSTRUCTIONS` dict (Task 4 §5) maps these strings to operator-facing messages.

## 6. SDK plumbing

Extend `firmware/krabby_mcu.py` `KrabbyMCUSDK` with **two methods total**, each taking an optional `board` parameter that defaults to the board on USB (the primary):

```python
mcu.send_set(role="PRIMARY", serial="PRI-0042")              # writes to the primary itself
mcu.send_set(board="LEFT", role="LEFT", serial="LEF-0007")   # primary forwards via Serial1
mcu.send_set(board="RIGHT", role="RIGHT")                    # primary forwards via Serial2

mcu.send_get("role", "serial")                # reads from the primary
mcu.send_get("role", "serial", board="LEFT")  # reads from the left follower
```

Internally the SDK picks the wire command — `SET` / `SET_LEFT` / `SET_RIGHT` (or `GET` / `GET_LEFT` / `GET_RIGHT`) — based on `board`. The caller never types `_LEFT` / `_RIGHT`.

The SDK is the validation layer (per §4): it rejects unknown keys and invalid role values **before serializing to the wire**, raising a Python `ValueError`. Nothing invalid reaches the MCU, so the wire format has no `ERR` reply path.

- `send_set` returns immediately (fire-and-forget). To confirm, the caller follows up with `send_get` for the same keys.
- `send_get` writes the GET line and **blocks on the tagged reply** in `_reader_loop` (~L135–141), parsing `GET …` / `GET_LEFT …` / `GET_RIGHT …` lines and routing them to the waiting caller — same pattern as `read_version` (~L226–237) waits for `VER …`. Returns the result as a dict.
- `firmware/cli.py` or `firmware/__main__.py` — add CLI subcommands matching the SDK shape: `krabby-firmware set [--board left|right|primary] <key>=<val>…` and `krabby-firmware get [--board …] <key>…`.

## 7. Update the bring-up flow in the README

Document the new procedure in `firmware/SETUP.md` (or the relevant existing README):

1. Flash all three boards with the current firmware (`krabby-firmware update`).
2. Plug USB into whichever Mega you want to be primary.
3. From the host: `krabby-firmware set role=PRIMARY`. (The board persists `PRIMARY` to EEPROM and re-inits.)
4. Power on the other two boards (no USB).
5. From the host (still on the primary's USB): `krabby-firmware set-left role=LEFT` and `krabby-firmware set-right role=RIGHT`.
6. Verify: `krabby-firmware get role`, `get-left role`, `get-right role`.
7. Power-cycle the whole rig; verify the same three GETs still return the right roles (persistence check).
8. Optionally set per-board serials now with `set serial=…`, `set-left serial=…`, `set-right serial=…`. Serials and joint cal are independent — assign whenever convenient.

## 8. Deliverable checklist
- [ ] All 18 motors driveable via GUI through the primary; whichever sensor each motor has (pot or Hall) plus current visible.
- [ ] Primary↔follower comms failure root-caused, fixed, and a repro test committed; Makefile now bakes in `-DSERIAL_RX_BUFFER_SIZE=256` so local builds match CI.
- [ ] SYNC election removed; role is loaded from EEPROM and set explicitly via `SET role …`.
- [ ] `SET` / `SET_LEFT` / `SET_RIGHT` / `GET` / `GET_LEFT` / `GET_RIGHT` (JOINTS-style key-value lists) work on firmware + SDK + CLI. SET is fire-and-forget; GET replies with a tagged key-value list. The SDK validates commands client-side; the firmware drops anything it doesn't recognize.
- [ ] `ERR <token> <errorcode>` telemetry channel implemented per §5 — async, throttled, single ASCII line, parsed by `_reader_loop` as a tagged telemetry event. Tasks 2/3/4 consume this for their error reporting.
- [ ] EEPROM persistence verified for role and serial across a power cycle.
- [ ] SETUP.md + Makefile updated with the new bring-up flow.
