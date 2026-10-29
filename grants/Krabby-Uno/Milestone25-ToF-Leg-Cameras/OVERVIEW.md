# Patina Foundation Grant — Krabby-Uno Milestone 25: ToF Leg Cameras

## Grant Overview

Add three DFRobot SEN0581 time-of-flight depth cameras (100x100 pixels at 20 fps, 70°x60° field of view, 0.15-1.5 m range, about $30 each) so the robot can see its own legs: one under the belly on each side between the middle and rear legs, aimed out and down at that side's three feet, and one on the rear wall aimed at the ground behind the robot. They replace the two MaixSense side cameras, so after this milestone the robot carries four cameras: the front ZED 2i and the three SEN0581s. The belly position comes from a simulation visibility study in which each camera sees all three feet on its side in 99-100% of walking frames; the full geometry and clearances are in the Information section.

The milestone:

1. Simulates the three cameras in Isaac and pipes their depth through HAL into the observations the AI model receives.
2. Brings up the first physical SEN0581 on the robot through a new serial driver, publishing alongside the front ZED camera.
3. Adds the cameras to both teleop viewers as selectable depth tiles.
4. Records the camera streams as H.265 in the data collector, ships the collector on fleet robots for the first time, and uploads finished recordings to S3.

## Why is this Important?

- The robot cannot see its own feet. Direct leg observation is the prerequisite for foot-contact estimation and lets an operator diagnose a stuck leg remotely.
- Fleet robots record nothing today. After this milestone every walk lands in S3 as training data.
- Depth is the only modality leg observation needs, so the $30 SEN0581 covers it; three units plus a spare is under $130.
- The M10 control plane already provisioned certificate-based S3 upload. Building the device side finishes it without putting AWS keys on robots.

## Tasks

Total: ~20 working days, roughly 4 weeks part-time at 20-25 hours per week. Implementation detail and exact code locations live in the per-task files linked below; the acceptance criteria are here. Tasks run in order: Task 2 builds on Task 1's depth-only observation support, Task 3 displays the streams Tasks 1 and 2 produce, and Task 4 records and uploads them.

### Task 1 - Simulated ToF cameras and depth-only HAL support -> [TASK-1-SIM-TOF-CAMERAS.md](TASK-1-SIM-TOF-CAMERAS.md)
#### Narrative

Simulate the three SEN0581s in Isaac and pipe their depth through HAL into the observations the AI model receives. This requires: 1) the observation format hardcodes an RGB image plus a depth map for every camera and must be restructured as named channels declared by each sensor's type, 2) the existing sim cameras ray cast against the terrain, bypassing the robot's body, and the new cameras must use rendered cameras instead so the legs show up, 3) the two simulated side cameras are removed, since the SEN0581s replace them. Validation is an integration test that checks each camera's frames and a short walk script to confirm the legs are visible.

#### Acceptance Criteria

- 1a - Catalog sensor entries carry named channels declared by the sensor type, serialization handles channels generically, and unit tests round-trip a sensor set mixing rgb-and-depth and depth-only entries.
- 1b - Three simulated cameras with catalog ids `belly_left_tof`, `belly_right_tof`, and `rear_tof`, matching the SEN0581 spec, publish depth into `rgbd_by_catalog_id` alongside the front camera during an Isaac run, and the old simulated side cameras are gone.
- 1c - An integration test runs the sim headless with a test object placed in view at a known distance and verifies each new camera delivers 100x100 depth frames showing it at that distance, under the `test-isaacsim` make target.
- 1d - A small committed script walks the robot in sim while showing the three depth streams, and the legs have been confirmed visible in each belly camera using it.
- 1e - `docs/SENSOR_INTERFACE.md` documents the depth-only modality and the three new catalog rows.

#### Time estimate (~6 days)

| Days | Sub-task title |
|------|----------------|
| 1.5 | Channel-based sensor entries, serialization, and unit tests |
| 1.5 | Sim camera configs at SEN0581 spec and scene wiring |
| 1 | Isaac HAL server readers and sensor listing for depth-only rows |
| 1.5 | Integration test and leg-visibility walk script |
| 0.5 | Documentation |

