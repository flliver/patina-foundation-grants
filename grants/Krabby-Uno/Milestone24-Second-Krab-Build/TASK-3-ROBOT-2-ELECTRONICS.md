# Task 3 — Second Jetson and second controller set

Goal: set up robot 2's Jetson, install the three older v0.1 motor controller boards, build and flash firmware for their pin revision, and get robot 2 walking under a gamepad.

(Acceptance criteria and time estimate live in [OVERVIEW.md](OVERVIEW.md#task-3---second-jetson-and-second-controller-set).)

Line numbers point at the right function and drift as the code changes.

---

## 1. The second Jetson

Set up the second Jetson the same way as robot 1's: get network and SSH up, `pip install krabby-launcher`, `krabby install` to pull the locomotion image and set up services, `krabby run` to start the stack. Entry points are `krabby/install.py`, `krabby/run.py`, and `krabby/_docker.py`. Record any steps that were needed but not documented.

## 2. Install the v0.1 boards

Install the three ATmega2560 boards on the v0.1 shields, with their H-bridge power boards, in their enclosures. Power comes from a 24 V AC-to-DC converter (robot 2 does not get a battery pack in this milestone). Run 24 V to the boards, the signal and motor harness out to all 18 actuators, and USB from the Jetson to the three Megas.

Label every connector by leg and joint, and tug-test the crimps. Keep the same leg/joint numbering Task 2 used so its validation table lines up. Design files for these boards are in `hardware/Uno-v0.1/shield/` and `hardware/Uno-v0.1/power-board/`.

## 3. The pin revision

The firmware carries three pin maps selected by `KRABBY_PIN_REV` in `firmware/arduino/board_pins.h`; the v0.1 boards are revision 2.

The difference is the enable pins: rev 3 (robot 1's v0.2 boards) uses D22/D24/D26 and D23/D25/D27, rev 2 uses D22/D23/D24 and D28/D26/D27. PWM, current sense, and pot pins are identical, so a board flashed with the wrong revision reads sensors fine and drives the wrong actuators. Rev 2 also has no hall wiring (`hall_hw.cpp:97`), so just make sure it works w/o the hall encoders.

Build and flash with `PIN_REV` (defaults to 3 in `firmware/Makefile`):

```
make -C firmware upload-firmware PIN_REV=2 PORT=/dev/ttyACM0
```

or, for boards on the Jetson's USB hub, one board at a time:

```
make -C firmware flash-remote REMOTE=<jetson> PORT=/dev/ttyACM0 PIN_REV=2
```

Do not flash robot 2 with `krabby firmware`: the publish workflow compiles without `KRABBY_PIN_REV`, so the published artifact is a rev 3 build. Note that gap as a backlog item.

Each board prints its pin revision at boot (`board_pins.h:60`); capture that for all three boards to confirm the right build is on the right hardware.

## 4. Channel check

Confirm role election completes and all three boards are addressable (`krabby firmware show`). If Task 1 resolved the `SERIAL_LEFT` wiring question, wire robot 2 to match.

Then check all 18 channels: right channel moves, right direction, no binding. Compare against Task 2's validation table — a joint that worked under borrowed controllers and fails now points at the board revision or the new wiring. Fix any firmware regressions preventing the old pin config from working, or report them to Fletcher if they can't be fixed within the task.

## 5. Walk test

Pair a controller to robot 2's Jetson, run `krabby run`, and drive the robot for 5-10 minutes: no board hang, no lost joint, no manual restart. Commit any rev-2 fixes to the repo rather than leaving local patches on the SD card.
