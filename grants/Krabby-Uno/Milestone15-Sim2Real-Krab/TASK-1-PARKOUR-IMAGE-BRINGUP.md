# Task 1 — Parkour image bring-up on real robot

Goal: Pull the existing pieces together and validate the M2 parkour image running end-to-end on real Krabby-Uno hardware. The components below have all been built and tested in earlier milestones — this task is **integration validation, including fixing anything that's broken at the integration seams**. Task 4 is exclusively about tuning and fixing the model itself; all end-to-end infrastructure work (HAL, MCUSDK, `krabby-launcher`, Bluetooth, auto-start) lands in Task 1.

Outputs
- Deployed M2 parkour image running on the real krab via `krabby-launcher`, auto-starting on boot.
- Validated HAL link between the policy container and `MCUSDK`, with bidirectional traffic captured to a log.
- Bluetooth controller paired to the on-robot compute, with controller inputs reaching the parkour policy over HAL as the command stream the policy was trained against.
- Updated Makefile targets in the affected repos covering build, deploy, and smoke-test steps.
- Updated README.md sections in the affected repos covering bring-up steps, expected outputs, and captured artifacts (boot log, HAL topic list, smoke-test log, Bluetooth pairing, stick test).

Acceptance Criteria
- **1a** — M2 parkour image builds and is deployable to the robot via `krabby-launcher`; deploy command documented.
- **1b** — Container auto-starts on boot; verified by power-cycling the robot and observing the process come up unattended.
- **1c** — HAL link between the parkour policy container and `MCUSDK` is live; verified with a HAL inspector or log capture.
- **1d** — Joint commands from the parkour policy reach the MCU; observed motor motion (even if random) on all 18 actuators.
- **1e** — Telemetry stream from `MCUSDK` back to the parkour policy is live; sample of joint positions, motor currents, and IMU captured in a log.
- **1f** — Bluetooth controller pairs to the on-robot compute; controller inputs arrive at the parkour policy over HAL as the command stream the policy expects; verified by moving a stick and observing the corresponding change in command input to the model.

---

**NOTE**: All commands and config snippets in this document are guidance. Actual paths, topic names, and APIs may vary between earlier milestones and current state; document the exact steps that work in your environment.

---

## 1. What should already work — re-validate, and fix if broken

The following components have been built and shipped in earlier milestones. Don't rebuild them from scratch in this task — re-validate that they still work with the M2 parkour image in the loop. **If any of them is broken, fixing it is in-scope for Task 1.** End-to-end infrastructure work belongs here; Task 4 is reserved for tuning and fixing the model itself, not patching plumbing.

| Component | Earlier milestone | What to verify |
|---|---|---|
| `krabby-launcher` deployment of a container image to the robot | Launcher milestone | Image push + run command completes; container reaches `Running` state |
| Container auto-start on boot | Launcher / fleet milestone | systemd or equivalent unit exists; survives power-cycle |
| HAL transport between an on-robot container and `MCUSDK` | HAL / MCUSDK milestone | HAL inspector lists expected topics; pub/sub round-trip works |
| `MCUSDK` driving all 18 actuators on joint commands | MCU PCB / firmware milestone | Manual joint command → motor motion on every actuator |
| `MCUSDK` publishing telemetry (joint positions, currents, IMU) | MCU PCB / firmware milestone | Telemetry rate matches spec; values look sane on a bench manipulation |
| Bluetooth pairing of a controller to the on-robot compute | Controller milestone | Controller pairs; OS sees axis/button events |
| Controller input surfacing on HAL | Controller milestone | HAL topic for controller input has live data when sticks move |

If anything in this table is broken when you start Task 1, fix it as part of this task. The end-to-end infrastructure has to work before Task 2 (joint mapping), Task 3 (depth pipeline), or Task 4 (model tuning) can produce meaningful results. Scope creep into building new infrastructure is a flag — talk it through — but any in-place repair of existing infrastructure is expected here.

---

## 2. What needs integration validation in this milestone

These are the seams that have **not** been tested with the M2 parkour policy specifically:

- The M2 parkour image (built from current `krabby-research` weights) running end-to-end on the robot through all of the components in Section 1.
- Bidirectional flow specifically between the parkour policy and `MCUSDK` over HAL — policy joint commands reach the MCU, `MCUSDK` telemetry reaches the policy at the rate the policy expects.
- Controller inputs flowing over HAL into the parkour policy as the command stream the policy was trained against (the velocity command vector or whatever the M2 student takes as conditioning input).

The robot will spazz out at this stage — random-looking flailing is expected and not a failure. The success condition is the data path through every seam, not the gait.

---

## 3. Build and deploy the M2 parkour image

### 3.1 Build

Build the parkour image with the current M2 weights from `krabby-research`. Reuse the existing image build pipeline; the only thing that changes here is which weights get baked in.

```bash
# Guidance — actual paths from the krabby-research repo
cd krabby-research
make parkour-image WEIGHTS=path/to/m2/student.pt TAG=parkour-m2
# or whatever the existing build target is
```

Document the exact command, the weights file used, the resulting image tag, and the image digest.

### 3.2 Deploy via krabby-launcher

```bash
# Guidance — actual command from the krabby-launcher milestone
krabby-launcher deploy --image parkour-m2 --target <robot-id>
```

Record the deploy command, the launcher version, and the output. If the launcher uses a manifest/config file, commit that to `krabby-research/deploy/` so the deploy is reproducible.

### 3.3 Auto-start on boot

The existing systemd unit (or fleet portal equivalent) should pick up the new image without modification. Verify by:

1. Deploying the image with the robot powered on.
2. Power-cycling the robot.
3. Observing the container come up unattended within the boot timeout.
4. Confirming the policy process is running inside the container (`docker ps`, `systemctl status <unit>`, or fleet portal status).

