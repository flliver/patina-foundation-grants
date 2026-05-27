# Task 2 — Qwiic OLED status display (the customer-facing UI)

Goal: Add the SparkFun Qwiic OLED 1.3" to the primary MCU's I2C bus and render a status screen that is good enough to be the **only customer-facing UI on the robot**. It shows which of the three controllers are alive, each actuator's live state as a stylized krab, IMU attitude, and (after Task 3) battery state. This task also fixes a real firmware bug surfaced by the UI: with a motor unplugged the position/current channels float and the firmware streams **noise** — we add current-based attachment detection and position filtering so a disconnected actuator reads as disconnected, not as a randomly-moving joint.

Outputs
- SparkFun Qwiic OLED 1.3" daisy-chained on the Task 1 Qwiic→Dupont bus, driven from `firmware/arduino/arduino.ino`.
- A stylized krab render matching the front face of the 3D-printed enclosures: body in three controller-thirds, six legs/hips, battery as two stacks on the rear.
- Per-controller detected/active indicators and per-actuator state glyphs.
- `LinearActuator` attachment detection (`isConnected()`) + position-noise filtering in `firmware/arduino/actuator_manager.h`.
- A discrete status LED on a spare GPIO for a true "red" alarm the monochrome panel can't show (reused in Task 4).

