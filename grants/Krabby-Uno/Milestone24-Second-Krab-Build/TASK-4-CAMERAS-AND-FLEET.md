# Task 4 — Cameras and fleet manager on both robots

Goal: mount and bring up the cameras on both robots, then enroll both into the fleet manager and test telemetry and teleop with two robots online, reporting a list of issues to Bruce/Fletcher.

(Acceptance criteria and time estimate live in [OVERVIEW.md](OVERVIEW.md#task-4---cameras-and-fleet-manager-on-both-robots).)

---

## 1. Mount the cameras

Each robot gets a front depth camera and two side cameras. Where the chassis needs an opening, mark it from the camera body, drill the corners, and cut with a jigsaw — cut undersized and file to fit. Mount the front camera level, with the lenses clear of the frame. Side cameras mount through drilled holes in the chassis sides.

Cut robot 1's openings first and write down the dimensions and reference edges, then reuse the notes on robot 2.

## 2. Bring the cameras up

Follow the camera setup guide in `krabby-research` (`hal/server/jetson/README.md` and the ZED steps in the setup docs), and update the guide with any steps that turn out to be missing, so the setup is reproducible on a fresh install.

Cameras come up through the Jetson HAL sensor catalog: `hal/server/jetson/front_camera_factory.py` registers the drivers (`"zed"`, which takes an optional `zed_serial_number`, and `"maixsense_a075v"`), and the HAL server opens one instance per RGB-D row in `JETSON_SENSOR_CATALOG`. With two robots, set each robot's catalog rows to identify its own cameras (serial numbers), and commit both configurations.

Reboot each robot and confirm every camera comes back without re-plugging anything.

## 3. Enroll both robots

Enroll each robot with its own identity per `fleet/ENROLL.md` (`krabby enroll`; fleet side in `fleet/service/` and `fleet/portal/`). Then check with both online: both show live telemetry in the portal at the same time, each robot's telemetry shows under the right robot, and a robot that goes offline and comes back shows up as the same robot, not a new one.

## 4. Teleop

Test a teleop session against each robot — video and control, landing on the robot it was aimed at. Then test with both robots reporting telemetry while a session is active on one, and confirm the idle robot is undisturbed.

The fleet path has never run with two devices, so some of this may not work. Keep a list of what breaks and report it to Bruce/Fletcher rather than trying to fix the fleet service inside this task.

## 5. Wrap-up

Note robot 2's known differences where the next person will find them (in the setup guide or the robot's config): rev 2 pin map, no hall wiring, v0.1 boards, AC-powered.