### Task 2 - SEN0581 bring-up on the robot -> [TASK-2-SEN0581-BRINGUP.md](TASK-2-SEN0581-BRINGUP.md)
#### Narrative

Plug the first real SEN0581 into the robot and get its depth into the observations the AI model already receives. This requires: 1) a driver for the camera's USB serial protocol, built like the existing ZED and MaixSense drivers, 2) sensor catalog entries for all three camera positions, where a position with no camera installed is skipped with a warning, 3) removal of the two MaixSense catalog entries, since the SEN0581s replace those cameras. Done means the depth shows up alongside the front camera, comes back after a reboot, and the setup is documented well enough to repeat on the second robot.

#### Acceptance Criteria

- 2a - A `sen0581` driver returns 100x100 depth in meters through the same camera interface the ZED and MaixSense drivers use, with its protocol parsing unit tested against recorded captures, no hardware needed.
- 2b - All three camera positions are in the sensor catalog, the two MaixSense entries are removed, and the installed camera's depth shows up on the robot alongside the front camera, confirmed with `hal/tools/hal_dump.py`.
- 2c - The camera returns after a reboot with no manual replugging, via a committed udev rule, and a setup document reproduces the bring-up on a fresh install.
- 2d - The production locomotion image builds with the driver's dependencies and the dev image runs the driver end to end.

#### Time estimate (~5 days)

| Days | Sub-task title |
|------|----------------|
| 2 | Serial protocol driver with recorded-fixture unit tests |
| 1 | Factory registration, catalog rows, validator extension |
| 1 | Image dependencies, udev rule, setup document |
| 1 | On-robot validation alongside existing cameras, including reboot |

### Task 3 - Teleop depth tiles -> [TASK-3-TELEOP-DEPTH-TILES.md](TASK-3-TELEOP-DEPTH-TILES.md)
#### Narrative

