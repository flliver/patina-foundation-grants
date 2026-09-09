# Task 2 — Finish robot 2's legs and body

Goal: finish robot 2's six leg modules, build the body, join the legs to it, and validate every joint by driving it with motor controllers borrowed from robot 1.

(Acceptance criteria and time estimate live in [OVERVIEW.md](OVERVIEW.md#task-2---finish-robot-2s-legs-and-body).)

---

## 1. Where robot 2 starts

Robot 2's leg modules are partly assembled and the body is not built. Start with an inventory: per leg, which pivots are pressed, which fasteners are in, which actuators are pinned, and which parts are missing or need a re-cut, so anything missing can be sourced without stalling the build.

Confirm hole positions and fastener counts against the CAD before drilling or bolting anything permanent.

## 2. Build steps

Each leg has three joints: hip yaw (gearmotor pivoting in the slot boards), and hip pitch and knee (linear actuators pushing on lever arms). Per leg: two 6201RS bearings, Teflon-taped joint disks per the CAD, 12 mm shoulder bolts with washers and nyloc nuts, two linear actuators, one yaw gearmotor, hinges, and connector pigtails.

1. Hip Yaw: Mount the motors, screw the cam slot and support boards together, dry-fit the cam and lever arm, mount it all and rotate it through full motion to make sure the cam doesn't hit anything.
3. Femur: screw the side plates to the spacer blocks, square before driving screws, install the knee pivot bolt and bearings, attach the hip-pitch lever arm.
4. Knee and lower leg: bolt the lower leg at the knee pivot, fold to the end stop by hand and confirm it reaches without binding, attach the knee lever arm, fit the foot.
5. Actuators: bolt body ends to frame mounts and rod ends to lever arms, unpowered. With both ends pinned the joint should hold position (lead screws self-lock). Run wires through hole in frame, mimicing first robot.
6. Body: assemble the panels (copy first robot) and mount the hinges. Confirm the yaw slot boards sit at the design height (robot 1 needed a hand-cut slot to drop the center boards an inch).
7. Legs onto the body: attach each leg's slot boards to the hinges (support block or a second pair of hands), swing each yaw by hand and confirm clearance. Legs 1-3-5 and 2-4-6 mirror; confirm the side before bolting.
8. Standing check: on blocks, hand-sweep every joint through its range; then on its feet, all six flat under the robot's own weight. 

## 3. Validate the legs with robot 1's controllers

Robot 1's controllers are known good after Task 1, so use them to test robot 2's legs before robot 2 has electronics of its own.

Move one board at a time. Photograph robot 1's harness and label both ends of every connector you move so it all goes back correctly. Wire the borrowed board to a leg, drive that leg's three joints in both directions at low duty, confirm direction is correct and nothing binds, then move to the next leg.

Record a table per joint: leg, joint, direction correct, range clear. Task 3 compares against it — a joint that works here and fails on the rev 2 boards points at the boards, not the leg.

## 4. Restore robot 1

Put robot 1's controllers back and confirm it still drives all 18 legs from its controller before closing the task.
