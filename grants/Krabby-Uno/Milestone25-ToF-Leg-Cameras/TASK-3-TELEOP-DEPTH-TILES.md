# Task 3 — Teleop depth tiles

Goal: show the three ToF cameras as depth tiles in the teleop UI and validate performance over the internet.

Outputs

- The three depth streams selectable and labeled in the teleop viewer.
- A latency report from an internet teleop session with every stream selected.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-3---teleop-depth-tiles).)

Line numbers below point at functions, not literal addresses; they drift as the code changes.

## 1. What already works

Multi-camera teleop is proven. The robot creates one WebRTC track per selected stream (`teleop/edge/rtc_session.py:90`), colorizes depth for every `{id}_gray16_depth` preview stream (`teleop/edge/depth_preview.py`), and the browser viewer renders a checkbox per advertised stream with tiles in an auto-fit grid. Tasks 1 and 2 gave the ToF catalog rows `gst_depth_quant_range_m=(0.15, 1.5)`, so each one already lists a preview stream, and selecting it produces a colorized depth tile with no new rendering code.

## 2. Updates for the new cameras

- `hal/server/jetson/main_model.py:123` starts teleop with only the primary camera id, so the robot does not advertise the rest until a browser asks. Source the list from `hal_rgbd_catalog_ids_for_recording()` plus the preview ids, matching the Isaac entrypoint (`hal/server/isaac/main.py:40`).
- With the MaixSense cameras removed there are five streams: the front camera's RGB and depth views plus the three ToF depth views. They fit under `MAX_VIDEO_M_LINES = 8` (`teleop/edge/robot_settings.py:39`), so the cap stays.
- Give the ToF tiles readable labels through an id-to-label map in the viewer, for example `belly right (depth)`; the raw id `belly_right_tof_gray16_depth` is not operator-friendly.
- The viewer checks every discovered stream by default. Change the default to the front RGB stream plus the three ToF tiles, everything else opt-in.
- Extend the depth colorizer unit tests with the 0.15-1.5 m range (`tests/unit/teleop/test_depth_preview.py`).
- The viewer exists as two copies, `teleop/portal/static/` and `fleet/portal/public/teleop/`; make every UI change in both.

## 3. Internet validation

Run a fleet-portal teleop session against the robot over the public internet, not the LAN, with every stream selected. Confirm the three ToF tiles render alongside the existing streams, control stays responsive, and the session degrades by lowering frame rates rather than failing. Measure camera-to-screen latency with the existing machinery (`teleop/edge/latency.py`, the HUD stats) and record the numbers in the task report; the documented targets are under 300 ms for one stream and under 500 ms at four, and this report sets the baseline at five. The ToF tiles are 100x100 at 20 fps, a small fraction of one RGB stream's bandwidth, so `QOS_KBPS_BUDGET_PER_STREAM` (`teleop/edge/robot_settings.py:53`) should hold; adjust only if the measurements say otherwise.
