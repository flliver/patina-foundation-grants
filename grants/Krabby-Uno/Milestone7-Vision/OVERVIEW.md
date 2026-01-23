# Patina Foundation Grant - Krabby-Uno Milestone 7: Vision Integration

## Grant Overview
Integrate ZED 2i stereo cameras into Krabby's HAL stack for both local image access (on-robot models) and remote streaming (WebRTC teleoperation). Extend HAL observations to include RGB/depth frames, build a GStreamer-based multi-camera interface, implement data collection for training pipelines, and enable remote teleoperation with video feeds and joystick control over WebRTC. All components work with both Jetson (real hardware) and IsaacSim (synthetic cameras).

## Why is this Important?
- Establishes the vision pipeline foundation: cameras → HAL → client, so downstream models (navigation, VLMs) can consume images by simply fetching from HAL observations.
- Enables remote teleoperation with live camera feeds: operators view WebRTC streams while controlling the robot from anywhere.
- Provides data collection infrastructure for training future perception/navigation models from real-world robot experience.
- Validates multi-camera bandwidth, encoding, and streaming before adding compute-heavy inference.
- Keeps the stack ROS-free while still enabling ROSBAG2 format for data portability with the broader robotics ecosystem.

## Tasks
### Task 1 - Single camera HAL integration (RGB + Depth observations)
#### Narrative
Extend the HAL server and client to support camera observations. On Jetson, integrate ZED SDK to capture RGB and depth frames from a single ZED 2i camera and populate new image fields in `KrabbyHardwareObservations`. On Isaac, configure a synthetic camera in the sim scene and wire it through the Isaac HAL server. HAL clients should be able to fetch the current RGB and/or depth frame via the standard observations interface, enabling on-robot models to access camera data without special plumbing.

This task includes: ZED SDK installation and configuration on Jetson, camera config (resolution, fps, depth mode), extending the observations struct, updating both HAL servers (Jetson and Isaac), updating the HAL client to receive/expose images, and validation scripts that display what the camera sees.
#### Acceptance Criteria
- `KrabbyHardwareObservations` extended with `camera_rgb` and `camera_depth` fields (single camera); documented format (resolution, encoding, timestamp).
- Jetson HAL server: ZED SDK integrated, captures RGB + depth at 15+ Hz, populates observation fields; 5-minute stability run without dropped frames.
- Isaac HAL server: synthetic camera added to sim scene matching real camera position/FOV, populates same observation fields.
- HAL client: `get_observations()` returns image data; example script displays RGB and depth frames from both Jetson and Isaac backends.
- `SETUP.md`: ZED SDK install, camera config, udev rules, troubleshooting; Isaac camera setup documented.
- Unit tests for observation serialization/deserialization with image data; 80%+ coverage on new code.

### Task 2 - GStreamer multi-camera interface (4x cameras)
#### Narrative
Define a new GStreamer-based sensor interface in HAL that supports multiple cameras (up to 4x ZED 2i). The interface should allow clients to: (1) list available sensors/cameras, (2) request a GStreamer handle for a specific sensor, and (3) generate a GStreamer pipeline that outputs H.264 encoded video from that camera. This abstracts the camera access so the same interface works with Jetson (real ZED cameras) or Isaac (synthetic cameras rendered to GStreamer-compatible buffers).

This task includes: designing the sensor listing API, implementing GStreamer pipeline generation for ZED SDK sources (with Jetson HW encoding via nvenc), implementing equivalent pipeline generation for Isaac synthetic cameras, handling multi-camera USB bandwidth on Jetson (resolution/fps tradeoffs, hub assignments), and validation that all 4 cameras can stream H.264 simultaneously.
#### Acceptance Criteria
- New HAL sensor interface: `list_sensors()` returns available cameras with metadata (id, type, resolution, fps); `get_gstreamer_handle(sensor_id)` returns a handle for pipeline construction.
- `generate_pipeline(handle, encoding='h264')` returns a GStreamer pipeline string that outputs encoded video; works for both Jetson (ZED + nvenc) and Isaac (synthetic + software encode).
- Jetson: 4x ZED 2i cameras stream H.264 simultaneously at 10+ fps each; documented USB hub/port assignments and bandwidth tradeoffs.
- Isaac: 4 synthetic cameras configured in sim, each produces H.264 via the same interface; camera positions match real hardware layout.
- Example script: lists sensors, generates pipelines for all 4, launches them, and displays H.264 decoded output in a grid; works on both backends.
- Documentation: sensor interface API, pipeline generation options, multi-camera bandwidth considerations, Isaac camera setup.

