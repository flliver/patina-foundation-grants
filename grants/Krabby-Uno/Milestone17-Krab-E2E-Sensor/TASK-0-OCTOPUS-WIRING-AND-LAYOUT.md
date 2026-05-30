# Task 0 — Octopus wiring, layout, and physical pre-flight

**Time estimate: ~0.5 day (range 0.5–1).** Ideally a single afternoon; reserve a day in case re-crimps or hardware issues surface.

Goal: Lay everything out on a single base board for bench testing — 3× Arduino Mega + V0.3 shields, 6× H-bridge boards, the octopus power harness, and all 18 motors (12 linear actuators + 6 hip-yaw) — with motors labeled, wires inspected/re-crimped as needed, and a manual controller smoke-test confirming every motor moves before any firmware/SDK work begins. The point is to surface every hardware issue **now** so Tasks 1–6 are debugging software, not chasing intermittent connectors.

Outputs
- A single base board (plywood / aluminum extrusion / whatever's at hand) with all 3 Megas + shields, 6 H-bridges, octopus harness, and 18 motors physically arranged for testing and easy transfer to the chassis later.
- Every motor labeled with its intended joint (`FLHY`, `FLHL`, `FLKL`, `FRHY`, …, `MRKL` — see `firmware/krabby_mcu.py` `JOINT_GROUP_NAMES` for the 18 names).
- A pre-flight checklist run against every wire (power, motor, pot, current sense, Hall A/B) — any failed crimp, mis-pin, or insulation issue re-done before any board is energized end-to-end.
- A manual controller test (gamepad or jog UI) confirming each of the 18 motors responds to commands and the firmware is reading sane pot / current / Hall values.
- Photos of the bench layout and a short hardware-issues log committed to `krabby-research/hardware/Uno-v0.3/bringup/`.

Acceptance Criteria
- **0a** — All 18 motors (12 linear actuators + 6 hip-yaw) are physically wired to their slots on the V0.3 shields and to the octopus 24 V bus.
- **0b** — Each motor is labeled (heat-shrink, label-maker, masking tape — any durable scheme) with its joint name from `JOINT_GROUP_NAMES`.
- **0c** — All 3 Megas + shields + 6 H-bridge boards are mechanically secured to a single base board with strain relief on the high-current wiring.
- **0d** — Pre-flight wiring inspection complete: every connector pulled and re-seated; any visibly bad crimp re-done; insulation continuity confirmed on the motor-power and pot-signal lines.
- **0e** — Octopus 24 V bus energized with the 150 A fuse installed first (battery safety per M16 Task 3 if available, otherwise general high-amperage DC discipline).
- **0f** — Manual smoke test: with the controller / GUI, every one of the 18 motors moves in both directions, pot value changes accordingly, and (on Hall-equipped joints, Rev 3 slots 0–5) the Hall edge count increments.
- **0g** — Any hardware issues surfaced during 0d / 0f are logged in `hardware/Uno-v0.3/bringup/issues.md` with photos and a resolution note (re-crimp, replacement part, etc.).
- **0h** — Bench layout photos committed to `hardware/Uno-v0.3/bringup/` so the next person who picks this up knows the wire routing.

---

## 1. The base-board layout

The base board exists so the bench setup can be moved to the chassis later without re-wiring. Suggested arrangement (adapt to materials at hand):

- **Three Megas + shields** mounted in a row, USB facing the operator. Mark which board is intended to be FRONT (primary), LEFT, RIGHT (matches `firmware/arduino/arduino.ino` `ACT_LIST_FRONT` / `_LEFT` / `_RIGHT` at ~L84–86). Until Task 1's role refactor lands, role is still SYNC-determined — labeling now just marks the operator's intent.
- **Six H-bridge boards** in two rows of three (one row per "side": front/middle/rear × left/right). The shield-to-H-bridge wiring is per the M8 PCB stack; reuse existing harness if available, otherwise crimp fresh.
- **Octopus** (24 V bus + fuse) along one edge; battery (or bench PSU stand-in) at the same edge. Keep the high-current rail physically separated from the signal-wire bundles.
- **18 motors** arranged in a 3 × 6 grid keyed to their joint name (front-left to rear-right, hip-yaw / hip-pitch / knee). Doesn't need to match leg geometry — it just needs to be unambiguous which physical motor is which named joint.

## 1.5 Bring the inter-board serial lines out of each case

Each Mega + shield stack sits in a 3D-printed enclosure (from M12). The shield's Serial1 (D18/D19) and Serial2 (D16/D17) connectors are *inside* the case — the primary needs to reach them to drive the LEFT/RIGHT followers per Task 1. Pull each board's Serial1 and Serial2 signal pairs out of its case in a way that **lets the case still close**:

- Crimp/solder a short flying lead (4 wires: TX1, RX1, TX2, RX2 — Serial1 TX/RX and Serial2 TX/RX) onto the shield's Serial1 and Serial2 connectors.
- Land the other end on a 2×1 (or 2×2) **Dupont** female header so it can mate with a standard jumper cable.
- **Hot-glue the wires to the Dupont** to keep it from being pulled loose, and route them out of the case through whatever gap or notch is available (the case lid will close around the flat wire bundle).
- Label each Dupont end with the board's intended role (FRONT primary, LEFT, RIGHT) and which UART (S1 vs S2) so the operator can plug the inter-board cables correctly.

The result: every case closes normally, and inter-board UART connections (primary→LEFT on Serial1, primary→RIGHT on Serial2) are accessible Dupont-to-Dupont jumpers outside the case. This is the cabling Task 1 §2's primary↔follower comms debug will be exercising.

## 2. Pre-flight wiring inspection

For each of the 18 motors, check every connection on the path:

| Wire | Path | Inspect |
|---|---|---|
| Motor power (+) | H-bridge OUT_R / OUT_L → motor + lead | crimp tight, no exposed strands, polarity-marked |
| Motor power (−) | H-bridge OUT → motor − lead | same |
| Pot VCC (5 V) | shield → pot 3-wire connector (red) | continuity end-to-end |
| Pot signal | shield A0–A5 → pot wiper | continuity, no shorts to VCC/GND |
| Pot GND | shield → pot GND | continuity |
| Current sense | H-bridge IS pin → shield A6–A11 | continuity |
| Hall A (Rev 3 only) | motor Hall A → shield D50/D51/D52 | continuity |
| Hall B (Rev 3 only) | motor Hall B → shield A12/A13/A14 | continuity |
| Hall VCC / GND | motor Hall power | continuity |

Use a multimeter on continuity mode with the system de-energized. Anything that doesn't ring continuous, or anything visibly mangled (frayed wire, bent pin, half-seated crimp), re-do before powering up. **Re-crimping is the expected slow step** — Fletcher's note: "some time is reserved for this task but ideally will not be necessary."

For Hall pins, reference `firmware/arduino/board_pins.h` Rev 3 block (~L47–55) — D50–D52 are Hall A (Port B PCINT0), A12–A14 are Hall B (Port K PCINT2). On Rev 2 boards (the older V0.1 shield) Hall is absent; the V0.3 boards are Rev 3 builds.

## 3. Powering up (high-amperage discipline)

Cross-reference with M16 Task 3's battery-safety section if you've read it. The summary:

- **Install the 150 A fuse on Pack+ first**, as the closest element to battery positive.
- Make/break all other connections with the fuse pulled.
- Insulated tools only. Eye protection. Don't bridge Pack+ to chassis.
- Bring up the octopus on its own first (no boards connected) and verify 24 V at the bus bar.
- Then connect one H-bridge at a time, verifying current draw at idle is sane (~tens of mA) before adding the next.

If M12's actual octopus harness isn't built yet, a bench harness with the same topology (2× 12 V LiFePO4 in series → 150 A fuse → 200 A shunt → bus bar) is fine. See [M16 Task 3 §1](../Milestone16-I2C-Sensors/TASK-3-POWER-BUS-INA228.md) for the bench-octopus fallback.

## 4. Manual smoke test (every motor moves)

With everything powered, run the existing GUI:

```bash
python -m firmware.gui [--port COM5]
```

(or whatever USB port the primary board is on — see `firmware/gui/app.py` and `firmware/krabby_mcu.py`). The GUI lists every joint by name; click `Retract` / `Extend` on each one in turn and confirm:

1. **Motion** — the motor actually moves.
2. **Pot** — the `Pot` column changes (raw ADC; 0–1023). If it's stuck or noise-only, the pot wiring is the suspect.
3. **Current** — the `Cur` column reads roughly zero idle and non-zero when driving. If it's stuck at zero or at max, the current-sense wiring is the suspect.
4. **Hall** — on Hall-equipped joints, the `Hall` column's edge count increments as the motor moves. The count may not be direction-aware yet (that's Task 2) — just confirm it's moving, not direction.

