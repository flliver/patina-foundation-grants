# Task 3 — Depth pipeline from ZED to deployed policy

Goal: Get the depth observation path working end-to-end on hardware. Write a HAL function that takes the front-facing ZED's native depth output and converts it to the exact tensor format the M2 student model was trained against — mirroring the existing Go2 / Extreme Parkour on-robot depth pipeline as closely as possible — then deploy the retrained Task 2 image with the depth pipeline live and validate that the policy reacts to real-world depth changes.

Acceptance Criteria
- **3a** — M2 student depth-input spec documented in the relevant repo README: resolution, channel layout, encoding (meters / inverse / normalized), FOV, near/far plane, any clipping or normalization the sim does. Pulled from the sim training config, not guessed.
- **3b** — Existing Go2 / Extreme Parkour on-robot depth pipeline studied; the krab's HAL depth-conversion function mirrors its structure (same downsample method, same crop strategy, same invalid-pixel handling) with only the parameters that have to change (ZED-vs-D435i FOV/resolution, hexapod-vs-Go2 camera pose) actually changed. Deviations from the Go2 reference are documented with a reason.
- **3c** — HAL depth-conversion function implemented and committed; produces the documented M2 student tensor format; runs on the inference path within the latency budget.
- **3d** — Sim and real are doing the *same* depth pre-processing: if the sim clips/normalizes, the real pipeline does the same clip/normalize; if the sim doesn't, the real pipeline doesn't either. The ZED is a nicer camera than the Go2's D435i and probably doesn't need clipping for its own sake; the rule is parity with sim, not what the camera could tolerate.
- **3e** — Bench test: a recorded ZED depth frame run through the HAL function produces a tensor that matches a sim-rendered depth frame of an equivalent scene within documented tolerance; data and result captured in the relevant README.
- **3f** — New container image built with Task 2 weights and the Task 3 depth pipeline; deployed to the real krab via fleet portal if available, otherwise `krabby-launcher`.
- **3g** — On the deployed robot, the depth tensor reaching the policy matches the M2 student depth format; verified by logging the tensor shape/range from inside the policy container.
- **3h** — Policy reactivity test: holding an obstacle in front of the robot at varying distances produces a corresponding change in the policy's command output; clip or log captured.

---

**NOTE**: All code examples in this document are design guidance. Treat them as design guidance, not production-ready code. Actual implementation may require adjustments based on the ZED SDK version, the HAL transport, and the M2 student observation schema.

---

## 1. Document the M2 student depth-tensor format

Before writing any conversion code, document **exactly** what the M2 student model expects on its depth input. This is the source of truth that the HAL function targets. If this is wrong, everything downstream is wrong.

Pull the information from the M2 student training config in `krabby-research/parkour/` (the depth-observation term in the policy config). Record:

| Field | Value | Where it's defined |
|---|---|---|
| Resolution (H × W) | (e.g. 58 × 87) | depth sensor config |
| Channel layout | (e.g. single-channel float32) | observation schema |
| Depth encoding | (m / inverse-depth / normalized [0,1] / clipped-and-scaled) | observation pre-processing |
| Near plane | (e.g. 0.05 m) | sensor config |
| Far plane | (e.g. 5.0 m) | sensor config |
| Horizontal FOV | (e.g. 87°) | sensor config |
| Vertical FOV | (e.g. 58°) | sensor config |
| Optical-axis orientation | (degrees from body forward, pitch/yaw) | sensor config (matched to Task 2 mount) |
| Update rate | (e.g. 10 Hz, 30 Hz) | observation update term |
| NaN/invalid handling | (e.g. clamped to far plane, set to 0, masked) | pre-processing |

If the M2 student was trained on a depth representation that's been already pre-processed (e.g. clipped to [0.2, 2.0] m then normalized to [0, 1]), the conversion function must produce that *same* pre-processed representation, not raw meters.

Stash the table at `krabby-research/docs/depth-format-m2.md`. This document is also what the M2 ZED mount in Task 2 (Section 5) should have been matched to in terms of FOV and orientation — cross-check.

---

## 2. Characterize the ZED native depth output

The ZED SDK publishes depth at the camera's native resolution. Document what arrives on the HAL bus from the ZED bridge:

| Field | Value | Notes |
|---|---|---|
| Resolution (H × W) | (e.g. 720 × 1280 in HD720 mode) | configurable in the ZED bridge |
| Output mode | (e.g. depth in meters, float32) | per ZED SDK setting |
| Near plane | (e.g. 0.3 m) | ZED minimum-depth setting |
| Far plane | (e.g. 20 m) | ZED maximum-depth setting |
| Horizontal FOV | (e.g. 110° in HD720 wide mode) | per camera + mode |
| Vertical FOV | (e.g. 70°) | per camera + mode |
| Invalid pixels | (typically NaN or 0) | ZED returns NaN for missing depth |
| Publish rate | (e.g. 30 Hz, 60 Hz) | configurable |
| HAL topic name | (e.g. `/sensors/depth/front`) | from earlier milestone wiring |