### Task 3 - Data collection agent (ROSBAG2 recording)
#### Narrative
Build a data collection agent that runs inside existing Isaac/Jetson Docker container, connects to the HAL client, pulls image observations (and optionally joint states, IMU, commands), and writes them to ROSBAG2 format on the device. The agent should rotate bag files every X minutes, and clean up oldest files when approaching a configured max size (default: 50% of Orin disk space), ensuring continuous recording without filling the disk. This enables collecting training data from real-world robot operation for future perception/navigation model training.

This task includes: Docker image setup with ROSBAG2 dependencies (without full ROS install if possible, or minimal ROS for mcap/rosbag2 libraries), HAL client integration for pulling observations at configurable rate, ROSBAG2 writer with configurable topics (images, joints, IMU, commands), disk space monitoring and automatic rotation, and validation on both Jetson and Isaac backends.
#### Acceptance Criteria
- Docker image (`krabby-data-collector`) with ROSBAG2 writing capability; documented Dockerfile and build instructions.
- Agent connects to HAL client, pulls observations at configurable rate (default 10 Hz for images, 50 Hz for joints/IMU).
- Writes to ROSBAG2 format (mcap) with configurable topics; default includes: `/camera/rgb`, `/camera/depth`, `/joints/state`, `/joints/command`, `/imu`.
- Disk monitoring: rotates to new bag file when current exceeds configured size; default max total usage is 50% of disk; oldest bags deleted when limit reached.
- Config file (YAML) for: HAL server address, recording rate, topics, max disk usage, bag rotation size.
- Works with Jetson (real cameras) and Isaac (synthetic); 30-minute recording test on each without disk overflow or dropped frames.
- Playback validation: recorded bags can be read with standard ROSBAG2 tools; example script replays and displays recorded images.

### Task 4 - WebRTC streaming agent
#### Narrative
Build a WebRTC agent that runs on the Jetson (or Isaac host), connects to a remote server app via URL, and establishes WebRTC connections to stream 1–4 camera feeds as requested by the server. The server app controls which cameras to stream and at what quality; the agent responds to these requests and maintains the WebRTC connections. This enables remote operators to view live camera feeds from anywhere with internet access.

This task includes: WebRTC client implementation (using aiortc or GStreamer webrtcbin), signaling protocol to connect to server app and negotiate streams, dynamic stream management (server can request/release individual camera streams), H.264 encoding pipeline integration (reuse Task 2 GStreamer handles), and validation with a simple test server that requests different camera combinations.
#### Acceptance Criteria
- WebRTC agent runs as a service on Jetson Docker image; connects to configurable server URL on startup.
- Signaling protocol: agent registers with server, server can request streams (camera IDs, resolution, fps), agent establishes WebRTC peer connections and streams H.264 video.
- Supports 1–8 simultaneous streams as requested by server (4x RGB, 4x depth); streams can be added/removed dynamically without restarting agent.
- Latency: <300ms glass-to-glass for single stream; <500ms for 4 streams; documented measurement method.
- Test server app (Python/Node): accepts agent connections, provides UI to request/view streams, displays latency stats.
- Works with Jetson (real cameras) and Isaac (synthetic); 10-minute streaming test on each without disconnects.
- Has reasonable degradation and QoS when insufficient bandwidth to stream all streams
- Documentation: signaling protocol spec, agent config, server app setup, firewall/NAT considerations.