If any motor doesn't respond, swap it through the diagnostic tree: command from GUI → reaches MCU? (check serial RX) → MCU drives correct H-bridge channel? (scope the PWM/EN pins) → H-bridge outputs voltage? (DMM at OUT pins) → motor leads connected? Capture the result in the issues log (0g).

Hand-jog is also fine: with the Mega off and the H-bridge disabled, move the motor's lead-screw by hand and watch the pot reading. This confirms the pot side without involving the H-bridge.

## 5. What if the primary↔follower comms bug shows up here

It probably will. Task 1 is where the fix lives — for Task 0, the bar is "the FRONT-board joints work when driven directly via USB." If LEFT / RIGHT joints don't respond because the leader isn't forwarding, **log it in the issues file and move on**. Don't sink Task 0's half-day into Task 1's open-ended debug.

If the FRONT board's joints (FLHY, FLHL, FLKL, FRHY, FRHL, FRKL — 6 joints) all work and LEFT / RIGHT don't, that's the expected pre-Task-1 state and Task 0 passes.

## 6. Deliverable checklist
- [ ] 18 motors physically wired and labeled with joint names.
- [ ] 3 Megas + 6 H-bridges mounted on a single base board with strain relief.
- [ ] Serial1 + Serial2 brought out of each case on a hot-glued 2×1 (or 2×2) Dupont; case still closes; ends labeled with role + UART.
- [ ] Pre-flight inspection complete; any failed crimps re-done.
- [ ] Octopus bus energized with fuse-first discipline.
- [ ] Manual smoke test: every motor moves; pot / current / (Hall where wired) values look sane.
- [ ] Hardware issues log + bench layout photos in `hardware/Uno-v0.3/bringup/`.
- [ ] Known-broken paths (e.g. follower comms) noted for Task 1, not fixed here.
