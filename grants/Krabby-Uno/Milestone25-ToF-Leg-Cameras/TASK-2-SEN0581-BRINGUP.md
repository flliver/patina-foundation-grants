# Task 2 — SEN0581 bring-up on the robot

Goal: drive the physical SEN0581 from the Jetson through a new serial driver, register it in the camera factory and sensor catalog, and prove its depth frames arrive in the observations the inference client receives alongside the existing three cameras.

Outputs

- `hal/server/jetson/sen0581_tof_camera.py`: a depth-only capture driver speaking the SEN0581 USB serial protocol, with recorded-fixture unit tests.
- Factory registration and three catalog rows with the belly and rear poses.
- Catalog validator rules for depth-only rows and the SEN0581's device-path field.
- `depthai`-free dependencies: the serial library added to both locomotion images, a committed udev rule, and `docs/SEN0581_SETUP.md`.
- On-robot validation alongside `front_rgbd` and both MaixSense rows, surviving reboot.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-2---sen0581-bring-up-on-the-robot).)

Line numbers below point at functions, not literal addresses; they drift as the code changes.

## 1. The driver

The SEN0581 connects over USB-C and speaks a documented serial protocol; the wiki's reference section (https://wiki.dfrobot.com/sen0581/) covers the frame format, and DFRobot publishes host tools and ROS drivers to crib framing details from. Expect a USB CDC serial device (`/dev/ttyACM*` or `/dev/ttyUSB*`) streaming 100×100 depth frames at up to 20 fps; verify against the real unit first and record a few raw captures; they become the unit-test fixtures.

Model the driver on the MaixSense pair: `hal/server/jetson/maixsense_a075v.py` is the protocol client and `hal/server/jetson/maixsense_rgb_depth_camera.py` adapts it to the `RgbDepthCamera` protocol (`hal/server/jetson/rgb_depth_camera.py`: `get_camera_frames() -> (rgb, depth)`, `is_ready()`, `close()`). The SEN0581 adapter returns `(None, depth)` with depth as `(100, 100) float32` meters; the server turns that into a catalog entry whose only channel is depth (Task 1's restructuring). Convert whatever integer depth unit the protocol uses to meters in the client, mark out-of-range and invalid readings as zeros the way the MaixSense client does, and keep a background read thread with a latest-frame slot so `get_camera_frames` never blocks the 100 Hz control loop. `is_ready()` reflects whether frames arrived recently; on serial errors, close and reopen with backoff rather than raising into the grab loop: `JetsonHalServer.set_observation` treats `None` frames as zero-fill but propagates exceptions (`tests/unit/hal/server/jetson/test_jetson_hal_grab_placeholders.py` pins both behaviors).

Unit tests replay the recorded raw captures through the protocol parser without hardware and assert frame shape, dtype, meter scaling, and recovery across a truncated frame.

## 2. Factory and catalog registration

Register `"sen0581"` in `FRONT_RGB_DEPTH_CAMERA_FACTORIES` (`hal/server/jetson/front_camera_factory.py:58`) and add its branch in `create_front_rgb_depth_camera` (line 64), which forwards driver-specific kwargs per driver today. The driver's selector kwarg is a device path.

In `JETSON_SENSOR_CATALOG` (`hal/server/jetson/sensor_backend_jetson.py:61`), remove the two MaixSense rows (`side_right_rgbd`, `side_left_rgbd`) and add three rows with ids `belly_right_tof`, `belly_left_tof`, `rear_tof`, matching Task 1's sim ids exactly so the sim and the robot expose the same cameras. The right-side MaixSense row carried `policy_scan_slot="side"`, and the active model configuration uses no side scan (`num_side_scan=0` in `compute/parkour/model_definition.py`), so removing it changes nothing for the policy. Fields per row: `type="rgbd"` so the existing open/grab machinery applies, a new `modality="depth"`, `resolution` and `depth_resolution` both `(100, 100)`, `fps=20`, `hal_open_rgbd=True`, `camera_driver="sen0581"`, `gst_depth_quant_range_m=(0.15, 1.5)` so the row lists a `{id}_gray16_depth` preview stream, no `policy_scan_slot`, and a new `sen0581_device` field carrying the udev-stable path (section 4). Poses follow the OVERVIEW's Information section; derive the numeric body-frame offsets once in Task 1 and reuse them here so sim and catalog agree.

`JetsonHalServer.initialize_cameras` (`hal/server/jetson/hal_server.py:219`) already opens every `hal_open_rgbd` row and warn-and-skips failures, so a catalog carrying three ToF rows with one camera physically present comes up cleanly; the absent two log a warning and publish nothing. The grab loop in `set_observation` (line 451) raises on a frame-shape mismatch against the catalog resolution, which now protects the 100×100 contract too. Where that method builds the entry (around line 589), emit only the channels the row declares, a single depth channel for these rows, instead of zero-filling an RGB image.

## 3. Validator extension

`assert_hal_rgbd_catalog_config` (`sensor_backend_jetson.py:147`) grows rules: a `camera_driver="sen0581"` row requires a non-empty `sen0581_device`; a `modality="depth"` row must not be `is_primary` (the primary camera sources RGB for teleop and the legacy policy fields, and the ZED additionally sources the IMU and odometry via `zed_imu.py` and `zed_tracking.py`, so a depth-only primary would silently zero the base state) and must not set `policy_scan_slot` (the scan extractors assume the primary camera's geometry). Extend `tests/unit/hal/test_sensor_interface.py` and `tests/unit/hal/server/jetson/test_front_camera_factory.py`, which currently pin the exact driver and row sets.

## 4. Dependencies, udev, and the setup document

Add the serial dependency (`pyserial` unless the protocol work in section 1 shows otherwise) to `images/locomotion/requirements.txt` and `images/locomotion/requirements.release.txt`; the container needs the device passed through, which the fleet launcher's `docker run` assembly in `krabby/_docker.py` must include (a `--device` argument per configured `sen0581_device`).

Serial device names can change between boots, and the robot will eventually carry three identical units, so a committed udev rule mapping each physical camera (by serial number if the protocol exposes one, otherwise by USB port path) to a stable symlink like `/dev/krabby-tof-belly-right` is required, installed by the setup document's steps. The catalog rows reference the symlinks.

`docs/SEN0581_SETUP.md` follows the shape of `docs/MAIXSENSE_A075V_SETUP.md`: unboxing to streaming, the udev rule, the aim check using the unit's own LCD against the mount study's 55°/15° nominal, and the verification commands. It must be complete enough that the second robot's camera comes up from the document alone; the M24 convention that configuration is committed and reproducible on a fresh install applies.

## 5. On-robot validation

With the stack running under `krabby run` (or the dev image via `scripts/jetson/run_jetson_hal_server_host.sh`), confirm with `hal/tools/hal_dump.py` that `rgbd_by_catalog_id` carries the ToF id with live 100×100 depth alongside `front_rgbd`, and that `hal/tools/multi_stream_display.py` shows its colorized preview. The inference client reads the same stream, so if the dump shows the camera, the model receives it. Watch the control loop rate while the camera streams (`tests/integration/test_timing.py` thresholds are the reference); at 200 KB per frame burst the budget is comfortable, and the check is that nothing regresses. Reboot the robot and confirm the camera returns with no replugging.
