# Task 3 — Octopus power bus, shunt, and INA228 battery monitoring

Goal: Instrument the robot's main power bus with two Adafruit INA228 power monitors so the firmware knows total pack voltage/current and each battery's voltage. This replaces dead-reckoning on battery state and feeds the protective shutdown in Task 4. It is real high-amperage DC work — the safety section is not optional.

Outputs
- Two INA228 boards on the Task 1/2 Qwiic→Dupont bus: a Pack monitor across an external shunt, and a Midpoint monitor for per-battery voltage.
- A `BATT` telemetry frame appended to the leader's line; SDK/GUI/OLED populated.
- Battery constants (addresses, shunt value, divergence threshold, poll cadence) added to `firmware/arduino/sensors_config.h`.
- A wiring diagram and assembled-harness photos committed to `krabby-research/assets/`.

Acceptance Criteria
- **3a** — The "octopus" is documented as the robot's main 24 V power bus; the bench-octopus fallback (batteries on a table, shunt wired directly to the bus) is documented for when M12 is incomplete.
- **3b** — Battery-safety section present and followed; the **150 A fuse/breaker is the first element on Pack+**, closest to the battery positive terminal.
- **3c** — 200 A / 75 mV external shunt installed inline downstream of the fuse, Kelvin sense to the **Pack INA228** (`0x40`); VBUS on the load side.
- **3d** — **Midpoint INA228** (`0x41`, A0 jumper strapped) on the pack midpoint, current channel grounded; per-battery voltages computed (`battery_b = pack − battery_a`).
- **3e** — Both INA228 onboard 15 mΩ shunts disabled (trace cut per the Adafruit guide); trace-cut photo committed.
- **3f** — Firmware calibrates the Pack INA228 with `setShunt(0.000375, 200.0)` and reads bus voltage, current, power, and charge/energy; reads Midpoint bus voltage.
- **3g** — A `BATT` telemetry frame (schema in §4) carries pack V/I/P/charge, battery A & B voltage, a divergence flag, and a power_state byte; parsed by the SDK and shown on GUI + OLED battery bars.
- **3h** — All addresses, shunt value/rating, divergence threshold (default 0.5 V), and poll cadence are hardcoded in `sensors_config.h`.
- **3i** — Battery-side calibration (shunt constant + per-board VBUS offset trim) captured, persisted in EEPROM, and documented.
- **3j** — Wiring diagram + assembled-harness photos committed to `krabby-research/assets/`.

---

## 1. What the "octopus" is

The **octopus is the robot's main power bus** — the high-current harness built in Milestone 12 that ties the two series-connected 12 V LiFePO4 batteries into one 24 V rail and fans it out to all six H-bridge boards. Per the M12 build (`grants/Krabby-Uno/Milestone12-V02-Build/OVERVIEW.md`, "Electronics & Power"): 2× 12 V 100 Ah LiFePO4 in series → 24 V, a **150 A inline circuit breaker on the + rail**, a marine-style bus-bar distribution block ([Amazon B0FL28CCC9](https://www.amazon.com/dp/B0FL28CCC9)), 2 AWG battery leads / 10 AWG to H-bridges. The octopus is the single place to measure total pack current and voltage.

**If M12 isn't finished yet, you don't need the whole robot.** Build a **bench octopus**: two 12 V LiFePO4 batteries on a table, wired in series, with the shunt and INA228 sense taps wired directly into the bus exactly as they will sit on the robot. The measurements, calibration, and firmware are identical; only the chassis is missing.

## 2. ⚠️ Battery safety — read before wiring

A 24 V / 100 Ah LiFePO4 pack delivers hundreds of amps into a short and will vaporize tools, wire, and skin.

- **Fuse first.** Install the **150 A ANL fuse / breaker on Pack+ as the closest element to the battery positive terminal**, before the shunt and everything else. Make/break all other connections with the fuse pulled or breaker open.
- **Insulated tools only**; never let a tool bridge Pack+ to Pack− or to chassis.
- **Verify polarity** before energizing; INA228 IN+/IN− and VBUS orientation matter.
- The **midpoint tap is live 12–14 V** — treat it like a battery terminal, fuse/limit the sense lead.
- Eye protection; fire-safe surface under bench batteries; don't do the first energize alone.

## 3. Topology and INA228 wiring

Nodes brought out of the octopus:

- **Pack+** → 150 A fuse → **200 A / 75 mV external shunt** → bus-bar / load.
- **Pack midpoint** (between the two batteries) → small sense connector.
- **Pack−** → distribution ground.