Capture the boot log (`journalctl -u <unit>` or equivalent) and link or embed it in the relevant README section.

---

## 4. HAL link bring-up

Once the policy container is running, verify the HAL link to `MCUSDK` before any motion testing.

### 4.1 Inspect HAL topics

Use the existing HAL inspector to list active topics and their publishers/subscribers. Expected:

- `MCUSDK` publishing telemetry (joint positions, joint velocities, motor currents, IMU).
- The parkour policy container subscribing to telemetry and publishing joint commands.
- A controller-input topic populated by the Bluetooth controller bridge.

If any expected topic is missing, this is a config issue between the parkour image and the HAL setup baked into earlier milestones. Diagnose before continuing.

### 4.2 Bidirectional smoke test (no motion yet)

With motors disabled at the MCU (or with the robot on a stand off the floor), confirm:

- Policy is publishing joint commands at the expected rate (matches M2 training step rate).
- `MCUSDK` is receiving those commands (count messages on the subscriber side).
- `MCUSDK` is publishing telemetry at the rate the policy expects (matches M2 observation rate).
- Policy is consuming telemetry (count messages on the policy subscriber side).

Latency budget: measure and record inter-process latency end-to-end (policy command issued → MCU receives, MCU telemetry published → policy receives). If latency is outside the envelope the policy was trained on, **fix it here** — buffer sizes, scheduling priority, transport choice, MCU loop rate are all in scope. Only if Task 1 can't make latency low enough does Task 4 need to widen the latency randomization range to cover it; the cheap fix is infrastructure-side, not retraining.

### 4.3 Enable motors and observe

E-stop within reach, robot on a stand or otherwise constrained from running away. Enable motors at the MCU. Observe motion on all 18 actuators. It will be random-looking — that's fine. The acceptance bar is **motion on every actuator**, not coherent gait.

Capture a short clip and a telemetry log (joint positions, currents, IMU) covering ~10 seconds of motion. Link or embed both in the relevant README section.

---

## 5. Bluetooth controller pairing and command stream

### 5.1 Pair the controller

Use the existing pairing procedure from the controller milestone. Document the exact steps, the controller model, and the OS-side device path.

### 5.2 Verify HAL bridge

Move sticks and press buttons. Watch the HAL controller-input topic. Stick deflection should appear as values on the topic in real time.

### 5.3 Verify policy consumption

The parkour policy takes a command vector as part of its observation — this is how the velocity command from the controller becomes the policy's "go forward" or "turn left" conditioning. Verify the bridge from controller-input HAL topic to the policy's command vector input:

- Log the policy's input command vector inside the container.
- Move the forward/backward stick. Observe the policy's input command vector change correspondingly.
- Move the left/right stick. Observe the same.

If the values are coming through but with wrong scaling, sign, or axis mapping — that's an integration bug to fix here, not a sim-to-real issue.

### 5.4 Stick test (acceptance)

With motors enabled, move the forward stick. Robot output will be flailing — but the *command stream into the model* should change predictably with stick position. Capture a log showing stick position and policy command-vector input synchronized in time.

---

## 6. Bring-up: update existing Makefiles and READMEs

**Do not create new doc files.** Anything the contractor would have written into a new bring-up doc goes into the existing Makefile targets and READMEs across the affected repos.

- **Makefiles** — any command that's now used to build, deploy, or smoke-test the parkour image should be a Makefile target in the relevant repo (`krabby-research`, `krabby-launcher`, wherever the existing build automation lives). If the targets already exist but are out of date, update them in place. If a step is being run manually but is reproducible, add a target. Targets should be runnable from a clean checkout.
- **README.md** — the existing READMEs in each affected repo are where bring-up steps, expected outputs, and troubleshooting notes get added. Update them in place. If a README doesn't mention the relevant step yet, add a section to the existing file rather than creating a separate one.
- **Captured artifacts** — the boot log excerpt, HAL topic list, smoke-test log, Bluetooth pairing steps, and stick-test log capture all get linked or embedded in the relevant existing README, not in a separate runbook doc.

The bar is: someone with a clean checkout can `git pull`, read the existing README, run a Makefile target, and reproduce the bring-up. No "where's the runbook?" — it's in the README and the Makefile.

---

## 7. Safety FYI

E-stop / kill-switch should be within reach for every step from Section 4.3 onward. This isn't an acceptance criterion — just don't be the person whose first real-robot motion test is also the first time anyone reaches for the kill switch on a 600 lb hexapod that's flailing. Keep motors disabled at the MCU until the data path is validated in 4.1–4.2.

---

## 8. Deliverable checklist

- [ ] M2 parkour image built; image tag, weights file, and digest documented.
- [ ] Image deployed via `krabby-launcher`; deploy command and launcher version documented.
- [ ] Auto-start on boot verified by power-cycle; boot log captured.
- [ ] HAL topic list captured; expected publishers/subscribers all present.
- [ ] Bidirectional smoke test passed (motors disabled): policy → `MCUSDK` and telemetry → policy at expected rates.
- [ ] All 18 actuators moving under policy command (motors enabled, robot constrained); clip + log captured.
- [ ] Bluetooth controller paired; pairing procedure, model, and OS device path documented.
- [ ] Controller input → HAL topic → policy command-vector input verified; stick-test log captured.
- [ ] Makefile targets for build, deploy, and smoke-test updated in the affected repos; runnable from a clean checkout.
- [ ] README.md sections in the affected repos updated in place with bring-up steps and captured artifacts (boot log, HAL topic list, smoke-test log, Bluetooth pairing, stick test).
- [ ] Known issues logged with justification for anything not fixed in this task.