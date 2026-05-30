# Task 5 — ZED 2i IMU through HAL into the model's body-frame state

**Time estimate: ~1.5 dev days (range 1–2).** Sub-table:

| Days | Sub-task |
|------|----------|
| 0.5 | `ZedCamera.get_imu()` — read `Camera.get_sensors_data()` from pyzed; return angular velocity / linear accel / orientation quaternion in a small dataclass |
| 0.5 | `JetsonHalServer` populates `HardwareObservations.base_ang_vel_b` and `base_quat_w` from `get_imu()`; mount-pose rotation applied (camera frame → robot body frame) |
| 0.5 | End-to-end verify: tilt robot by hand, watch roll/pitch in the mapper change; document the assumed mount pose; failure-mode behavior (no IMU sample) covered |

Goal: Plumb the ZED 2i's onboard IMU into HAL so the parkour model gets a live body-frame angular velocity and orientation. The current `hal/server/jetson/zed_camera.py` only exposes RGB + depth — the IMU is reachable through pyzed's `Camera.get_sensors_data()` but isn't wired anywhere. This task adds the read path, the fixed camera→body rotation, and the population of `HardwareObservations.base_ang_vel_b` and `base_quat_w` (the two required IMU fields the mapper at `compute/parkour/mappers/hardware_to_model.py` already consumes).

Outputs
- `ZedCamera.get_imu()` returning a small dataclass with angular velocity (rad/s), linear acceleration (m/s²), and orientation quaternion (xyzw).
- `JetsonHalServer.set_observation()` populates `base_ang_vel_b` and `base_quat_w` on every observation from the latest IMU sample.
- Camera→body rotation matrix loaded from a config (default identity); documented mount pose recorded alongside.
- Failure handling: missing IMU sample logs a warning and emits zeros (does not crash the loop).
- Verified end to end with the existing mapper — tilt the robot, watch the roll/pitch output change in the model's proprioceptive vector.

Acceptance Criteria
- **5a** — `ZedCamera.get_imu()` reads `Camera.get_sensors_data(..., TIME_REFERENCE.IMAGE)` once per observation tick and returns:
  - `ang_vel_rad_s`: `np.ndarray (3,)`, body-frame angular velocity in **rad/s** (pyzed returns deg/s; this method converts).
  - `lin_acc_m_s2`: `np.ndarray (3,)`, body-frame linear acceleration in m/s².
  - `orientation_quat_xyzw`: `np.ndarray (4,)`, IMU-integrated attitude.
  - Returns `None` if the sensors-data fetch fails (not a hard crash).
- **5b** — Exact pyzed API names (`get_sensors_data`, `get_imu_data`, `get_angular_velocity` / `get_linear_acceleration`, `get_pose().get_orientation()`) are verified against the locally installed pyzed version before merging — they're not guessed.
- **5c** — `JetsonHalServer.set_observation()` reads `ZedCamera.get_imu()` and writes `HardwareObservations.base_ang_vel_b` (3-vec, rad/s, body frame) and `base_quat_w` (4-vec, xyzw). The existing `__post_init__` validation in `hal/client/data_structures/hardware.py` (~L122–127) is satisfied for both fields.
- **5d** — A **camera-frame → robot-body-frame rotation matrix** is applied to `ang_vel` and `orientation_quat` before they're written to `HardwareObservations`. The matrix loads from a config (default identity for a flat mount; documented assumed pose recorded in the firmware/HAL config). M15 Task 2 will refine the mount pose against the actual camera location on the V0.2 chassis.
- **5e** — Missing IMU sample (e.g. pyzed `get_sensors_data` returns non-SUCCESS) logs a warning at WARNING level, increments a counter, and falls back to zeros (`base_ang_vel_b = [0,0,0]`, `base_quat_w = [0,0,0,1]`). Does **not** crash the control loop.
- **5f** — The existing mapper's roll/pitch extraction (`compute/parkour/mappers/hardware_to_model.py` `_extract_proprioceptive` ~L131–141) lights up with live values: tilt the robot ±45° and observe `proprioceptive[3]` (roll) and `proprioceptive[4]` (pitch) change accordingly. A short bench test log is captured for the deliverable.
- **5g** — Documentation: a short note in `hal/server/jetson/README.md` (or equivalent) covers the IMU path, the assumed mount pose, the camera→body rotation contract, and how to update the rotation when the mount changes.

