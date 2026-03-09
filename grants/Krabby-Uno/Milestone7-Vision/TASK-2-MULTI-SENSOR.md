# Task 2 — Multi-sensor hardware summary (for Bruce)

This document summarizes sensor research for Milestone 7 vision integration and records the **chosen side/rear hardware**: three DFRobot SEN0583 RGB-D ToF cameras (sides + back).

---

## Chosen hardware: DFRobot SEN0583 × 3 (sides + back)

**Decision:** Use **DFRobot RGB-D 3D ToF Sensor (SEN0583)** for side and rear coverage. Purchase **three units** (left, right, rear).

| Spec | Value |
|------|--------|
| **Product** | DFRobot SEN0583 — RGB-D 3D ToF Sensor Camera |
| **Depth** | 320×240 ToF, **0.2–2 m** indoor range |
| **RGB** | Integrated color camera |
| **Interface** | Linux, ROS1/ROS2, Python SDK |
| **Price** | ~**$110** per unit |
| **Quantity** | **3** (left side, right side, rear) |

**Rationale**

- **Decent camera:** Integrated RGB + depth in one device; no separate RGB + radar wiring for basic obstacle coverage.
- **2 m ToF range:** Sufficient for object avoidance and side/rear awareness at crab speeds; no need for 5 m on sides/back for this milestone.
- **$110 each:** Low enough to buy three for full perimeter coverage (~$330 total) while staying under the cost of a single higher-end RGB-D (e.g. CS30 ~$299 for one).
- **Easy to get 3x:** Same part, same driver/SDK; simplifies HAL backend (one RGB-D pipeline × 3).

**Deployment**

- Front: ZED 2i (Task 1) — primary stereo RGB-D.
- Left / right / rear: one SEN0583 each, wired over USB (ensure USB hub and bandwidth are documented in SETUP.md when integrating).

---

## Research summary: RGB cameras (reference)

For **inexpensive RGB-only** options (if needed elsewhere or as fallback):

- **8MP USB camera (IMX219, Jetson/Pi-ready):** ~$48–55, UVC plug-and-play, 3264×2448, 30 fps at 1080p. Good default for “cheap RGB” on Jetson.
- **Logitech C920/C922:** 1080p @ 30 fps, ~$60–80, very well supported UVC.
- **ELP 3MP wide-angle:** ~$64–74, WDR, wide FOV, UVC; mentioned in OVERVIEW Option B for side RGB.

**GMSL2:** Not recommended for M7. Higher cost and integration effort; UVC USB is sufficient for current side/rear and bandwidth goals.

---

## Research summary: Radar (reference)

If you add **radar** later (e.g. for low-light or redundant range), use a sensor that gives **generic obstacle/range data**, not “human presence only.”

### Avoid: “Human presence” radar modules

Modules such as **LD2410**, **LD2410C**, and DFRobot’s **24 GHz “human presence”** radar:

- Run onboard firmware tuned for **human** signatures (e.g. breathing, micro-motion).
- Output only high-level results (e.g. “human at 2.5 m” or “no human”).
- **Do not** provide raw ADC, range bins, or a generic “something at X m” list.

**Implication:** A dog, baby, pole, or box may not trigger detection or may be misclassified. You cannot implement “anything in range = obstacle” yourself.

### Prefer: Object-agnostic range / obstacle data

**Acconeer A121 (60 GHz pulsed coherent)**

- **Range profile** (amplitude vs distance) and/or **obstacle detection** in the SDK — **object-agnostic** (“obstacle at X m”), not “human at X m.”
- **Moving platform:** Designed for use on a moving robot; supports `update_robot_speed()` for better separation of static vs moving obstacles.
- **Interface:** USB via XC120 + XE121 eval; ~\$120.
- **Use case:** Generic collision avoidance (dog, pole, baby, box).

**Texas Instruments IWR6843 (77 GHz FMCW)**

- Onboard processing outputs **point clouds / target lists** (range, angle, velocity) for **any** strong reflector, not just humans.
- Commonly used on moving platforms (e.g. cars, robots); detections are in radar frame.
- **Cost:** ~\$160–230 (EVM). Raw ADC (DCA1000) not needed for obstacle avoidance.

**Takeaway:** If you add radar, choose A121 or TI IWR-style devices that expose range/obstacle or target list; avoid LD2410-style “human presence” modules for generic obstacle detection.

### Moving platform (crab robot)

Both Acconeer A121 and TI IWR6843 are suitable for **mounting on the moving crab**:

- A121: Obstacle detection app is explicitly for “moving platform like a robot”; feed robot speed for best angle/velocity interpretation.
- TI IWR: Standard for vehicle/robot mounting; detections are relative to the radar. Vibration on the body is generally acceptable; mounting on the body (not leg) is recommended.

---

## Task 2 integration notes

- **HAL:** Extend the multi-sensor interface (Task 2) so that **three SEN0583** devices are listed as side_left, side_right, rear (or equivalent IDs). Reuse the same RGB-D backend pattern (e.g. DFRobot SDK / V4L2 or vendor API) for all three.
- **GStreamer:** One pipeline (or pipeline template) per SEN0583; same encoding and resolution constraints as in OVERVIEW (bandwidth/power when streaming all sensors).
- **Isaac:** Add three synthetic RGB-D sensors in sim with similar FOV/range (0.2–2 m) and pose to match left, right, rear mounting.
- **ROSBAG2 / WebRTC:** Include side_left, side_right, rear in topic list and stream list (e.g. `/camera/side_left/rgb`, `/camera/side_left/depth`, etc.) as in Task 3/4.

---

## Checklist

- [ ] Confirm USB topology: three USB devices (plus ZED 2i) on Jetson; document hub and bandwidth in SETUP.md.
- [ ] Install DFRobot SDK/driver on Jetson; verify one SEN0583 delivers RGB + depth at target resolution/fps.
- [ ] Add SEN0583 backend to HAL (list_sensors, get_gstreamer_handle, pipeline generation for left/right/rear).
- [ ] Add Isaac synthetic equivalents for the three SEN0583s; validate multi-sensor stream (ZED front + 3× SEN0583) in Task 2.