Decide on the **best ZED capture mode for this milestone** based on the M2 format. A higher-resolution capture is usually unnecessary if the policy consumes a small tensor; lower resolution + higher framerate is often a better trade. Document the chosen mode and rationale.

---

## 3. HAL depth-conversion function

The function lives in the HAL layer between the ZED bridge and the policy. It subscribes to the ZED native depth topic, transforms each frame to the M2 student format, and publishes to a topic the policy is configured to read.

The structure of this function is not a design problem to solve from scratch — **port it from the existing Go2 / Extreme Parkour on-robot depth pipeline.** That pipeline already takes a real depth camera (Intel RealSense D435i) and produces the tensor a parkour student policy expects. The krab is doing the same thing with a different camera and a different policy mount. Read that code, mirror its structure, and only change parameters that have to change.

### 3.1 Mirror the Go2 reference pipeline

Reference repo: **`change-every/Extreme-Parkour-Onboard`** — the Go2 deployment fork, which publishes D435i depth at 100 Hz on a ROS topic and feeds it to the policy. The on-robot scripts there are the model to follow. Specifically:

- The downsample method (interpolation type) used to get from D435i native resolution to the policy's input resolution — **use the same method.**
- The crop strategy (center crop, asymmetric crop, alignment with the camera's principal axis) — **use the same strategy**, only adjusted for the ZED's different FOV.
- Invalid-pixel handling (how NaN / zero depth pixels are dealt with before the tensor reaches the policy) — **use the same handling.**
- Whether and how the depth values are clipped or normalized before reaching the policy — see Section 3.2.

If something in the Go2 reference doesn't apply to the krab — e.g. a Go2-specific axis convention, an orientation correction tied to the Go2 mount — document the deviation with a one-line reason. Default is mirror; deviations are exceptional.

### 3.2 Clipping and normalization — match sim, don't introduce new transforms

The rule is **sim and real do identical pre-processing**, full stop. Whatever the M2 training config does to its rendered depth before feeding it to the policy, the HAL function does to the ZED depth. Whatever the M2 training config doesn't do, the HAL function doesn't do.

In particular:

- **If the sim clips depth** (e.g. clamps to a near/far range): the HAL function clips with the same bounds.
- **If the sim doesn't clip**: the HAL function doesn't clip. The ZED is a better camera than the D435i and likely doesn't need clipping for its own noise floor — but that's irrelevant. The point is parity with sim, not what the camera could tolerate.
- **Same for normalization**: match the sim's encoding (raw meters / inverse-depth / normalized-to-[0,1]) exactly.

Document what the sim does (this should also be in Section 1's spec) and confirm the HAL function does the same.

### 3.3 Example shape of the conversion

Skeleton only — the actual steps and parameters come from the Go2 reference and the documented M2 sim config. This sketch omits clipping and normalization on the assumption that the sim doesn't do them; if the sim does, add the matching step.

```python
import numpy as np
import cv2

# Parameters from Section 1 (M2 sim config) and Section 2 (ZED native output).
# FOV crop ratios are computed from documented FOVs; downsample resolution is
# the documented M2 student input.
M2_HEIGHT = 58           # placeholder — pull from sim config
M2_WIDTH = 87            # placeholder — pull from sim config
CROP_RATIO_W = ...       # = M2_HFOV / ZED_HFOV
CROP_RATIO_H = ...       # = M2_VFOV / ZED_VFOV
NAN_FILL = ...           # whatever the Go2 reference uses (often far plane or 0)

def zed_to_m2_depth(zed_depth_m: np.ndarray) -> np.ndarray:
    """Convert ZED native depth (meters, NaN for invalid) to the tensor the M2
    student policy expects. Mirrors the Go2 reference pipeline; only the FOV
    crop ratios and target resolution differ."""
    h, w = zed_depth_m.shape

    # FOV crop — same crop strategy as the Go2 reference, parameters adjusted
    # for ZED FOV vs D435i FOV.
    crop_w = int(w * CROP_RATIO_W)
    crop_h = int(h * CROP_RATIO_H)
    y0 = (h - crop_h) // 2
    x0 = (w - crop_w) // 2
    cropped = zed_depth_m[y0:y0+crop_h, x0:x0+crop_w]

    # Downsample — same interpolation as the Go2 reference.
    downsampled = cv2.resize(
        cropped, (M2_WIDTH, M2_HEIGHT), interpolation=cv2.INTER_AREA
    )

    # Invalid handling — same as the Go2 reference.
    downsampled = np.nan_to_num(downsampled, nan=NAN_FILL, posinf=NAN_FILL)

    # NOTE: no clipping or normalization here. If the M2 sim config does either,
    # add the matching step. If not, leave the output in raw meters.

    return downsampled.astype(np.float32)
```

### 3.4 Where it runs

Latency budget matters — the conversion sits on the inference path. For a small target resolution it should be sub-millisecond on the on-robot compute. If it isn't, profile and either move to GPU (the on-robot compute has one) or reduce ZED capture resolution.

Run options:
- As a node inside the policy container that subscribes to the raw ZED HAL topic and republishes the converted tensor.
- As a process inside the ZED bridge that publishes the converted tensor directly.
- As an in-process call inside the policy code (lowest latency, tightest coupling).

Implementer's choice — document the chosen architecture and the measured per-frame latency. If the Go2 reference made a particular choice here, default to the same.

### 3.5 Unit tests

- Shape test: arbitrary ZED-shape input produces M2-shape output.
- Range test: output is in the encoding the sim uses (raw meters / inverse / normalized) — whatever Section 1 documents.
- NaN test: input with NaN pixels produces a valid output (no NaN propagation).
- Invariance test: a uniformly-far frame produces a uniform output tensor at the expected value.
- **Parity test against Go2 reference behavior:** feed a known-shape test frame; compare the steps performed and the per-pixel result to what the Go2 reference would produce on an equivalent frame (modulo the FOV/resolution differences).

---

## 4. Bench test against sim-rendered depth

The acid test is whether the HAL output matches what the sim would render for the same scene. Use a scene with **distinct geometric features at known positions** — not a flat wall. A flat wall gives a near-uniform depth field that hides exactly the kinds of bugs this test should catch (wrong FOV crop, wrong scaling, wrong downsample). A scene with edges, multiple depth planes, and known object dimensions makes mismatches obvious.

### 4.1 Test scene

Pick one of the following (whichever is easier to stage). Both work; the second is preferred if floor space allows because it covers more of the depth range.

**Option A — Cube + back wall.**
- One cardboard box of known dimensions (e.g. 30 × 30 × 30 cm) on the floor centered ~1.0 m in front of the robot.
- A flat wall ~2.0 m in front of the robot (so 1.0 m behind the box).
- No other clutter in the ZED's field of view.

**Option B — Multiple boxes on the floor.**
- 3 or 4 cardboard boxes of known dimensions on the floor at staggered distances and positions — e.g. one at 0.7 m centered, one at 1.2 m offset to the left, one at 1.8 m offset to the right, one at 2.5 m centered.
- Background can be open floor or a far wall; document whichever.

Measure and write down: each object's dimensions, its position (X forward, Y left, Z up) relative to the robot's body frame, and the position of any back wall. These measurements become the sim scene spec.

### 4.2 Capture procedure

1. Stage the real scene per 4.1; measure and record object positions.
2. Capture one depth frame from the ZED with the robot stationary. Run it through the HAL function from Section 3. Save the output tensor.
3. Build the same scene in the sim: place primitives (boxes, plane) at the measured positions in `crab_hex.usd`'s world, or in a separate scene USD loaded around the crab for this test. Use the same camera (placed in Task 2) at the same body pose.
4. Render the sim's depth observation — the one the policy sees during training, **after** whatever pre-processing the sim does on it.
5. Save both tensors and a visualization of each (heatmap or grayscale) at the same color scale.

### 4.3 Comparison

The most diagnostic thing isn't an aggregate MAE across the tensor — it's whether the same features appear at the same pixel locations with the same depth values. Compare:

- **Object edges** — the cube's vertical and top edges (or each box's edges) should appear at the same pixels in both tensors. Misaligned edges indicate FOV crop wrong, camera pose wrong, or downsample method different.
- **Per-feature depth values** — at the center of each known object, the depth value in both tensors should be the same (within the encoding tolerance). Real will be noisier; sample several pixels and compare means. Systematic offset = encoding mismatch or near/far config mismatch.
- **Background / back wall** — for Option A, the visible portions of the back wall should read ~2.0 m in both. For Option B, the floor or far wall should match.
- **Depth gradient across the floor** — for Option B, the floor between boxes gives a continuous depth ramp from near to far; it should look the same in both tensors.

Compute per-pixel MAE and max error as summary stats, but treat them as supporting evidence — the visual + per-feature comparison is what catches the bugs that matter.

### 4.4 Tolerance

Acceptance tolerance is contractor's judgment. Real ZED depth will be noisier than sim, and the scene has measurement error (objects placed by hand, not laser-aligned). Reasonable bar: per-feature center-pixel depth agrees within ~5 cm at ranges out to 2 m, and object edges land within 1–2 pixels of where the sim renders them. Document the bar chosen.

If the test shows a systematic offset or scale mismatch, fix it in the conversion function and rerun. Common bugs the structured scene catches well:

- FOV crop region wrong (off-center or wrong aspect, or not matching the Go2 reference's crop strategy) — features appear at the wrong pixel locations.
- Pre-processing parity broken — sim clips/normalizes and HAL doesn't, or vice versa, or both do but with different bounds — features at the right pixel locations but wrong depth values.
- Encoding mismatch (sim uses inverse-depth; HAL output is linear-depth, or vice versa) — depth values are inverted or off by a nonlinear factor.
- Sim ZED camera (placed in Task 2) not actually matched to the real ZED mount — entire scene shifted or rotated relative to sim.
- Downsample method different from Go2 reference — edges look softer/sharper in real vs sim.

Capture both heatmaps side-by-side, the per-feature comparison numbers, and a one-paragraph result.

---

## 5. Build, deploy, verify

### 5.1 Build

Rebuild the parkour image with:
- Task 2 weights (retrained against the prismatic action space and matched ZED camera).
- The HAL depth-conversion function in whichever architecture was chosen in Section 3.3.

```bash
cd krabby-research
make parkour-image WEIGHTS=path/to/task2/student.pt \
                   DEPTH_CONVERTER=zed_to_m2 TAG=parkour-m3t3
```

Document the build command, weights, and image digest.

### 5.2 Deploy

Via fleet portal if available, else `krabby-launcher` as in Task 1. Reuse the Task 1 deploy procedure — only the image tag changes.

### 5.3 Verify the depth tensor on the deployed robot

Log the depth tensor as the policy sees it inside the container:

```python
# Inside the policy container, around the inference call
depth = observation['depth']  # whatever the policy's observation dict uses
log.info(f"depth shape={depth.shape} dtype={depth.dtype} "
         f"min={depth.min()} max={depth.max()} mean={depth.mean()}")
```

Expected: shape and dtype match Section 1, range within the documented encoding bounds. If shape or dtype is wrong, the conversion function isn't producing what the policy expects, even though it might pass unit tests in isolation. Diagnose before continuing.

---

## 6. Policy reactivity test

This is the acceptance demo for Task 3.

Procedure:

1. Put the robot on a stand or in a constrained position where motion is acceptable (or motors disabled and only the command stream observed, if safer).
2. Log the policy's command output (the joint commands going to `MCUSDK`) at the policy update rate.
3. Hold a flat obstacle (clipboard, large book) at approximately the ZED's eye level.
4. Move the obstacle through several distances in front of the robot: ~2 m, ~1 m, ~0.5 m, ~0.25 m, and back out.
5. Capture the command-output log synchronized with a video of the obstacle position.

Expected: the policy's command output changes in a way that correlates with obstacle distance — exact behavior depends on what the M2 student learned, but holding a wall close should produce a different command stream than open space. If the command output is identical across all distances, depth is not making it to the policy in a usable form.

If the policy isn't reacting:

- Re-verify Section 5.3 (depth shape and range as expected inside the container).
- Compare logged depth tensor against an M2 sim-rendered tensor for a similar scene.
- Check the depth-tensor update rate is matching what the policy expects.

Capture the reactivity log/clip and stash in the deliverable.

---

## 7. Deliverable checklist

- [ ] M2 student depth-input spec documented (resolution, encoding, FOV, near/far, whether sim clips/normalizes).
- [ ] ZED native depth output characterized; chosen capture mode and rationale recorded.
- [ ] Go2 / Extreme Parkour on-robot depth pipeline studied; HAL function mirrors its structure (downsample method, crop strategy, invalid-pixel handling). Deviations documented with reasons.
- [ ] HAL depth-conversion function implemented, unit-tested, and committed; pre-processing matches sim exactly (clip iff sim clips; normalize iff sim normalizes); run architecture and latency documented.
- [ ] Bench test against sim-rendered depth: captured frames, per-pixel stats, tolerance + result documented.
- [ ] Container image with Task 2 weights and Task 3 depth converter built; image tag, digest, and deploy command documented.
- [ ] Image deployed to the real robot; depth tensor inside the policy container logged and matches the documented spec.
- [ ] Policy reactivity test: obstacle-at-distance log + synchronized video captured.
- [ ] Known issues logged with justification for anything not fixed in this task. Depth-pipeline infrastructure issues belong in Task 3; Task 4 covers only model-side work.