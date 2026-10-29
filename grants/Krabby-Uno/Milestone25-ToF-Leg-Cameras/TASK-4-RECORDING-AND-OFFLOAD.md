# Task 4 — Recording and S3 offload

Goal: record the ToF depth streams as H.265 inside the existing rosbag segments, ship the data collector on fleet robots for the first time, and build the offloader that moves closed segments to S3 under the robot's fleet identity and reclaims the disk.

Outputs

- An H.265 depth recording path for depth-only catalog rows, with a playback script recovering metric depth.
- Per-catalog-id record rates in the collector configuration.
- The `krabby-data-collection` wheel published and installed in the release locomotion image, with the bags volume and collector flag wired through the fleet launcher.
- `krabby/offload.py`: a host-side worker uploading closed segments via the IoT credential provider, verifying, deleting, resuming, and rate-limiting.
- CDK: a bags bucket and upload role grant in the control plane stack, with configuration in `fleet/config/fleet.toml`.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-4---recording-and-s3-offload).)

Line numbers below point at functions, not literal addresses; they drift as the code changes.

## 1. H.265 recording

The collector (`data_collection/collector.py`) is a second `HalClient` in the HAL process writing rosbag2/MCAP segments via `data_collection/serialization.py` and `data_collection/rotating_bag.py`. Today every catalog entry records as raw `sensor_msgs/Image` (`rgb8` and `32FC1`). Add an encoded path per channel kind: an rgb channel feeds a GStreamer H.265 appsrc pipeline directly, and a depth channel is first quantized into the row's `gst_depth_quant_range_m` (0.15-1.5 m for the ToF rows; 0.2-25 m for the front camera, about 24 mm per step at 10-bit, fine for logging). Encoded frames are written as `sensor_msgs/CompressedImage` messages with `format` carrying the codec and, for depth, the quantization range (for example `hevc; range=0.15,1.5`), on `/camera/{id}/{channel}_h265` topics. This covers the front camera's RGB and depth and the three ToF depth streams. The encoder is `nvv4l2h265enc` on the Jetson and `x265enc` elsewhere; the hardware/software split already exists for the streaming pipelines in `hal/server/jetson/sensor_backend_jetson.py` `build_pipeline` and `hal/server/streaming_map.py`, and the encoder should emit a keyframe at least once per second so each recording segment decodes on its own. Choose 10-bit (about 1.3 mm per step over the range) if the Jetson encoder path cooperates, otherwise 8-bit (5.3 mm per step) is inside sensor noise past half a meter; record the choice and reasoning in the code.

A committed playback script, `data_collection/tools/decode_h265.py`, reads a bag, decodes a chosen camera's stream, and returns RGB frames directly or reconstructs float32 meter frames from the depth format metadata, asserting depth round-trip error stays within the quantization step. The existing raw float path remains available behind the per-camera config for bench work.

Encoding happens on the collector's thread at the recording rate (three 100x100 streams at 20 fps plus the front camera's two streams at 10 Hz, light work for nvenc), never on the 100 Hz control path. Change the segment length default to 5 minutes (`data_collection/collector_settings.py`, currently 250 MiB or 30 minutes per segment). At the default rates a segment is about 100 MB: roughly 1.5 Mbps for the front camera's RGB and depth, 0.45 Mbps for the three ToF cameras, and 0.7 Mbps of joint and IMU topics. That size makes each segment a quick upload unit and bounds what a crash can lose. The 250 MiB size cap stays as the safety bound, and the delete-oldest quota at 10% free disk stays.

## 2. Per-camera record rates

`RecordingRates` in `data_collection/config.py` has one global `images_hz` (default 10). Replace it with a per-catalog-id rate map with a default rate, with committed defaults of 20 fps for the ToF rows and 10 Hz for the front camera's RGB and depth. The YAML shape in `data_collection/config/collector.yaml` and the committed defaults in `collector_settings.py` follow. The per-id stream lists from Task 1's recording catalog say which topics each id registers, so a depth-only id pre-registers only its `depth_h265` (or raw depth) topic. Update `tests/unit/data_collection/` for the config shape and `tests/integration/test_data_collector_bag.py` for a bag mixing raw and H.265 topics.

## 3. Shipping the collector on fleet robots

Nothing records on a fleet robot today: `images/locomotion/requirements.release.txt` excludes `krabby-data-collection`, and the fleet launcher never passes the collector flag. What is missing:

