# Task 1 — Fix robot 1 and get it running stably

Goal: fabricate and mount robot 1's six hip yaw motors, install the Orin, 24 V battery, and Bluetooth controller, validate all 18 legs from the controller, and fix the two firmware bugs: only one actuator moves at a time, and the yaw encoder wiring floods the Arduino with interrupts until it locks up.

(Acceptance criteria and time estimate live in [OVERVIEW.md](OVERVIEW.md#task-1---fix-robot-1-and-get-it-running-stably).)

Line numbers point at the right function and drift as the code changes.

---

## 1. Hip yaw motors

1. Assemble the slot boards. Left and right mirror each other, so confirm orientation first. Press the 6201RS bearings into their pockets and dry-fit the shoulder bolt — it should spin freely with no wobble.
2. Fit the lever arms. Bolt each gearmotor to its mount plate, fit the output to the yaw pivot, and rotate the joint by hand through its full range. Sand the slot wherever it rubs. Legs 1-3-5 and 2-4-6 mirror; confirm the side before bolting.
3. Crimp, label, and tug-test every lead — power and POT. The pin assignment at the board is `firmware/MOTOR_HEADER_PINOUT.md`; connector families are in `hardware/Uno-v0.2/diagrams/BOM.md`.

## 2. Orin, battery, and controller

Install the Orin and the 24 V pack with its breaker and distribution block. Pair the Bluetooth controller per `controller/scripts/jetson/CONNECT_PRO_CONTROLLER.md` and start the stack with `krabby run` (see `controller/scripts/jetson/E2E_GAMEPAD_KRABBY.md` — the two-process split there is the debug path).

Validate all 18 legs respond from the controller. For anything that doesn't, do basic triage — connector seated, right channel, right direction — and let Fletcher know if help is needed root-causing controller or other issues. Debugging time is budgeted in the estimate.

## 3. One-motor-at-a-time regression

The hardware supports all six actuators per board moving at once (`LinearActuator::driveActuator`, `actuator_manager.h:224`, writes each channel independently), so this is a firmware regression. Four candidate causes; check which are actually happening before changing anything, since more than one may be true.

1. **Stuck calibration state machine.** `updateAll` (`actuator_manager.h:274`) skips all actuator updates unless `calState` is `CAL_IDLE`, and `updateCalibration` (`actuator_manager.h:371`) drives one hard-coded actuator at a time while all `T`/`B`/`J` commands are ignored. A single stray `C` byte on any serial line enters this mode (`arduino.ino:347`), and `isStalled` (`actuator_manager.h:171`) uses function-local statics shared across all actuator instances, so the exit condition can wedge with no timeout or abort. Check `calState` first. Fixes: per-actuator stall state, a timeout back to `CAL_IDLE`, and a guard on the bare `C` byte.
2. **Blocking batch-jog parser.** The `B` handler (`arduino.ino:315`) reads tokens straight off the stream and breaks on `peek() == '\n'`; mid-transmission `peek()` returns -1, so it blocks for the 1 s stream timeout and then eats part of the next command line. Read the whole line into a buffer, then parse.
3. **All-or-nothing target parser.** `parseCommands` (`command.h:50`) drops the entire `T` line if it carries fewer than 18 name-value pairs (`CMD_BUF_SIZE`, `arduino.ino:102`). A partial command should apply the joints it names.
4. **Failed role election.** A follower that misses the one-shot `ROLE:` assignment (`arduino.ino:214`) falls to the timeout path (`arduino.ino:227`) and installs the FRONT actuator list anyway, leaving two boards answering to the same six joint names. Also settle the wiring contradiction: `arduino.ino:15` defines `SERIAL_LEFT` as Serial1 while the header comment and `MOTOR_HEADER_PINOUT.md` say Serial3 — make the code, harness, and docs agree.

To prove the fix, command a whole-body pose and confirm every joint moves, and add a test that fails against the unfixed code. The parsers in `command.h` and the `B` handler are unit-testable without hardware.

## 4. Encoder lockup

The yaw encoder wiring sends interrupts faster than the board can service them and the Arduino locks up — the yaw gearmotors carry a 2500-line optical encoder, which is a huge edge rate on the PCINT pins at motor speed.

The likely fix is rewiring: move the encoder input to the slower tick wire instead of the high-resolution channel, and confirm counts still change in telemetry. The ISRs themselves are minimal (`hall_hw.cpp:35`), so if lockups continue after the rewire, check the blocking code around them: `cli()` in `hallHwGetEdgeCount` (`hall_hw.cpp:105`), the two `mainSerial->flush()` calls at ~35 ms each (`arduino.ino:424`, `:432`), and the `readVerLine` spin-wait (`arduino.ino:256`).

Verify by running the yaw motors at speed for several minutes with telemetry flowing and no reset.

## 5. Wrap-up

Drive the robot on the controller for 5-10 minutes: no board hang, no lost joint, no manual restart. Log other reliability issues for later fixes rather than chasing them now.

Before Task 2 borrows the motor controllers, photograph the harness and note which actuator is at which joint.
