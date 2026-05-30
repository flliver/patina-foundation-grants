# Task 2 — Per-joint calibration (single sensor per motor, direction flip, EEPROM, normalized to [0.0, 1.0])

**Time estimate: ~3 dev days (range 2.5–4).** Sub-table:

| Days | Sub-task |
|------|----------|
| 0.5 | Sensor abstraction in `LinearActuator` (`sensor_type` = POT or HALL); `sensorReversed` plumbing at `getRawPos()` |
| 1 | Hall signed-count quadrature (ISR with B-phase check) for Hall-equipped joints; drift check on repeat sweep |
| 1 | Per-joint cal state machine in `ActuatorManager::calibrateJoint`; continuous runtime `ERR` stream on motion-without-sensor-response |
| 0.5 | EEPROM `JointCal` struct in the shared `eeprom_layout.h` (Task 1); SDK + CLI entry points |

**Each motor has exactly one position sensor**, either an internal **potentiometer** (linear actuators in this category) or a **Hall encoder** (the planetary-gear yaw motors and any linear-actuator variant with built-in Hall). The shield wiring accommodates both options per slot (Rev 3 board pinout, see `board_pins.h`), but the motor cable only carries one. Calibration auto-detects which sensor is wired during the first commanded nudge and records that sensor type in EEPROM alongside the min/max.

Goal: Build the single-joint calibration function the whole-robot sequence in Task 3 will call repeatedly. For each joint it: (1) auto-detects whether the motor's sensor is a pot or a Hall encoder, (2) records min/max stops on whichever sensor is wired, (3) detects when the sensor reading is reversed relative to drive direction and saves a `sensorReversed` flag, (4) normalizes position to a **float in [0.0, 1.0]** matching what the locomotion model consumes, (5) persists all of it per-joint to EEPROM, and (6) emits an `ERR <joint> <reason>` line on serial **any time** the motor is being driven but the sensor isn't following — during cal or normal operation.

Outputs
- A single per-joint calibration entry point on the MCU (callable on any one of the 18 joints) that sweeps both directions, auto-detects sensor type, records min/max + flip, and writes per-joint cal to EEPROM.
- A continuous **`ERR <joint> <reason>` runtime stream** emitted whenever motion is commanded but the sensor doesn't follow (any time, not just during cal). MCU SDK captures and surfaces these.
- Per-joint `sensor_type` (POT or HALL) and `sensorReversed` plumbing in `LinearActuator` with `getRawPos()` doing the right thing for whichever sensor is wired.
- Hall signed-count quadrature decoding for Hall-equipped joints; existing pot path stays as-is for pot-equipped joints.
- `JointCal` field added to the shared `EepromLayout` struct in `firmware/arduino/eeprom_layout.h` (Task 1) — single `EEPROM.put`/`EEPROM.get` for everything.
- SDK + GUI display normalized `[0.0, 1.0]` joint position as the canonical value.

