# Task 1 — Simulated ToF cameras and depth-only HAL support

Goal: simulate the three SEN0581s in Isaac and pipe their depth through HAL into the observations the AI model receives.

Outputs

- Catalog sensor entries restructured as named channels declared by sensor type, with generic serialization and unit tests.
- Three Isaac cameras (`belly_left_tof`, `belly_right_tof`, `rear_tof`) at SEN0581 spec, publishing through HAL.
- An integration test that verifies the frames, and a walk script for confirming the legs are visible.
- Updated `docs/SENSOR_INTERFACE.md`.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-1---simulated-tof-cameras-and-depth-only-hal-support).)

Tenet for this rework: code that moves, stores, or validates observations must not know what a sensor's data looks like. Only the code that interprets the data (the teleop renderer, the recorder's per-channel serializers, the policy scan extraction) selects channels by sensor type. Judge every change in this task against that line: if a transport or serialization function grows a branch on sensor type, the design is wrong.

Line numbers below point at functions, not literal addresses; they drift as the code changes.

## 1. Sensor channels in the observation format

Every sensor in `rgbd_by_catalog_id` is currently a fixed pair of fields, `rgb` and `depth` (`RgbdCatalogObservation`, `hal/client/data_structures/hardware.py:35`). The catalog holds different sensor types, and these fields hardcode one of them. Replace the fixed fields with composition: an entry holds named channels (`channels: dict[str, np.ndarray]`, for the current sensors `"rgb"` and `"depth"`), and the catalog row's sensor type declares which channels it provides with their shape and dtype. A ZED row declares rgb and depth; a SEN0581 row declares depth; a future grayscale or radar row declares its own channel without touching the schema again. Rename the type to match (`SensorObservation` or similar) and keep `scan_features` as the one optional extra field, since it is policy output rather than sensor data. Legacy fields like `camera_rgb`/`camera_depth` stay as they are.

Code that moves or stores observations must not care what the channels are:

- `to_bytes` (around line 227) and `from_bytes` (around line 354) write and read each channel as name, dtype, shape, bytes in a loop. No per-sensor-type branches.
- Validation (`__post_init__`, around line 104) checks each channel is an array of the declared rank and dtype, generically.
- The ZMQ transport already treats the whole observation as bytes and does not change.

Only format-aware consumers compose behavior from the sensor type:

- The recorder (`data_collection/serialization.py:193`, `observation_to_writes`) maps channel kind to ROS serializer (`"rgb"` to an rgb8 Image, `"depth"` to a 32FC1 Image; Task 4 adds the H.265 path for depth). One topic per channel, `/camera/{id}/{channel}`, which is what `catalog_camera_topic` (line 173) already produces. Topic pre-registration (`catalog_camera_topic_msgtypes`, line 178, and `hal/server/rgbd_recording_catalog.py`) derives from each row's declared channels instead of assuming rgb and depth.
- The teleop poll worker (`hal/server/teleop_portal_signaling.py`, around line 384) reads the `"rgb"` channel for RGB streams and colorizes the `"depth"` channel for preview streams (Task 3).
- The policy scan extraction reads the primary row's depth channel, unchanged in behavior.

Extend `tests/test_hardware_serialization.py` and `tests/unit/data_collection/test_serialization_unit.py` with a sensor set that mixes rgb-and-depth entries with depth-only entries, asserting the round-trip and the per-row topic lists.

## 2. The three sim cameras

Match the sensor: 100x100 pixels, 20 fps, 70° horizontal by 60° vertical field of view, depth valid 0.15-1.5 m.

The existing sim depth cameras (`RayCasterCameraCfg` in `hal/server/isaac/sim_rgbd_camera_cfgs.py`) compute depth by casting rays against the terrain, so the robot's own body never appears in their images. These cameras point at the legs, so use Isaac's rendered camera instead: `CameraCfg` with `data_types=["distance_to_camera"]` and `clipping_range=(0.15, 1.5)`, following the shape of `_side_pinhole_rgb_cfg` (line 127) but requesting depth instead of RGB. Set the focal length and both apertures explicitly to hit 70°x60°; the pixel grid is square but the FOV is not, so the vertical aperture cannot be derived from aspect ratio. Rendered cameras are normally too expensive for training, but at 100x100 the cost is small and these attach only in HAL and teleop runs.

