# Task 2 — Real robot on two buses

Goal: Put six flashed boards on the robot and move all 18 joints from the CLI, confirmed by a sweep script and an unplug test.

Outputs
- SDK support for more than one CAN bus.
- A `show` command that lists boards from their BoardInfo messages.
- `firmware/stm/scripts/sweep_joints.py`, which moves and checks every joint.
- Orin CAN setup steps in `docs/JETSON_DEPLOYMENT.md`.
- A task report with each board's ID and firmware version, any polarity fixes, and the measured hall counts.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-2---real-robot-on-two-buses).)

---

Note: line numbers below point at the right function at the time of writing and may drift.

---

## 1. Wiring

The J4012's carrier board has two separate CAN ports, CAN0 and CAN1, which should appear in Linux as `can0` and `can1`. Check this on the robot, and follow Seeed's J401 documentation if the ports need enabling. Connect the left legs to one port and the right legs to the other, with each chain running from front to rear:

```
can0: Orin -> FL (1) -> ML (3) -> RL (5)
can1: Orin -> FR (2) -> MR (4) -> RR (6)
```

The Orin ports use different connectors (CAN0 is XT30 2+2 and CAN1 is GH 1.25), and the boards use 4-pin JST-PH, so each bus needs its own adapter cable. Use only the CANH, CANL, and GND wires from the Orin.

Each bus needs 120 Ω at both ends. A rear board takes a terminator plug in its unused CAN connector. The Orin end of each bus also needs termination, which differs from the Milestone 20 design, where the Orin sat in the middle of one bus. Check whether the carrier board terminates its ports. With the power off, a correctly terminated bus measures about 60 Ω between CANH and CANL.

Set each board's switch to its leg ID before installing it. Closed switches read as 1 bits, with `NODE_ID0` as bit 0, so FL (ID 1) has only the `NODE_ID0` switch closed.

The motor connectors `MOTOR1`, `MOTOR2`, and `MOTOR3` go to yaw, hip, and knee. The sensor connectors carry 5 V, and a miswired motor has burned out pot inputs before, so check each sensor cable before connecting it.

## 2. Orin setup

The host sets the CAN bit rate before any container starts. Add this to `docs/JETSON_DEPLOYMENT.md`, using `systemd-networkd` so that it survives a reboot:

```
# /etc/systemd/network/80-can.network
[Match]
Name=can0 can1

[CAN]
BitRate=1M
RestartSec=100ms
```

## 3. SDK changes

`KRABBY_CAN_BUSES` becomes a comma-separated list, with a default of `can0,can1`. This is the only configuration the task adds, and a simulator run sets it to `vcan0`.

`connect()` opens every listed bus, and each bus gets its own receive thread writing to the same `joints` map. `set_duty()` sends each frame on every bus. Each board already ignores other legs' commands, so the SDK needs no leg-to-bus map. If a send fails because a bus is down, log the error and continue with the other bus.

Add `is_stale(leg)`, which returns true when the leg has sent nothing for 500 ms. `telemetry` shows `STALE` for those legs.

`show` listens on all buses for 2 seconds and prints each leg that sent BoardInfo with its version and commit. It lists any of the six legs that stayed silent as `missing`, and then prints the S3 branch list as the old `cmd_show` does (`firmware/cli.py` ~L137).

## 4. Bring-up order

Start with one board at a time, so that a wiring fault can only damage one. For each board, set the switch, flash it over SWD, and connect it alone to the Orin with a terminator. Confirm that `show` lists it, then jog its three motors at low duty. Correct any motor or hall polarity in the constants from Task 1, and note the correction in the task report.

Next, connect the three left legs to `can0` and run the same check, followed by the three right legs on `can1`. Then run both together.

## 5. Sweep script

`sweep_joints.py` visits the 18 joints in order. For each joint it applies a duty of +0.3 for 1 second, applies a duty of -0.3 for 1 second, and stops. It checks that the pot and hall values moved in the expected direction both times, and at the end it prints a pass or fail per joint. Run it with the robot on a stand, and watch that the joint it names is the joint that moves. A person has to check this, because if a motor cable and its sensor cable are both plugged into the wrong joint's connectors, the telemetry still looks correct. Add `--joint FL_yaw` to repeat a single joint and `--duty` to change the speed.

Joints that have no pot connected fail the pot check. Report these joints in the output and record them in the task report, and use the hall count as the pass condition for them.

## 6. Unplug test

Command zero duty on all legs before this test, because a leg that loses its CAN cable keeps running at its last duty (FAQ 3). Unplug one leg's CAN cable at the point where it joins its bus. Legs further down that chain lose their connection too, so unplug a rear leg for this check. Confirm that `telemetry` marks the leg `STALE` within 1 second and the other five legs still jog. Plug it back in and confirm the leg returns without a restart, which proves the bus-off recovery from Task 1.

## 7. Hall count measurement

Drive one yaw, one hip, and one knee joint slowly from one mechanical stop to the other, and record the change in hall count for each. Stop as soon as the joint reaches its limit, because this firmware has no end-stop protection. Record the results in the task report and set `stm_counts_per_full_travel` in `firmware/sim/hall_encoder_scale.json`. If the three joint types differ substantially, make the value per joint type.