Acceptance Criteria
- **2a** — A single per-joint calibration function (`calibrateJoint(jointIndex)` in `ActuatorManager` or equivalent) sweeps the joint to both end-stops and records `sensor_type` (auto-detected: POT or HALL), `sensor_min`, `sensor_max`, and `sensorReversed` for that joint.
- **2b** — Direction detection: during cal, the function issues a small commanded extension; if the wired sensor's smoothed reading rises, `flip = false`; if it falls, `flip = true`. The flag is saved to EEPROM and applied on every `getRawPos()` read (`raw_corrected = flip ? (sensor_full_scale − raw) : raw`, where `sensor_full_scale` is 1023 for pots and `(sensor_max − sensor_min)` for Hall).
- **2c** — Hall path (only on joints where `sensor_type == HALL`): the `hall_hw` module is extended to decode A/B quadrature and return a **signed** count that tracks direction, not just edges. `sensor_min` / `sensor_max` are recorded by retracting fully to stop → setting zero, extending fully to stop → setting max. The cal sweeps the joint twice to verify the count is reproducible within a documented tolerance (e.g. ±2 counts).
- **2d** — Cal data persists in EEPROM as a `JointCal` field inside the shared `EepromLayout` struct (Task 1 §4), written via `EEPROM.put` and verified by the struct's magic + CRC on `EEPROM.get`. Survives reboot.
- **2e** — `LinearActuator::getPos()` continues to return a `float` in `[0.0, 1.0]` — verified end to end through `JointTelemetry.pos`, the SDK's `KrabbyMCUSDK.joints`, and the GUI's normalized-position display. Implementation reads from whichever sensor is wired (pot pin or Hall count) and applies flip.
- **2f** — **Continuous runtime sensor-health monitoring.** The firmware emits `ERR <joint> <reason>` on the leader's USB serial **any time** the motor is being driven (PWM nonzero, EN high) but the wired sensor's reading does not change for a configurable timeout (extends the existing `isStalled()` mechanism, `actuator_manager.h` ~L169). This is **not cal-only** — it fires during normal operation too, calibration just incidentally triggers it on broken wiring. Reason codes come from the canonical vocabulary in Task 1 §5: `motor_did_not_move`, `motor_jammed`, `pot_value_invalid`, `hall_no_edges`, `hall_drift` (cal-time only). Throttle to one `ERR` per stall event (don't spam every loop iteration). The MCU SDK's `_reader_loop` (`firmware/krabby_mcu.py` ~L135–141) parses `ERR ` lines per Task 1 §5; the GUI prints a human-readable message via the SDK's `_FIX_INSTRUCTIONS` table (Task 4 §5).
- **2g** — A SDK call (`KrabbyMCUSDK.calibrate_joint(name)`) and a CLI subcommand (`krabby-firmware calibrate-joint <name>`) trigger the per-joint cal end-to-end. There are no progress lines on the wire — the SDK just sends the command and watches the existing joint-telemetry stream plus any `ERR <joint> <code>` lines that fire. When the operator wants to see the recorded values, they issue `GET calibration` (Task 4 §4) — the only way the SDK learns what landed in EEPROM is via that retrieval, not via a streaming completion message.
- **2h** — **Self-heal on first end-stop contact.** After a reboot (with `JointCal` already in EEPROM), each joint comes up in a `PARTIALLY_CALIBRATED` state — the firmware knows `sensor_max − sensor_min` (the full range) but not its current absolute position on that scale. During any motion, when the joint hits a hard end-stop — detected as PWM applied + position not changing + `avgIS ≈ 0` (the current-aware `isStalled()` split, see [OVERVIEW FAQ](OVERVIEW.md)) — the sensor reading snaps to `sensor_max` (extend stall) or `sensor_min` (retract stall), and the joint transitions to `FULLY_CALIBRATED`. Anchoring one end implicitly fixes the other from the known span. **A high-current stall is _not_ an end-stop** — that's a `motor_jammed` ERR per §2f, not a self-heal anchor. No operator action; no boot-time homing dance; no live position in EEPROM. `calibration_state` is queryable per joint via Task 1's `GET` command.

---

## 1. What's already there to build on

`firmware/arduino/actuator_manager.h`:
- `LinearActuator` already has `minStop` / `maxStop` (~L37–38), `getRawPos()` (~L96), and `getPos()` (~L88–94) which returns `(rawPot - minStop) / (maxStop - minStop)` clamped — so the float `[0.0, 1.0]` output path already exists; this task verifies it end-to-end and extends `minStop`/`maxStop` with Hall and flip.
- `ControlConfig.alphaPot = 0.15` smooths the pot reading (~L18); `updateSensors()` (~L77–85) writes `avgPot`. We use `avgPot` for direction detection in §3.
- `ActuatorManager::startAutoCalibration()` (~L364) and `updateCalibration()` (~L371) implement a hardcoded multi-joint sequence with the `CalState` enum (~L329). Task 3 will rework this; **Task 2 should add a single-joint state machine** that the multi-joint sequence calls in Task 3.
- `CalData` (~L356) saves `minVals[6]` / `maxVals[6]` to EEPROM at addr 0 with magic `0xDEADBEEF`. We extend this schema (§4).
- `LinearActuator::isStalled()` (~L169) detects end-of-travel by position-doesn't-change-while-driving. Reuse for end-stop detection.

`firmware/arduino/hall_hw.h` / `hall_hw.cpp`:
- Today: `hallHwGetEdgeCount(slot)` returns a uint32 cumulative count, edges only (~L11). No direction.
- Rev 3 wires Hall A on D50–D52 (PCINT0) and Hall B on A12–A14 (PCINT2). The B channel is read but isn't currently combined into a quadrature signal. Task 2 needs to add a signed-count function.

`firmware/krabby_mcu.py` / `firmware/gui/app.py`:
- `JointTelemetry.pos` (a `float`) is the canonical position the SDK and GUI surface. Task 2 confirms every consumer uses this, not raw ADC.

## 2. The per-joint calibration sequence

Conceptually a small state machine, one joint at a time:

```
IDLE
  → DRIVE_NUDGE         (small commanded extension, ~30 ramped PWM, 250 ms)
  → DETECT_DIRECTION    (compare avgPot before/after; set sensorReversed)
  → STOP                (briefly hold)
  → DRIVE_RETRACT       (manualDrive(-150) until isStalled(250))
  → RECORD_MIN          (potMin = getRawPos(); hallMin = signedHallCount(slot))
  → DRIVE_EXTEND        (manualDrive(+150) until isStalled(250))
  → RECORD_MAX          (potMax, hallMax)
  → DRIVE_RETRACT_AGAIN (repeat for Hall drift check)
  → CHECK_REPEAT        (|hallMin_2 - hallMin_1| within tolerance?)
  → SAVE_EEPROM
  → DONE / FAIL <reason>
```

Reuses `manualDrive()` and `isStalled()` from `LinearActuator` (~L115, L169). The "small nudge" before the full sweep is the direction-detection step (§3) — do this first so the rest of the sweep can use the flip-corrected value.

Wrap it in `ActuatorManager::calibrateJoint(uint8_t jointIndex, Direction direction = NONE)` that drives one actuator's state machine to completion while leaving the other 17 joints idle (no PID, no PWM). The `direction` argument is what Task 3's whole-robot sequence uses to drive one end at a time:

- `NONE` (default) — **fully calibrate**: sweep to retract stop, then extend stop, recording both end-stops + sensorReversed + sensor_type. Parks wherever the second sweep ended (could be either stop). Used when an operator runs `calibrate-joint <name>` by itself.
- For **linear joints** (hip-pitch, knee): `EXTEND` or `RETRACT`. Drives that one direction until stalled at the end-stop, records that one end-stop, **parks the joint at that end-stop** (no return-to-center). The orthogonal stop is recorded the next time `calibrateJoint` runs against the same joint with the other direction (or has already been recorded if the no-direction sweep was used earlier).
- For **yaw joints**: `LEFT` or `RIGHT`. Same shape — drive to that one yaw stop, record, park there.

Wire-format from the SDK matches: `K <name>` for full sweep, `K <name> extend` / `K <name> retract` for linear directional, `K <name> left` / `K <name> right` for yaw directional. Sending an `extend` direction to a yaw joint (or `left` to a linear) is a programming error; the firmware silently drops it (no `ERR` wire reply — the SDK validates client-side per Task 1's pattern).

The existing `updateAll()` already has the `if (calState != CAL_IDLE) updateCalibration()` branch (~L274–285); the new single-joint logic plugs in there.

## 3. Pre-test nudge: detect sensor type, direction, and motor health in one step

Every calibration (and every per-joint cal in Task 3) starts with a **bidirectional pre-test** that simultaneously verifies the motor moves at all, auto-detects whether the wired sensor is a pot or a Hall, and detects the direction-flip — robust to the motor starting at either end-stop.

```
1. Read both sensor channels before driving (let them settle ~50 ms after EN goes high):
   pot_before  = avgPot
   hall_before = signedHallCount(slot)

2. Drive +30 PWM (extend direction) for ~250 ms.
3. Read after: pot_after, hall_after. Compute:
   pot_delta  = pot_after  - pot_before
   hall_delta = hall_after - hall_before

4. If |pot_delta| > NUDGE_THRESHOLD (e.g. 20 raw ADC counts):
     sensor_type    = POT
     sensorReversed = (pot_delta < 0)        # extend drove sensor down → reversed
     ✓ motor moves; pre-test passes.

5. Elif |hall_delta| > HALL_NUDGE_THRESHOLD (e.g. 4 counts):
     sensor_type    = HALL
     sensorReversed = (hall_delta < 0)
     ✓ motor moves; pre-test passes.

6. Else (neither sensor changed — motor may be at the extend end-stop, or broken):
   - Drive −30 PWM (retract direction) for ~250 ms.
   - Read after, recompute pot_delta / hall_delta against the original "before" values.

7. Repeat steps 4–5 with the opposite sign convention:
   - |pot_delta| > NUDGE_THRESHOLD  → sensor_type = POT;  sensorReversed = (pot_delta > 0)
   - |hall_delta| > HALL_THRESHOLD → sensor_type = HALL; sensorReversed = (hall_delta > 0)

8. If still no movement in either direction:
     emit ERR <joint> motor_did_not_move  ← motor truly didn't respond
     abort calibration for this joint.
```

This shape handles the "motor starts at an end-stop" case naturally: if the joint is parked at extend, step 2's extend drive can't move it further, step 6 tries retract instead, and step 7 catches the movement (with the inverted sign convention for the retract direction). Only if **both** directions fail to move the sensor — a real wiring/power problem — does the ERR fire.

The flag goes into the per-joint EEPROM entry. Application is centralized: change `LinearActuator::getRawPos()` (~L96) to read whichever sensor is wired and apply the flip:

```cpp
int32_t getRawPos() {
    int32_t raw = (cal.sensor_type == HALL) ? signedHallCount(slot) : (int32_t)avgPot;
    return cal.sensorReversed ? (full_scale_for(cal.sensor_type) - raw) : raw;
}
```

where `full_scale_for(POT) = 1023` and `full_scale_for(HALL) = cal.sensor_max - cal.sensor_min` (so the flip math works on the count range). Every consumer (`getPos()`, `printTelemetry()`, `isStalled()`) sees the corrected value. **Do not** rewrite `avgPot` or the underlying Hall count itself — keep the raw smoothed sensor values inspectable and apply the flip at the read site.

## 4. EEPROM schema extension

Existing layout (from `arduino.ino` ~L25–27 and `actuator_manager.h` `CalData` at addr 0):

| Addr | Field |
|---|---|
| 0–25 | `CalData { minVals[6], maxVals[6], magic 0xDEADBEEF }` |
| 32–33 | Role magic `0xAB` + role byte |
| 34 | Config schema version (Task 1) |
| 35–50 | Per-board serial number (Task 1) |

Extend per-joint cal at a new region (e.g. addr 64+) with a versioned struct:

```cpp
struct JointCal {
    uint16_t potMin;       // raw ADC at retract-stop (pre-flip)
    uint16_t potMax;       // raw ADC at extend-stop  (pre-flip)
    int32_t  hallMin;      // signed quadrature count at retract-stop
    int32_t  hallMax;      // signed quadrature count at extend-stop
    uint8_t  sensorReversed;      // 1 = invert before normalize
    uint8_t  hasHall;      // 1 if Hall is wired on this slot (Rev 3 only)
    uint8_t  reserved[6];  // pad to align
};

struct JointCalBlock {
    uint16_t magic;        // 0xCA17  (cal-17, M17)
    uint8_t  schema_ver;   // 1
    uint8_t  joint_count;  // 6 (this board's slots)
    JointCal joints[6];
    uint32_t crc32;        // simple Arduino CRC over the prior bytes
};
```

(Exact bytes are sketch; size carefully against Mega's 4 KB EEPROM and document the address map in a header comment block alongside `arduino.ino`'s existing map. Pick a base address that does not collide with Task 1's role/serial/config region or any future M16 IMU/battery cal region.)

`saveJointCal()` / `loadJointCal()` follow the `EEPROM.put` / `EEPROM.get` pattern of the existing `saveCalibration` / `loadCalibration` (~L518–548). The legacy `CalData` block stays intact so existing flashed boards aren't bricked — `loadJointCal()` returns `false` if magic mismatches and the firmware falls back to the legacy values until a re-cal runs.

## 5. Hall quadrature (signed direction)

The current `hall_hw.cpp` increments an edge count on any PCINT trigger. To get direction, on each A-channel edge interrupt, sample the B channel level (or vice versa) and decide direction by phase. For a standard rising-A interrupt:
- B low → +1 (forward / extend)
- B high → −1 (backward / retract)

A reference implementation pattern (Greg Reiter and many Arduino encoder libs use it):

```cpp
ISR(PCINT0_vect) {           // Port B change — Hall A channels D50/D51/D52
    for (slot 0..2) {
        if (rising_edge_on_A) {
            if (digitalRead(B_pin)) signedCount[slot]--;
            else                    signedCount[slot]++;
        }
    }
}
```

(Production code should read the port register directly rather than `digitalRead` for ISR speed; see `hall_hw.cpp` for the existing port-register pattern.) Add:

```cpp
int32_t hallHwGetSignedCount(uint8_t hallSlot);
void    hallHwResetCount(uint8_t hallSlot);
```

`hallHwGetEdgeCount` stays for legacy telemetry. `LinearActuator` gets a `int32_t hallCountSignedAtBoot` and a getter that returns delta from boot, so `JointCal.hallMin/hallMax` represent end-stop positions in stable counts.

**Drift verification (2c):** the cal state machine sweeps min→max→min twice. Record `hallMin_1` on the first retract, `hallMin_2` on the second; if `|hallMin_2 - hallMin_1| > HALL_DRIFT_TOLERANCE` (e.g. 4 counts), fail with `hall_drift`.

For joints **without** a Hall sensor (Rev 2 boards entirely; Rev 3 slots 3–5 if no Hall is wired), set `hasHall = 0` and skip the Hall path entirely — pot-only cal is sufficient.

## 6. Normalized [0.0, 1.0] is the canonical value

The **GUI and SDK only ever see the normalized `[0.0, 1.0]` joint position** — the raw Hall edge count is a firmware-internal value used for fine motion tracking, not a thing operators look at. Confirm every downstream consumer agrees:

- `LinearActuator::getPos()` (~L88–94) returns `float` in `[0.0, 1.0]`. ✓
- `LinearActuator::printTelemetry()` (~L196–219) writes `getPos()` with 3 decimals as the first numeric token. ✓
- `firmware/interfaces/joint_telemetry.py` `JointTelemetry.pos` (~L13) — `float`. ✓
- `firmware/krabby_mcu.py` — stores `JointTelemetry` in `self.joints`. ✓
- `firmware/gui/app.py` — `JointRow.update_from_telemetry` (~L59) currently shows `pot` (raw ADC) and `Hall` (raw cumulative edge count). **Rework the GUI so the displayed position column is `jt.pos` (the normalized `[0.0, 1.0]` value)**, and **drop the raw Hall edge-count column** — that count is an internal firmware quantity, not an operator-facing one. Raw pot ADC can stay as a debug field (it's useful for spotting wiring issues), but the canonical position the user reads is `jt.pos`.
- `hal/server/jetson/krabby_mcusdk.py` `apply_command` (~L105) currently converts radians → PWM jog (the TODO at ~L122–124 notes "pot is not connected for all joints, we do not call `send_command_joints` for now"). **Fixing this is part of Task 2, not a downstream milestone.** Once per-joint cal exists for all 18 joints, swap `apply_command` to use `send_command_joints` — pass normalized `[0.0, 1.0]` targets (the calibrated `getPos()` scale) instead of PWM jog. Concretely: delete the `_jog_all_joints(cmd_dict)` call at ~L124 and use the `cmds_by_fw` dict (built one line above by `_map_mcu_joints_to_normalized`, already in `[0.0, 1.0]`) via `self._mcu.send_command_joints(cmds_by_fw)`. Remove the TODO comment. Verify on the bench: target a joint to 0.5 via `send_command_joints` and confirm it drives to mid-travel (not just runs at constant PWM until the operator releases).

The firmware keeps `hallHwGetSignedCount(slot)` and `LinearActuator::getRawPos()` available for internal use (velocity computation, fine motion tracking, drift diagnostics) — those just aren't surfaced to the operator-facing telemetry path.

## 6.5 Boot state depends on which sensor the joint has

A joint's position needs to be **absolutely known** before the model trusts it. There are two ways to get there: (a) write live position to EEPROM and trust it across reboots, or (b) re-acquire absolute position from physical references on every boot. **Krabby uses (b)**, for the reasons every production quadruped does:

- **Atmega 2560 EEPROM is rated for ~100,000 writes per cell.** Writing live position even once per second exhausts a cell in ~28 hours.
- **Reboots are unreliable about position.** A brown-out, a watchdog reset, or a power glitch can leave the saved position stale by an unknown amount — the joint can drift / sag / be pushed while the MCU is dark.
- **Saved state being wrong is worse than saved state being absent** — the model thinks it knows the position when it doesn't.

**The boot-state behavior splits by `sensor_type`** because pots and Hall encoders have fundamentally different absolute-position properties:

### Pot-equipped joints — `FULLY_CALIBRATED` immediately on boot

A potentiometer is **absolute by physics**: the wiper voltage at any joint position is the same on every boot, regardless of power-off duration or how the joint moved while dark. The moment EEPROM loads `JointCal` (giving us `sensor_min` / `sensor_max` / `sensorReversed`), the live ADC reading plus those constants yields the absolute normalized position. **No self-heal is needed and no `PARTIALLY_CALIBRATED` state exists for pot joints** — they come up directly in `FULLY_CALIBRATED` and the model can trust the position from the first tick.

### Hall-equipped joints — `PARTIALLY_CALIBRATED` until first end-stop contact

A Hall encoder is **incremental**: it counts edges as the joint moves, and the count is lost on power-off. After EEPROM loads `JointCal`, the firmware knows the **range** (`sensor_max − sensor_min`) but not the current absolute position on that scale. Hall joints come up in `PARTIALLY_CALIBRATED` and need a physical anchor before they're trustworthy.

**Self-heal on first end-stop contact** is the anchor. The joint is allowed to move in `PARTIALLY_CALIBRATED` state — the firmware tracks the signed Hall count incrementally from boot, and reports an *un-anchored* relative position (the model can treat it as "rough" or just ignore until calibrated, the SDK exposes the state). The firmware watches for an end-stop hit during any motion: when the current-aware `isStalled()` (see [OVERVIEW FAQ](OVERVIEW.md)) fires with `avgIS ≈ 0` (end stop, not a jam):

- `currentPwm > 0` (commanded extend) → snap signed Hall count to `sensor_max`. Joint state flips to `FULLY_CALIBRATED`.
- `currentPwm < 0` (commanded retract) → snap signed Hall count to `sensor_min`. Same.

Anchoring one end implicitly fixes the other from the known span. Hall joints calibrate themselves during the first motion that drives them to either extreme — no operator action, no boot-time homing dance. From `FULLY_CALIBRATED` onward, Hall is the canonical position source for the joint (high resolution, drift-free as long as the encoder doesn't miss counts).

### Summary

| `sensor_type` | Boot state | Source of position once `FULLY_CALIBRATED` | Self-heal mechanism |
|---|---|---|---|
| `POT` | `FULLY_CALIBRATED` on boot | live ADC scaled by `sensor_min`/`sensor_max` (+ `sensorReversed`) | not needed; pot is inherently absolute |
| `HALL` | `PARTIALLY_CALIBRATED` on boot | signed Hall count anchored to a known end-stop | first end-stop contact (current-aware `isStalled` with `avgIS ≈ 0`) snaps count to `sensor_min` or `sensor_max` |

### Position commands on `PARTIALLY_CALIBRATED` joints are rejected with an ERR

A position target only makes sense if the firmware knows the joint's absolute position. A `PARTIALLY_CALIBRATED` Hall joint doesn't — the count is unanchored. So the firmware **rejects any non-jog command targeting that joint** and emits `ERR <joint> not_calibrated` on serial (shared vocabulary with Task 4):

- **Position targets (`T <name> <val>` via `send_command_joints`)** on a `PARTIALLY_CALIBRATED` joint → drop the command for that joint, emit `ERR <joint> not_calibrated`. Other joints in the same `T` batch that *are* calibrated still apply normally.
- **Jogs (`J` / `B`)** are **always allowed**, even on `PARTIALLY_CALIBRATED` joints, because they're direct PWM drive and don't depend on absolute position. This is how the operator drives a Hall joint to an end stop to trigger the self-heal in the first place.
- **Hold (`H`)** is also always allowed — it de-energizes the joint, no position needed.
- The `ERR` is throttled to one per joint per command (don't spam the line for every `T` while the joint stays uncalibrated).

The SDK enforces the same rule client-side as a courtesy (skip uncalibrated joints in `send_command_joints` and log a single warning), but the firmware-side rejection is the authoritative gate — the wire ERR is the source of truth.

### Exposing state

Add a `calibration_state` field per joint, queryable via Task 1's `GET calibration_state` and surfaceable as an optional GUI column. For pot joints this always reads `FULLY_CALIBRATED` after boot; for Hall joints it flips on first end-stop contact. Useful for debugging — "why is the yaw noisy?" → "still PARTIALLY_CALIBRATED, hasn't hit a stop since reboot."

### Hip-yaw on the V0.2 crank-and-slot is out of scope

The continuously-rotating crank-and-slot mechanism (M12) has no true mechanical end stops on the yaw axis, so the Hall self-heal above doesn't apply once that mechanism is in place. See the [OVERVIEW FAQ](OVERVIEW.md) for the planned single-magnet "forward sensor" approach in a future milestone. For M17, the yaw motors are operated in a configuration that **does** have hard end stops (Task 0's bench mounting), so they fall under the Hall self-heal mechanism above just like any other Hall-equipped joint.

**What lives in EEPROM:**

| Data | Where | Write frequency |
|---|---|---|
| Joint cal constants (`JointCalBlock`: min/max/flip) | Task 2 — addr 64+ | Once per `calibrate-joint` or `calibrate-all` run; effectively never |
| Per-board serial number | Task 1 / Task 3 — addr 35–50 | Once per `calibrate-all` run |
| Board role | Task 1 — addr 32–33 | Once per `SET role …` |
| **Live joint position** | **NEVER** | — |
| **Live Hall count** | **NEVER** | — |

EEPROM wear is therefore a non-issue for M17 — there's a handful of writes over the device's lifetime, all driven by explicit operator commands.

## 7. SDK and CLI entry points (2g)

- `KrabbyMCUSDK.calibrate_joint(name: str)` — writes a new single-line command (e.g. `K <name>`) the firmware parses to invoke `calibrateJoint(jointIndex)`. The firmware emits no completion reply — the SDK watches for any `ERR <joint> <code>` lines that fire (Task 1 §5) and, once the firmware has gone idle again, reads the recorded cal back via `GET calibration` (Task 4 §4).
- `krabby-firmware calibrate-joint <name>` in `firmware/cli.py` — wraps the SDK call so an operator can run a single-joint cal from a shell.
- The existing whole-robot `C` command (~L347–354) stays for now; Task 3 replaces its implementation with a series of per-joint cals.

## 8. Verify
- **Pot flip:** intentionally swap a pot's leads on one slot, run `calibrate-joint`, confirm `sensorReversed = true` is set and `getPos()` returns sensible values afterward.
- **Hall direction:** rotate a Hall-equipped joint by hand back and forth; confirm signed count rises and falls (not just monotonically increases).
- **Persistence:** run cal, power-cycle, read back via a new `CAL GET <name>` reply (mirroring Task 1's `GET` style); values match.
- **End-to-end normalized:** drive a joint to mid-travel; confirm `jt.pos ≈ 0.5` in the GUI and via `KrabbyMCUSDK.joints[name].pos`.
- **Self-heal on reboot:** with `JointCal` already in EEPROM, power-cycle; confirm `GET calibration_state` returns `PARTIALLY_CALIBRATED` on the joint. Jog the joint until it stalls against one stop; confirm the state flips to `FULLY_CALIBRATED` and that subsequent `getPos()` readings are anchored to the known end-stop value. Power-cycle again, confirm `PARTIALLY_CALIBRATED` returns (no live state persisted).

## 9. Deliverable checklist
- [ ] Single per-joint cal function (`calibrateJoint`) implemented in `ActuatorManager`.
- [ ] Pot direction-flip detection; flag applied in `getRawPos()`.
- [ ] Hall signed-count infrastructure (`hallHwGetSignedCount`, ISR with B-phase check); drift check on repeat sweep.
- [ ] EEPROM schema extended with `JointCalBlock` (magic + version + CRC); address map documented.
- [ ] Normalized `[0.0, 1.0]` joint position verified end-to-end (firmware → SDK → GUI).
- [ ] Failure reason codes emitted as a small fixed set (shared with Task 4's schema).
- [ ] `KrabbyMCUSDK.calibrate_joint(name)` + `krabby-firmware calibrate-joint <name>` work from the host.