Two **Adafruit INA228** boards (repo <https://github.com/adafruit/Adafruit_INA228>, header <https://github.com/adafruit/Adafruit_INA228/blob/main/Adafruit_INA228.h>, guide <https://learn.adafruit.com/adafruit-ina228-i2c-power-monitor/arduino>):

- **Pack INA228** — I2C `0x40` (default). IN+/IN− on the external shunt's Kelvin sense terminals; VBUS on the load side of the shunt. Measures total pack voltage and current.
- **Midpoint INA228** — I2C `0x41` (A0 solder jumper strapped). IN+ and IN− tied together (current channel unused); VBUS on the pack midpoint. Measures lower-battery voltage directly; upper battery = `pack_voltage − midpoint_voltage`.
- **Disable both onboard 15 mΩ shunts** (cut the trace on the back per the Adafruit guide) so only the external 200 A/75 mV shunt carries pack current. Commit a photo of the cut (3e).

Both daisy-chain onto the same Qwiic→Dupont bus as the BMI270 (Task 1) and OLED (Task 2).

```
 BATTERY 1 (12V) ──┬── Pack+ ──[150A FUSE]──[200A/75mV SHUNT]──┬── BUS BAR (+24V) ──► 6× H-bridge + Orin DC-DC
      (4S LiFePO4) │                         Kelvin sense ─► Pack INA228 (0x40) IN+/IN−, VBUS=load side
                   │
              midpoint ●───────────────────► Midpoint INA228 (0x41) VBUS  (IN+=IN−, grounded)
                   │
 BATTERY 2 (12V) ──┴── Pack− ─────────────────────────────────┴── BUS BAR (GND)
      (4S LiFePO4)
   I2C chain: BMI270 0x68 → OLED 0x3D → Pack INA228 0x40 → Midpoint INA228 0x41   (D20/D21, 100 kHz)
```

## 4. Firmware

API (from the Adafruit examples; default-board example uses `setShunt(0.015, 10.0)`):

```cpp
#include "Adafruit_INA228.h"
Adafruit_INA228 pack, mid;
pack.begin(0x40, &Wire);
mid.begin(0x41, &Wire);
pack.setShunt(0.000375, 200.0);     // external 75 mV / 200 A shunt
// pack.readBusVoltage(); pack.readCurrent(); pack.readPower();  (+ energy/charge readers — see Adafruit_INA228.h)
// mid.readBusVoltage();   // battery A; battery B = pack - A
```

- Confirm the library's return units (mV/mA/mW vs V/A/W) against `Adafruit_INA228.h` and convert to SI for telemetry.
- **Poll cadence:** read both INA228s on the existing telemetry tick (`TELEMETRY_INTERVAL_MS` in `sensors_config.h`, currently 20 Hz). No separate slow path during normal operation (the low-power slow path is Task 4).
- Compute `battery_divergence_flag = |Va − Vb| > divergence_threshold` (default 0.5 V).
- Append the `BATT` frame to the **leader's** telemetry line (leader-only), the same append-only way as the Task 1 `IMU` segment, and populate the OLED battery bars (Task 2 §2):

```
;BATT <pack_v> <pack_i> <pack_w> <pack_charge> <batt_a_v> <batt_b_v> <divergence_flag> <power_state>
```

| field | unit | source |
|---|---|---|
| pack_v / pack_i / pack_w | V / A (signed) / W | Pack INA228 |
| pack_charge | C (accumulated) | Pack INA228 charge/energy register |
| batt_a_v | V | Midpoint INA228 VBUS |
| batt_b_v | V | pack_v − batt_a_v |
| divergence_flag | 0/1 | `|Va−Vb| > 0.5 V` |
| power_state | enum | 0=normal 1=warn 2=soft_cut 3=hard_cut 4=over_volt 5=sleep 6=resuming (Task 4) |

Parser/SDK/GUI: add a `@dataclass BatteryTelemetry` to `firmware/interfaces/joint_telemetry.py`, store it on `KrabbyMCUSDK` (`firmware/krabby_mcu.py` `_parse_joint_line`), and show pack/per-battery values in `firmware/gui/app.py` and the compact debug log.

## 5. Configuration & calibration

- **`sensors_config.h` (3h):** add INA228 addresses (`0x40`/`0x41`), shunt value (`0.000375`) and rating (`200.0`), divergence threshold (`0.5`), and the poll cadence. This header is the single contract for ports/addresses/rates.
- **Calibration (3i):** capture the shunt calibration constant and a per-board VBUS offset/gain trim so the two INA228s agree at a known reference voltage and current. Persist in EEPROM in the IMU-adjacent region (byte 40+, magic + `schema_version`, alongside Task 1's block). Document the procedure (apply known V / known current, record offsets).

## 6. Verify
- **Pack accuracy:** compare `pack_v` to a calibrated DMM across the pack; compare `pack_i` to a clamp meter under a known load.
- **Per-battery:** confirm `batt_a + batt_b ≈ pack` and both track a DMM on each battery; force a small divergence and confirm `divergence_flag` trips at 0.5 V.
- **End-to-end:** `BATT` frame appears in telemetry, parses, and the OLED battery bars move with pack state.

## 7. Deliverable checklist
- [ ] Octopus defined as the main bus; bench-octopus fallback documented; battery-safety section followed.
- [ ] 150 A fuse first on Pack+; 200 A/75 mV shunt inline; Kelvin sense to Pack INA228 (0x40).
- [ ] Midpoint INA228 (0x41) on midpoint, current channel grounded; per-battery voltages computed.
- [ ] Both onboard shunts disabled (trace-cut photo committed).
- [ ] `setShunt(0.000375, 200.0)`; pack V/I/P/charge + midpoint V read in firmware.
- [ ] `BATT` frame appended, parsed by SDK, shown on GUI + OLED battery bars.
- [ ] Addresses/shunt/divergence/cadence in `sensors_config.h`; calibration persisted to EEPROM + documented.
- [ ] Wiring diagram + assembled-harness photos committed to `krabby-research/assets/`.