Multi-camera teleop already works, so this task updates it for the new cameras and validates performance over the internet. This requires: 1) the robot advertises the three new depth streams at startup, 2) the new tiles get readable labels and a sensible default selection. The five streams (the front camera's RGB and depth views plus the three ToF depth views) fit the existing cap of 8. Validation is a fleet teleop session over the public internet with every stream selected and the measured video latency recorded in the task report.

#### Acceptance Criteria

- 3a - The three ToF depth tiles are selectable and correctly labeled in the teleop viewer alongside the existing streams, on both the sim and the robot.
- 3b - A session with every stream selected connects and degrades by lowering frame rates rather than failing.
- 3c - A teleop session against the robot over the public internet shows all streams with responsive control, with measured video latency recorded in the task report.

#### Time estimate (~2.5 days)

| Days | Sub-task title |
|------|----------------|
| 1 | Advertise the new streams, labels and defaults |
| 0.5 | Unit test updates |
| 1 | Internet teleop session and latency report |

### Task 4 - Recording and S3 offload -> [TASK-4-RECORDING-AND-OFFLOAD.md](TASK-4-RECORDING-AND-OFFLOAD.md)
#### Narrative

Record what the cameras see and upload it. The data collector already handles any number of cameras, but three gaps remain: 1) it stores raw uncompressed frames, fixed by H.265-compressing the camera streams (the front camera's RGB and depth, and the three ToF depth streams) and adding per-camera recording rates, 2) it is not installed on fleet robots, fixed by adding it to the release image and the fleet launcher, 3) nothing uploads its output, fixed by a new offloader that uses the robot's fleet certificate to get temporary S3 credentials, uploads finished recordings, deletes local copies after a verified upload, resumes after network drops, and stays under a bandwidth cap. Recordings rotate in 5-minute segments, about 100 MB each at the default rates, so uploads trickle continuously in the background, pause while a teleop session is active, and never touch the control loop. S3 keys are robot id, then date and time. The S3 bucket and permissions are added to the fleet's cloud setup.

#### Acceptance Criteria

- 4a - The front camera's RGB and depth and the three ToF depth streams are recorded as H.265 at their configured rates, and a committed playback script decodes them back to viewable RGB and depth in meters.
- 4b - Record rates are configurable per catalog id with committed defaults, replacing the single global image rate.
- 4c - The collector is included in the release locomotion image, records on a fleet-enrolled robot started by `krabby run`, and still records after a reboot.
- 4d - Finished segments upload to S3 using credentials obtained with the device certificate (no long-lived AWS keys on the robot), at a capped background rate that pauses while a teleop session is active; local segments are deleted only after a verified upload, and an interrupted upload resumes.
- 4e - End to end: after ten minutes of driving the robot, its recordings are in S3 keyed by robot id, then date and time, they play back with standard tools, and local disk usage stays bounded.

#### Time estimate (~6.5 days)

| Days | Sub-task title |
|------|----------------|
| 2 | H.265 recording path for RGB and depth, and the playback script |
| 1 | Per-camera record-rate configuration |
| 1 | Publish and deploy the collector on fleet robots |
| 1.5 | Offloader worker: credentials, upload, verify, delete, resume |
| 1 | CDK bags bucket, role grant, and the end-to-end check |

## Information

- Repositories: `krabby-research` (HAL, teleop, data collection, the `krabby` CLI, fleet service and infra), `patina-foundation-grants` (this grant), `krabby-contracts` (the milestone ICA).
- Hardware: DFRobot SEN0581, https://www.dfrobot.com/product-2741.html, $29.90 each, wiki at https://wiki.dfrobot.com/sen0581/ (PC guide, ROS guides, serial protocol reference). Order four: three mounts plus a spare. USB 2.0 Type-C. The 10 mm belly brackets are fabricated outside this milestone.
- Mount geometry (from the visibility study): each belly camera sits halfway between the middle and rear hip plates, 6 cm in from the side edge, hanging about 2.5 cm below the belly on a 10 mm bracket, lowest point about 11 cm above the bottom of the hip plates. Aim is 55° down, yawed 15° toward the front legs, facing outward; the study allows 50-60° and 10-20°, with criterion 1c as the binding check. Measured clearances: 90-94% of the lower shins visible, at least 12 cm from any leg while walking, about 3 cm in limp falls. The rear camera's nominal is the rear wall center, aimed rearward and 45° down, tuned by the same method.
- Sensor catalog and drivers: `hal/server/jetson/sensor_backend_jetson.py` (`JETSON_SENSOR_CATALOG`), `hal/server/jetson/front_camera_factory.py`, with `maixsense_rgb_depth_camera.py` / `maixsense_a075v.py` as the driver template. Observation schema: `hal/client/data_structures/hardware.py`.
- Simulation: `hal/server/isaac/sim_rgbd_camera_cfgs.py`, `hal/server/isaac/main.py`, `hal/server/isaac/hal_server.py`, `hal/server/isaac/sensor_backend_isaac.py`.
- Teleop: `hal/server/teleop_portal_signaling.py`, `teleop/edge/`, the viewers under `teleop/portal/static/` and `fleet/portal/public/teleop/`, and `docs/TELEOP.md`.
- Recording and fleet: `data_collection/`, `docs/DATA_COLLECTOR.md`, `krabby/_locomotion_config.py`, `krabby/agent.py`, `fleet/infra/control_plane_stack.py`, `fleet/config/fleet.toml`.
- M7 owns HAL observations, the WebRTC protocol, and the rosbag recorder in `data_collection/` that Task 4 extends; M10 owns fleet identity, the MQTT channel, and the control plane, including the S3-upload-via-device-certificate pattern Task 4 is the first to use; M14 owns `krabby install`/`run` and image lifecycle.

## Looking ahead

The dataset the offloader accumulates is the raw material for the models that will use these cameras: foot-contact estimation to replace the zero-filled contact forces, and eventually a vision input to the locomotion policy, which is model training work rather than software work here. The current policy consumes only the 132-value depth scan from the front camera, so the new frames ride the observation bus unread until a checkpoint trained on them exists. The offloader also generalizes: once segments flow to S3, raising recording rates or adding future sensors is a configuration change.

## FAQ

1. Why the SEN0581 instead of the SEN0583 that M7's sensor memo selected, or an OAK-D Lite? Price and sufficiency. The SEN0583 is about $110 and the OAK-D Lite about $150, and both spend that on RGB and resolution that leg observation does not need. The SEN0581 delivers metric depth over exactly the right range for $30, with a serial protocol simple enough to drive without a vendor SDK, and its onboard LCD shows the live depth map, which speeds up bench debugging and aiming.

2. Why can't the new sim cameras be built like the existing sim depth cameras? The existing ones compute depth by casting rays against the terrain, which is cheap but means the robot's own body never appears in the image. These cameras exist to watch the legs, so they use Isaac's rendered cameras, which see the whole scene. Rendered cameras are normally avoided because they are expensive, but at 100x100 the cost is small, and they attach only for HAL and teleop runs, never during training.

3. Why don't the new cameras feed the locomotion policy? The policy's only camera input is a 132-value scan extracted from the front depth map. Its architecture has an image-input slot that is disabled and unimplemented on hardware (`vision_dims=()` in `compute/parkour/model_definition.py`). Feeding images in means retraining with an image encoder, which is a later milestone. This milestone guarantees the frames are present in the `HardwareObservations` the inference client receives every tick, so a future checkpoint can use them with no further plumbing.

4. Does H.265-encoding the depth lose information? Frames are quantized into the sensor's 0.15-1.5 m range before encoding. At 10-bit that is about 1.3 mm per step, under the sensor's stated accuracy of ≤1% of distance. The quantization range is recorded with the stream so playback recovers metric depth. Raw float32 is 800 KB/s per camera versus roughly 100 kbps encoded, which matters on the fleet's LTE and Wi-Fi uplinks.

5. Why does the offloader run on the host instead of inside the locomotion container? The device certificate lives at `/etc/krabby/iot/` root-owned on the host, and the container deliberately never sees it. The host already runs `krabby agent` with that identity; the offloader is a sibling worker reading the bags volume the container writes. The container stays stateless and credential-free.

6. Why belly mounts rather than the side-wall option the study also measured? The side-wall position drops the worst foot to 57-70% visibility while walking because the middle leg occludes it. Ground strikes are not a concern for the belly unit since its lowest point stays above the bottom of the hip plates. The side-wall variant is documented in the task file as the fallback.

7. What happens to the MaixSense side cameras? Their catalog entries and simulated counterparts are removed in Tasks 1 and 2, and the hardware comes off the robots. The driver code stays in the repo. The right-side MaixSense entry carried the optional side policy scan slot, and the active model configuration does not use a side scan (`num_side_scan=0` in `compute/parkour/model_definition.py`), so removing it changes nothing for the policy.

8. How much data does recording produce? At the default rates the front camera's RGB and depth encode to roughly 1.5 Mbps together, the three ToF cameras add about 0.45 Mbps, and the joint and IMU topics about 0.7 Mbps: roughly 20 MB per minute, or 1.2 GB per hour of driving. A 5-minute segment is about 100 MB, which the default 1 MB/s background upload clears in under two minutes, so the upload queue drains faster than recording fills it. Without compression the front camera alone would be 18 MB per second, which is why H.265 comes first.

9. Why restructure the sensor entries as named channels instead of just making the RGB field optional? The catalog holds different sensor types, and fixed rgb-plus-depth fields hardcode one of them, so every new type would force another schema change through transport and serialization. With channels declared by the sensor type, the code that moves and stores observations handles any sensor without knowing its format, and the consumers that interpret the data compose on type: the recorder picks a serializer per channel, teleop renders rgb directly and colorizes depth, and the policy scan reads the primary sensor's depth channel.