Poses, in the robot body frame: each belly camera sits halfway between the middle and rear hip plates along X, 6 cm in from the side edge in Y, about 3.5 cm below the belly surface in Z (2.5 cm of hang plus the 10 mm bracket), aimed 55° down, yawed 15° toward the front legs, facing outward from its side. The rear camera goes at the rear wall center, aimed rearward and 45° down. Derive the numeric offsets from the hexapod model the scene loads and commit the derivation as comments, the way `_SIDE_RIGHT_POS` is annotated (line 55). Build the rotations with the quaternion helpers already in the file. The mount study allows 50-60° of pitch and 10-20° of yaw; if the legs are not visible in the walk check (section 4), adjust within those ranges.

## 3. Wiring the cameras into the Isaac HAL server

The HAL transport is already generic: `rgbd_by_catalog_id` is a dict and takes any number of cameras. What is hardcoded is the Isaac-side wiring, which names the current cameras one by one in three places:

- `sim_rgbd_camera_cfgs_for_robot_link` (`sim_rgbd_camera_cfgs.py:176`) returns a fixed 6-tuple and `attach_sim_rgbd_sensors_to_scene_cfg` (line 273) assigns six named attributes.
- `hal/server/isaac/main.py` (around line 328) sets the update period on each of the six by name. The new cameras run at 20 fps against the existing 30, so each camera needs its own target rate here.
- `hal/server/isaac/hal_server.py` `set_observation` (around line 556) reads the three cameras in three copies of the same block.

The fix is a few lines of linking config, not an abstraction: one module-level tuple of entries, each naming a catalog id, its scene sensor attribute names, its clip range, and its frame rate, with the front camera pair and the three new cameras as the entries. The three places above iterate that tuple instead of naming cameras one by one. The robot body and its sensors are hardcoded by nature, so no registries, factories, or plugin layers around this; the tuple is the whole mechanism. Delete the simulated side cameras (the `side_right_*` and `side_left_*` configs and their ids): the SEN0581s replace the MaixSense cameras they mirror. The existing reader `_native_rgb_depth_pair` (line 110) already handles a rendered camera that outputs `distance_to_camera`; a ToF entry uses its depth half and emits an entry whose only channel is depth.

In `ISAAC_HAL_RGBD_CATALOG_IDS` (`hal/server/isaac/sensor_backend_isaac.py:30`), replace the side ids with the three new ones, and extend `_sensor_infos_from_scene_cameras` (line 92) so a depth-only camera lists only its `{id}_gray16_depth` preview stream with `depth_range_m=(0.15, 1.5)`.

Unit tests that pin the current three-camera set need extending: `tests/unit/hal/test_sensor_interface.py`, `tests/unit/hal/isaac_sensor_fixtures.py`, `tests/unit/hal/server/isaac/test_rgbd_recording_catalog.py`, and the camera paths in `tests/integration/test_isaacsim_hal.py`.

One gate to remove: cameras only attach when `--teleop-ip` is set (`main.py` around line 235). Add an explicit `--enable-hal-cameras` flag, implied by the teleop flag, so the integration test can run cameras headless.

## 4. Validation

Integration test: a new module under `tests/integration/`, running under the `test-isaacsim` make target (`Makefile:243`), following the client pattern in `tests/integration/test_isaacsim_hal.py`. Place a box at a known distance inside one belly camera's view, run the sim, subscribe with a `HalClient`, and assert each new camera delivers 100x100 depth frames at the expected rate with the box appearing at its known distance (within a few centimeters). This catches a wrong FOV, a flipped axis, a wrong clip range, or a camera on the wrong side.

Walk script: a small committed script (suggested `scripts/isaac/walk_tof_preview.py`) that runs the scripted forward walk the evaluation loops use while displaying or saving the three depth streams, so a person can confirm the legs are visible in each belly camera. Run it, confirm the legs show, and note the result in the task report. The mount study says each belly camera should see all three feet on its side nearly all the time; if it clearly does not, fix the pose before moving on.

## 5. Documentation

In `docs/SENSOR_INTERFACE.md`: the depth-only modality, the three new catalog rows with poses and ranges, the rule that depth-only cameras expose only a depth preview stream, and the new `--enable-hal-cameras` flag.
