# Task 3 — Support week 1

Goal: Spend 20 hours fixing what breaks when the Task 1 and Task 2 work meets Denny's real boards and Nick's robot work, worked from one shared, prioritized list.

Outputs
- `SUPPORT-LOG.md` in this folder: the prioritized list, with hours and outcome recorded per item.
- The fixes themselves, each merged with a test or a recorded check.
- A Discord summary at the end of the week.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-3---support-week-1).)

---

Note: this task is time-boxed, not scope-fixed. The commitment is 20 hours, worked in priority order. The number of items completed is an outcome.

---

## 1. The list

Keep one list in `SUPPORT-LOG.md`. Each item records who reported it, what goes wrong, an estimate in hours, and whether it needs the robot or can be done against the sim. Denny and Nick set the priority at the start of the week. Items that block robot bring-up go first.

Start the list with these items, in this order:
1. Bring up `can0` and `can1` on the Orin as CAN FD at 1 Mbit/s arbitration and 5 Mbit/s data. Denny's bench tests so far used a CANable2 USB adapter on Windows (`hardware/PCB-v0.3/Firmware/bench/CAN_TEST_NOTES.md`), so the Orin's own ports are untested.
2. Check that the `krabby-firmware` CAN SDK talks to Denny's firmware as described in `hardware/PCB-v0.3/Firmware/docs/APPLICATION_PROTOCOL.md`, and fix any gaps.
3. Add the Task 1 calibrate command and calibrated position telemetry to Denny's firmware, using its existing flash configuration storage.
4. Move Denny's control GUI (`hardware/PCB-v0.3/Firmware/tools/krabby_control_gui.py`, described in `CONTROL_GUI.md`) into `krabby-firmware`. Today it talks to a CANable2 over a COM port. Make it use SocketCAN on the Orin, and list every board it hears on the bus. It replaces `firmware.gui` for v0.3 boards.
5. Publish the `krabby-firmware` version with these changes to PyPI.
6. On the Orin, `pip install krabby-firmware` from PyPI, plug in a board, and confirm the GUI lists it, shows its telemetry, and drives its motors.
7. Run the release image with `--stm` on the Orin against the real boards, and fix what breaks.
8. Fix differences between the simulated boards and the real ones, such as drive polarity, hall counts per full travel, and telemetry rates, so the sim matches the robot.
9. Help Nick turn the model's actions into duty for the linear actuators.

Bugs Denny or Nick report join the list as they come in.

## 2. Working the list

Work from the top down. An item that runs over its estimate is either finished, split so the useful part lands, or returned to the list with a new estimate and a note on what the first one missed. Record actual hours against every item, including returned ones.

An item is done when the problem it describes no longer occurs. Show this with a test where one can be written, or by repeating the step that failed on the robot or in the sim and recording the result.

## 3. End of the week

Post a short summary in Discord: the items finished, the hours spent, and the top of the remaining list. Task 4 continues from the same list.