- Publish the wheel from `.github/workflows/publish-packages.yml` like the other `krabby-*` packages, and pin it in `requirements.release.txt` (`docs/PUBLISHING.md` has the release process).
- `krabby/_locomotion_config.py` `build_hal_argv` (line 132) gains `--data-collector-output-dir /data/krabby_bags` when a `data_collector` key in `/etc/krabby/locomotion.json` enables it, and `fleet_volume_mounts` (line 174) gains the bags mount from a host directory (suggested `/var/krabby/bags`) created by the launcher.
- The collector rides the HAL entrypoint (`hal/server/jetson/runtime.py`, `--data-collector-output-dir` handling around line 49), so no new service is involved; `docs/JETSON_DEPLOYMENT.md` (bags notes around line 316) and `docs/DATA_COLLECTOR.md` document the fleet path.

## 4. The offloader

New module `krabby/offload.py`, running as a worker under the host-side `krabby agent` (`krabby/agent.py`), which already owns the device identity and starts at boot via the systemd unit `krabby enroll` installs. The worker:

- Scans the bags directory for closed segments (the rotating writer's completed segment directories; the active segment is excluded).
- Obtains temporary AWS credentials from the AWS IoT credential provider: an HTTPS request with the device certificate at `/etc/krabby/iot/` as client TLS, against the account's credential-provider endpoint, for the bags role alias. The endpoint is discovered once at enroll time (`iot describe-endpoint --endpoint-type iot:CredentialProvider`) and stored next to the other enroll outputs in `krabby/enroll.py`'s state. This is the mechanism the control plane already grants for image upload (`fleet/infra/control_plane_stack.py`, the `AssumeImageUploadRole` policy statement around line 112) and that no device code has used yet.
- Uploads each segment with boto3 multipart under `s3://<bags bucket>/<thing-name>/<YYYY-MM-DD>/<HHMMSS>_<seq>/`, taking the date and time from the segment's start timestamp, so one robot's day of driving lists together in order. Verifies object presence and size against the local files, then deletes the local segment. A small JSON state file in the bags directory tracks in-progress uploads so a crash or network loss resumes rather than restarts, and multipart uploads are aborted or resumed on recovery.
- Rate-limits to a configured ceiling, default 1 MB/s: roughly three times the ~340 KB/s the recorder produces, so the queue drains while staying gentle on the Wi-Fi and LTE uplinks fleet robots ride. Pauses while a teleop session is active so uploads never compete with live video; the agent already brokers teleop signaling (`krabby/teleop_shim.py`), so it knows when a session is open. Retries with backoff on failures. The worker reads only finished segments, never the one being written, so recording and control are never interrupted. MQTT is never used for payload since AWS IoT caps messages at 128 KiB, though a one-line shadow or log entry noting the last successful upload is cheap and useful for the portal.

Configuration (bucket name, role alias, rate ceiling, enable flag) comes from `fleet/config/fleet.toml` via the same path other fleet settings reach the device, with the enable flag defaulting off until a robot opts in.

## 5. Control plane

In `fleet/infra/control_plane_stack.py`, alongside the existing `KrabReportedImages` pattern (bucket, upload role, role alias, policy statement, around line 277): a `KrabBags` bucket (block-public, SSE-S3, SSL-enforced, `RemovalPolicy.RETAIN`, Intelligent-Tiering lifecycle and no expiration, since these are training data), an upload role trusted by `credentials.iot.amazonaws.com` scoped to `s3:PutObject`/`s3:AbortMultipartUpload`/`s3:ListMultipartUploadParts` on the bucket, a role alias, and a device policy statement permitting `iot:AssumeRoleWithCertificate` on it. Export the bucket name as a CfnOutput and record it in `fleet.toml`. The e2e pattern in `fleet/infra/tests_e2e/test_control_plane_e2e.py` covers the existing resources; extend it for the new ones.

Scoping note: the device policy template in M10 confines each robot to its own topics; the upload role is shared, so prefix discipline (`<thing-name>/...`) is by convention in the worker. If per-thing prefix enforcement is cheap via the credential provider's thing-name condition keys, take it; if it fights, note the decision and move on; robots are trusted peers in this fleet.

## 6. End-to-end verification

On a fleet-enrolled robot with the collector and offloader enabled: drive for ten minutes (teleop or gamepad), then confirm segments appear in S3 under the robot's thing name, play back with the decode script and standard MCAP tooling, and the local bags directory has been emptied of uploaded segments while staying under quota throughout. Pull the network mid-upload once and confirm the segment uploads completely after reconnection. Record the observed upload throughput and the steady-state disk high-water mark in the task report.