### Task 5 - WebRTC joystick teleoperation (WebRTCInputController)
#### Narrative
Extend the WebRTC connection to support bidirectional control: the server app sends joystick/gamepad commands down to the robot, and a new `WebRTCInputController` receives these commands and feeds them into the ControlLoop (same interface as the local `InputController` from M6). This enables full remote teleoperation: operator views camera feeds and controls the robot from a browser, with commands flowing over the same WebRTC connection as the video (or separate depending on async vs sync command requirements for webRTC, do what makes sense).

This task includes: extending the signaling/data channel protocol for control commands, implementing `WebRTCInputController` that mirrors the `InputController` interface, wiring it into ControlLoop as an alternative input source, server app UI for gamepad input (browser Gamepad API or virtual joystick), and end-to-end validation with remote operator driving the robot while viewing camera feeds.
#### Acceptance Criteria
- WebRTC data channel added for control commands; protocol supports the same command struct as local `InputController` (leg selection, hip/knee/yaw axes).
- `WebRTCInputController` implements same interface as `InputController`; can be selected in ControlLoop config (`--controller webrtc`).
- Server app: captures gamepad input (browser Gamepad API) or provides virtual joystick UI; sends commands over WebRTC data channel at 50+ Hz.
- Command latency: <100ms from server input to robot HAL command; documented measurement.
- End-to-end test: operator on remote machine views 4 camera feeds and drives robot via browser gamepad; robot responds to commands; 5-minute session without disconnects.
- Works with Jetson (real robot) and Isaac (sim); video capture of remote teleoperation session on each.
- Documentation: control protocol spec, server app gamepad setup, ControlLoop config for WebRTC input, latency tuning.

## Information
- Hardware: 1–4x ZED 2i stereo cameras, Jetson Orin, USB 3.0 hubs (for multi-camera bandwidth).
- SDK: ZED SDK for Jetson; Python bindings (`pyzed`).
- Streaming: GStreamer with nvenc (Jetson HW encoder); aiortc or webrtcbin for WebRTC.
- Data format: ROSBAG2 (mcap) for data collection; enables compatibility with ROS ecosystem tools without running ROS.
- HAL: extend `krabby-research/hal/` for image observations and sensor interface.
- Isaac: synthetic cameras configured to match real camera positions/FOV; same HAL interface for both backends.

## FAQ
- **Why ROSBAG2 without ROS?**  
  ROSBAG2 uses mcap format which can be written/read without a full ROS install. This gives data portability with the ROS ecosystem for training pipelines while keeping the runtime stack ROS-free.
- **How does the server app work?**  
  The server app is a simple web application that the WebRTC agent connects to. It handles signaling, displays video streams, and captures gamepad input. It can run on any host with internet access. It will eventually run on EC2, but for now keep it simple, an app accepting basic HTTPS on a configured port to get webRTC over a second port.
- **Can I test without 4 cameras?**  
  Yes. The interface supports 1–4 cameras; start with 1 for development, scale to 4 for full validation.
- **How do on-robot models get images?**  
  Via HAL observations (`get_observations().camera_rgb`). The GStreamer/WebRTC path is for remote viewing; HAL observations are for local inference.
- **What about depth in WebRTC?**  
  Depth can be encoded as colorized video for visualization. Raw depth for inference should use HAL observations, not WebRTC.
- **How do I get sufficient bandwidth to test four cameras from one USB 3.0 bus?**
  Honestly you don't. I think we'll need either a different box that has 4x GSML port, but in that case the box costs like $2k instead of $1k and needs the more expensive $1k/pop Zed2 GSML cameras, and suddenly it needs 80% of the compute on the box just to do the depth calcs. Or, more likely we move to another much cheaper/lower bandwidth camera like intellisense or DFRobot CS30. I will buy some cameras for us to test bandwidth on. If we still don't like what we see, we'll switch to low res monocular and add one or more dome lidar. 