Acceptance Criteria
- **2a** — SparkFun Qwiic OLED 1.3" (default I2C **0x3D**) daisy-chained onto the primary Mega's Qwiic→Dupont bus; init failure does not crash the firmware (control continues without the display).
- **2b** — A stylized krab is rendered: body split into three controller thirds (front/left/right), six legs/hips, battery as two stacked bars on the rear (placeholder until Task 3).
- **2c** — For each of the three controllers the display shows **detected/active vs missing**, derived from role election + forwarded-telemetry freshness (not just this board's own role).
- **2d** — Each local actuator shows a state glyph mapping to the intended color code (extend/blue, retract/yellow, holding/green, disconnected/red); since the panel is 1-bit, these are distinct glyphs, not colors.
- **2e** — `LinearActuator` gains current-based attachment detection; a channel with no motor is flagged disconnected.
- **2f** — Position reporting is filtered so a disconnected/floating channel reports invalid or last-valid rather than streaming noise — on the OLED **and** in telemetry.
- **2g** — A discrete status LED on a free GPIO lights/blinks on disconnected-motor (and is wired so Task 4 can reuse it for low-battery).
- **2h** — OLED refresh does not measurably impact gait-loop timing (non-blocking or partial-page updates; verified with timing instrumentation).

---

**NOTE:** Line numbers are pointers to the right function and may drift.

---

## 1. Why monochrome, and what "color" means here

Fletcher's intent is a color code for the motor dots — **blue** extend, **yellow** retract, **green** active-but-not-moving, **red** disconnected/unknown. The chosen display is the **SparkFun Qwiic OLED 1.3" (128×64)**, which is **1-bit monochrome** (SSD1306-class). It cannot show color. So the color *semantics* are kept but rendered as **distinct glyphs**:

| State | Intended color | Glyph |
|---|---|---|
| Extending (+PWM) | blue | outward/up arrow ▲ |
| Retracting (−PWM) | yellow | inward/down arrow ▼ |
| Active, attached, ~0 PWM | green | filled dot ● |
| Disconnected / unknown | red | hollow ○ or ✕ |

To get a true hardware "red" alarm the panel can't produce, **add a discrete LED** (red or bicolor) on a spare GPIO (2g). Confirm a free pin against `firmware/arduino/board_pins.h` for the active `KRABBY_PIN_REV` (the Rev 2 build uses D2–D13, D16–D28, D20/D21 — leaving plenty of digital pins free; do not reuse Hall pins on Rev 1/3). This LED is reused for the low-battery indicator in Task 4.

- Library: **`SparkFun_Qwiic_OLED_Arduino_Library`**, class **`Qwiic1in3OLED`**. Repo: <https://github.com/sparkfun/SparkFun_Qwiic_OLED_Arduino_Library>. Product/hookup: <https://www.sparkfun.com/sparkfun-qwiic-oled-1-3in-128x64.html>.
- **Default I2C address is `0x3D`** (selectable to `0x3C` via the address-jumper). *(The earlier draft of this milestone said 0x3C — that is the jumpered address, not the default.)*

## 2. The krab render

Draw a stylized krab matching the front face of the 3D-printed enclosures (reference the V0.2 enclosure / `hardware/Uno-v0.2/diagrams/` art). Layout on 128×64:

- **Body** split into three regions = the three controllers (front / left / right). Each region is shaded filled vs outline to show that controller **detected/active vs missing** (2c).
- **Six legs + hips** drawn off the body; each leg's hip/knee glyphs (§3) show that actuator's state (2d).
- **Battery** as two stacked bars on the rear ("butt"); placeholder until Task 3 fills them with per-battery level.
- A small text strip (role, IMU roll/pitch, pack V) can sit along an edge.

Keep it legible at 128×64; this is the face a customer sees.

## 3. Driving it from firmware

Render from the leader's `loop()` near the telemetry block (`firmware/arduino/arduino.ino` ~L426). The OLED is leader-only; followers never touch it.

**Per-controller presence (2c).** The leader already learns follower presence during role election — `determineRole()` sets `syncFromLeft`/`syncFromRight` and assigns `leftSerial`/`rightSerial` (~L181–223). After boot, freshness comes from `forwardFullLines()` (~L117) forwarding follower telemetry; track a last-seen timestamp per follower and mark a controller "missing" if its line goes stale (mirror the OLED command-freshness timeout). This is local state on the leader — it is **not** added to the telemetry schema (role-in-telemetry is M14/James).

**Per-actuator state (2d).** For each local actuator, pick the glyph from its drive state. `LinearActuator` already exposes `currentPwm` (sign = direction), and `hasTarget`/`stopMotor()` semantics (`actuator_manager.h` ~L41–44): `+PWM` → ▲, `−PWM` → ▼, attached & ~0 PWM → ●, not attached → ○/✕ (from `isConnected()`, §4).

**Non-blocking (2h).** Don't block the gait loop on a full-frame I2C write. Throttle refresh (e.g. 5–10 Hz) and/or use partial-page updates; verify loop timing is unchanged.

## 4. Fix the disconnected-motor noise (real bug)

With a motor/actuator unplugged, the current-sense pin (`pinIS`, analog A6–A11) and pot pin (`pinPot`, A0–A5) float, and the firmware streams the floating ADC as a real position. Two changes in `firmware/arduino/actuator_manager.h` `LinearActuator`:

1. **Attachment detection via current sense (2e).** The class already keeps a smoothed current `avgIS` (updated in `updateSensors()` ~L77–85, smoothing `alphaIS=0.10`). Add an `isConnected()` that uses the current-sense signature to decide whether a motor is actually wired: a connected channel draws/settles to a characteristic current when its EN line is driven; a floating channel does not. This is **how the UI and telemetry know a motor is present**. (Note `driveActuator()` ~L224 toggles `pinEn`; sample current while enabled.)
2. **Position-noise filtering (2f).** When `isConnected()` is false (or the pot reading is outside a sane band / clearly floating), stop reporting the live noisy value. Report position as invalid (a sentinel) or hold the last valid filtered reading, in both:
   - `LinearActuator::printTelemetry()` (~L196–219) — the per-joint telemetry segment that goes to the Orin, and
   - the OLED glyph (render ○/✕, not a random ▲/▼).

Reflect "disconnected" on the discrete LED (2g). The Python side (`firmware/interfaces/joint_telemetry.py`, `firmware/gui/app.py`) already carries `current`/`pos`; if you add an explicit connected flag to the wire format, extend the parser the same append-only way as the Task 1 `IMU` segment (don't break the fixed 9-token joint segment).

## 5. Verify
- **Display up (2a/2b):** OLED renders the krab; pull its Qwiic plug and confirm the firmware keeps running (init/refresh failure handled).
- **Presence (2c):** power a follower off and confirm its body-third flips to "missing" within the staleness timeout; power it back and confirm it recovers.
- **Glyphs (2d):** jog an actuator extend/retract and watch ▲/▼; release and watch ● (if attached).
- **Disconnect (2e/2f/2g):** unplug one actuator; its glyph goes ○/✕, the red LED lights, and its telemetry position stops streaming noise.
- **Timing (2h):** loop timing unchanged with the OLED active vs. removed.

## 6. Deliverable checklist
- [ ] Qwiic OLED 1.3" on the leader bus at 0x3D; init/refresh failure non-fatal.
- [ ] Stylized krab rendered: 3 controller-thirds, 6 legs/hips, 2 rear battery bars (placeholder).
- [ ] Per-controller detected/active vs missing, from role election + telemetry freshness.
- [ ] Per-actuator state glyphs mapping to blue/yellow/green/red semantics.
- [ ] `LinearActuator::isConnected()` added (current-based); disconnected channels flagged.
- [ ] Position filtered so disconnected/floating channels don't stream noise (telemetry + OLED).
- [ ] Discrete status LED on a free GPIO for the red alarm (reused in Task 4); free pin confirmed against board_pins.h.
- [ ] OLED refresh non-blocking; loop timing unchanged.