---

## 1. Why ZED IMU and not the MCU IMU

M16 will add a BMI270 on the MCU as the canonical body-frame IMU. M17 predates M16 in dependency order (calibration + sim-to-real first), and M15 needs an IMU now. The ZED 2i has an onboard IMU (Bosch BMI088 in the ZED 2i specifically); pyzed exposes it through `Camera.get_sensors_data()`. Wire that path here as the M17 deliverable; when M16 lands, the BMI270 on the MCU becomes the canonical source and the ZED IMU stays as a sanity check / failover.

## 2. The pyzed sensors-data API

The ZED SDK Python bindings expose IMU via `Camera.get_sensors_data(SensorsData, time_reference)`. The relevant calls (verify against the installed pyzed version's headers before merging — the exact method names can change between SDK majors):

```python
import pyzed.sl as sl

sensors_data = sl.SensorsData()
status = self.camera.get_sensors_data(sensors_data, sl.TIME_REFERENCE.IMAGE)
if status != sl.ERROR_CODE.SUCCESS:
    return None

imu = sensors_data.get_imu_data()
ang_vel_deg_s = np.array(imu.get_angular_velocity())        # deg/s (verify units)
lin_acc       = np.array(imu.get_linear_acceleration())     # m/s²
quat_xyzw     = np.array(imu.get_pose().get_orientation().get())  # ZED integrates an attitude estimate
```

**Verify units before merging.** Stereolabs's docs say IMU angular velocity is in deg/s and acceleration in m/s² for the ZED 2i family, but pyzed has occasionally surfaced different conventions across versions. Print one sample on startup and confirm against a known-good source (e.g. a phone on the same table).

## 3. The dataclass and the read path

Add to `hal/server/jetson/zed_camera.py`:

```python
from dataclasses import dataclass

@dataclass
class ZedImuSample:
    ang_vel_rad_s: np.ndarray         # (3,), body-frame, rad/s
    lin_acc_m_s2: np.ndarray          # (3,), body-frame, m/s²
    orientation_quat_xyzw: np.ndarray # (4,), IMU-integrated attitude
    timestamp_ns: int

class ZedCamera(RgbDepthCamera):
    # ... existing init / grab path ...

    def get_imu(self) -> Optional[ZedImuSample]:
        if not self.initialized:
            return None
        sensors_data = self._zed_module.SensorsData()
        if self.camera.get_sensors_data(sensors_data, self._zed_module.TIME_REFERENCE.IMAGE) \
                != self._zed_module.ERROR_CODE.SUCCESS:
            return None
        imu = sensors_data.get_imu_data()
        ang_vel_deg = np.asarray(imu.get_angular_velocity(), dtype=np.float32)
        return ZedImuSample(
            ang_vel_rad_s=ang_vel_deg * (np.pi / 180.0),
            lin_acc_m_s2=np.asarray(imu.get_linear_acceleration(), dtype=np.float32),
            orientation_quat_xyzw=np.asarray(imu.get_pose().get_orientation().get(), dtype=np.float32),
            timestamp_ns=time.time_ns(),
        )
```

`get_imu()` is callable independently of `get_camera_frames()` — pyzed's sensors-data path doesn't need a fresh `grab()`, it has its own polling cadence (the ZED 2i IMU runs at 400 Hz; `TIME_REFERENCE.IMAGE` returns the sample aligned with the most recent frame).

## 4. Integration with `JetsonHalServer`

`JetsonHalServer.set_observation()` (in `hal/server/jetson/hal_server.py`) is what populates `HardwareObservations` every tick. Add the IMU read alongside the existing observation-population code, gated on having a ZED camera initialized:

```python
imu_sample = self._zed_camera.get_imu() if self._zed_camera is not None else None
if imu_sample is not None:
    ang_vel_body = self._camera_to_body_rot @ imu_sample.ang_vel_rad_s
    quat_body    = rotate_quaternion(self._camera_to_body_rot, imu_sample.orientation_quat_xyzw)
else:
    self._imu_miss_count += 1
    if self._imu_miss_count % 100 == 1:
        logger.warning("ZED IMU sample missing (count=%d)", self._imu_miss_count)
    ang_vel_body = np.zeros(3, dtype=np.float32)
    quat_body    = np.array([0, 0, 0, 1], dtype=np.float32)

obs.base_ang_vel_b = ang_vel_body
obs.base_quat_w    = quat_body
```

(`rotate_quaternion` is a small helper — quaternion-times-rotation-matrix-times-quaternion; reuse whatever's already in `compute/parkour/utils/math.py` if a comparable helper exists.)

## 5. Camera→body rotation (5d)

The ZED 2i is mounted on the robot at some pose — most likely forward-facing and pitched slightly down. The IMU's coordinate frame is the camera's, not the robot's. Apply a fixed `R_camera_to_body` rotation to `ang_vel` and `orientation_quat` before they're written.

Default config: identity (camera frame = body frame; only correct if the camera is rigidly axis-aligned with the body).

Store the matrix in a config alongside `hal/server/jetson/` settings (or in a YAML the HAL server loads at startup). Document the assumed mount pose in plain text:

```yaml
# hal/server/jetson/config/zed_mount.yaml
# Assumed pose: ZED 2i mounted at the front-center of the krab body,
# camera Z-axis (forward) aligned with robot +X (forward), camera Y-axis (up)
# aligned with robot +Z (up), camera X-axis (right) aligned with robot -Y.
# Update this matrix if the camera mount changes.
r_camera_to_body:
  - [1, 0, 0]
  - [0, 0, 1]
  - [0, -1, 0]
```

M15 Task 2 will refine this against the actual mount on the V0.2 chassis (it also moves the sim camera to match — same alignment work). M17 just lays the plumbing.

## 6. Failure modes (5e)

Three things to handle:

- **ZED not initialized** (camera missing or pyzed import failed) — `JetsonHalServer` already handles this for the camera path (graceful degradation to mock); extend the same fall-back to IMU (zeros + warning).
- **`get_sensors_data` returns non-SUCCESS** — log at WARNING with rate-limiting (`% 100` so we don't flood); emit zero observations.
- **Stale samples** — if the IMU timestamp doesn't advance across consecutive observation ticks, log once at INFO and continue. Don't try to interpolate — the model treats zero ang vel + identity quat as "robot is stationary" which is the safe default.

## 7. Bench-without-chassis

This task is bench-friendly: the ZED only needs to be plugged into the Orin via USB-C and powered. No chassis required to verify the IMU comes through HAL. Tilt the camera (or the bench it sits on) by hand and watch the mapper's `proprioceptive[3:5]` (roll/pitch) change. Save a short log of the bench tilt for the deliverable.

Final mount-pose tuning needs the chassis but is M15 Task 2's job.

## 8. Verify
- **API parity:** run a one-off script that imports pyzed, calls `get_sensors_data`, and prints one IMU sample. Confirm units, signs, and frame convention before plumbing into HAL.
- **HAL path:** with `krabby run --gamepad-only` or any HAL-up entry point, log one `HardwareObservations` and confirm `base_ang_vel_b` is non-zero when the camera is moved, and `base_quat_w` is a normalized 4-vec.
- **Mapper path:** with the model loaded (`--control-source inference`), enable DEBUG logging on the mapper and confirm `proprioceptive[3:5]` reflects roll/pitch with the right sign.
- **Failure path:** unplug the ZED mid-run; confirm the loop continues, the warning fires, and observations carry zeros.

## 9. Deliverable checklist
- [ ] `ZedCamera.get_imu()` returns a validated `ZedImuSample` with rad/s + m/s² + xyzw quat.
- [ ] Pyzed API names verified against the installed version (don't guess).
- [ ] `JetsonHalServer.set_observation()` populates `base_ang_vel_b` + `base_quat_w` every tick.
- [ ] Camera→body rotation loaded from config; default + assumed mount pose documented.
- [ ] Missing-IMU fall-back emits zeros and a rate-limited warning; no crash.
- [ ] Bench tilt test produces visible roll/pitch in the mapper output; log captured.
- [ ] `hal/server/jetson/README.md` updated with the IMU path + mount-pose contract.
