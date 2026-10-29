# Known Issues — carried into Milestone 23

Issues recorded before M23 began, ordered alongside Task 1's friction log at the start of Task 2. Each entry names where the problem lives, why it matters, and how it was established.

Provenance: entries marked verified were checked against `krabby-research` while writing this document. Entries marked reported come from notes kept in earlier working sessions and have not been re-checked. Entries marked flagged were raised in a grant document and inherit that document's confidence. This list covers what was written down; friction that was solved in the moment and never recorded is exactly what Task 2 exists to catch.

Severity is judged the way Task 2 scores impact: how often the problem is hit, multiplied by what it costs each time.

---

## Release and CI

### 1. Commits to `main` publish nothing

Verified. All three publish workflows — `.github/workflows/publish-firmware.yml`, `publish-locomotion.yml`, and `publish-packages.yml` — trigger on `mainline` and `release/**`. The repository default branch, and the target of `origin/HEAD`, is `main`. Both branches exist on the remote.

A commit merged to `main` therefore produces no firmware artifact, no locomotion image, and no package. Anyone who assumes the default branch is the release branch ships nothing and gets no signal that they did. This is the highest-severity item on the list because it silently breaks the assumption every other release process rests on, and because Task 1's bench test has to know which branch is authoritative before it can watch anything.

Resolution is a decision rather than a fix: either `main` becomes the publishing branch, or the two branches get documented roles and the discrepancy is made visible.

### 2. No tests run on commit

Verified. `publish-packages.yml` runs a package's tests before uploading it, but that workflow fires only on version tags. There is no workflow that runs the test suite on a push or a pull request, so a commit that breaks tests reaches `mainline` without complaint and stays there until someone tags a release.

This is the gap Task 1 closes for the hardware path. The pure-software test run is a smaller, separate fix and belongs in the backlog.

### 3. No notification on any CI outcome

Verified. None of the three workflows report anywhere. Success and failure are equally silent, so the state of the published artifacts is only knowable by opening the Actions tab or trying to use them.

### 4. The IsaacSim image is built but never published

Verified, and flagged in the M21 grant. `images/isaacsim/` builds locally, and there is no `publish-isaacsim` workflow to match `publish-locomotion.yml`. Every consumer of the simulator image therefore builds it themselves, which makes the simulated path meaningfully harder to adopt than the real one.

M21 deferred publishing this image to M23 explicitly, along with detecting an absent MCU and offering the simulator, GPU capability validation, and IoT-based resume.

---

## HAL

### 5. The Isaac contact sensor is never assigned

Verified at `hal/server/isaac/hal_server.py:317`, which reads `self.contacvt_sensor = self.scene.sensors['contact_forces']`. The typo means `self.contact_sensor` is never set, so any code reading it fails or silently sees nothing. Anything routing contact or current data through the HAL server hits this first.

### 6. Position commands to the real MCU are disabled

Verified at `hal/server/jetson/krabby_mcusdk.py:122`. `apply_command` builds normalized position commands and then does not send them, falling back to `_jog_all_joints` instead, because a potentiometer is not connected on every joint. The comment records the reason.

The effect is that closed-loop position control exists in the firmware and in the SDK but is not the path the real robot runs. Anything reasoning about the robot as position-controlled is reasoning about a path that is currently bypassed, and the sim-to-real gap includes this divergence.

### 7. Jetson observations are partly unimplemented

Verified at `hal/server/jetson/hal_server.py:390`, `:398`, and `:708`. Real sensor extraction from IMU and encoders is marked as future work, and joint positions are not read back from encoders in the observation path.

### 8. The HAL has no abstract backend contract

Verified. `HalServerBase` in `hal/server/server.py` is a concrete ZMQ transport rather than an abstract base, with no runtime factory. Backends subclass it and diverge: the Jetson `apply_command` takes a command and returns None, while the Isaac one takes no argument and returns a tensor. Backend selection happens at the packaging and console-script layer.

This works, and it is the reason adding a backend means reading two existing ones carefully to learn which shape to copy.

---

## Firmware

### 9. Milestone 16 sensors are specified but not implemented

Verified and flagged in the M21 grant. `firmware/arduino/` contains no `sensors_config.h`, no `#include <Wire.h>`, and no BMI270, SSD1306, or INA228 code. M16 is spec-complete and code-incomplete.

M21's Task 4 depends on this existing to simulate against, so the dependency crosses milestones.

### 10. John's OLED simulator is not in the repository

Verified by search, and flagged in the M21 grant. It is referenced as an input to M21 Task 4 and exists nowhere in `krabby-research`. Locating it and vendoring it in is a prerequisite for that task and an unowned action today.

### 11. Actuator identity is hardcoded in several places

Verified at `firmware/arduino/actuator_manager.h:49`, `:358`, and `:373`. Actuator order is hardcoded, joint information is hardcoded rather than stored alongside the joint, and the pin configuration cannot be reused across leader and follower boards with different actuator names. The EEPROM struct is named as the right home for the naming information.

Each is marked in the source as a known compromise. Together they are why board role and actuator naming are harder to change than they look.

### 12. Actuators are instantiated before role election completes

Verified at `firmware/arduino/arduino.ino:241`, where the source notes that instantiation should be delayed until role election finishes rather than corrected afterward.

### 13. An error condition is silently swallowed

Verified at `firmware/arduino/arduino.ino:137`, commented "THIS SHOULD THROW SOME KIND OF BAD ERROR CONDITION". Whatever this branch catches currently passes unreported, which is the failure mode most likely to waste a bring-up afternoon.

### 14. Actuator lookup is linear and quadratic in places

Verified at `firmware/arduino/actuator_manager.h:263` and `:289`. Both are marked as acceptable at six actuators per board and both are noted as needing a map if the actuator count grows.

---

## Controller and teleop

### 15. The Krabby model-controller mode is unimplemented

Verified at `controller/control_loop.py:128`, where `MODEL_CONTROLLER_KRABBY` is a stub.

### 16. Observation timestamps are not propagated

Verified at `controller/control_loop.py:292`, which passes None because no observation timestamp is available. Latency accounting through the teleop path is therefore incomplete.

### 17. Starting joint positions are not read from robot state

Verified at `controller/mappers/gamepad_to_isaacsim_hal_mapper.py:193`. The mapper assumes rather than reads them.

### 18. The gamepad path is tested on macOS only

Verified at `controller/scripts/isaac/test_gamepad_to_isaacsim_hal.py:23`, dated January 2026 and stating that Linux and Windows are untested. The Bluetooth controller check in Task 1 runs on the Orin, which is neither.

### 19. Gamepad launch is pinned to the Jetson backend

Flagged in the M21 grant, at `krabby/_docker.py` around line 125, where the gamepad mode hard-codes `krabby-hal-server-jetson`. Any other backend cannot be launched through that path.

---

## Fleet operations

### 20. Updates require an operator to SSH in

Reported. M10 deliberately excluded cloud-driven rollout: the agent publishes telemetry, carries teleop signaling, and opens Secure Tunneling proxies, and does not listen for update commands. Operators SSH in and run `krabby update` themselves.

This was a scope decision rather than a defect, and it is listed here because it is friction that M23 will encounter during bring-up and may choose to reduce without reversing the decision.

### 21. A stopped simulator shows offline until manually restarted

Flagged in the M21 grant. There is no IoT-based resume, so recovery is a manual CLI step.

---

## Simulation and developer environment

### 22. Running `play.py` on Windows requires three undocumented fixes

Reported from an earlier session, and unconfirmed as to whether the fixes landed upstream. The task README is bash-only, and three changes were needed on Windows: the launcher had to be a `cmd.exe` script because PowerShell does not activate conda cleanly; `h5py` had to be imported before `AppLauncher` boots Kit, or Kit's own `hdf5.dll` maps first and h5py's `_errors` extension fails to link; and `ParkourHexContactSensorCfg` was missing `track_friction_forces` and `max_contact_data_count_per_prim`, both of which its sensor reads.

The third of those is a genuine half-finished refactor in the shared code rather than a Windows quirk, and is worth confirming against the current tree.

### 23. Kit reports non-fatal sensor extension failures on Windows

Reported. `omni.sensors.nv.lidar`, `omni.sensors.nv.radar`, and `isaacsim.sensors.rtx` all fail to load with a DLL error on cold start. Kit continues, and the student MDP does not need them, so the practical cost is that a new developer cannot tell these errors apart from real ones.

---

## Documentation and process

### 24. The grants repository README describes a convention nobody uses

Verified. `patina-foundation-grants/README.md` describes task files as `T1-<name>.md` and states that each grant directory contains an `AUDIT-LOG.md`. Every milestone actually uses `TASK-N-<NAME>.md`, and no milestone has an audit log. New contributors following the README produce files that do not match anything.

### 25. Grant time estimates were not internally consistent

Verified in M21, where the per-task day tables summed to 5, 5, and 5.5 days against headings of 6, 6, and 7. Corrected in M21. Whether the same drift exists in other milestone documents has not been checked, and checking it is a small, well-specified task.

### 26. The M21 contract references merging to `main`

Verified. `krabby-contracts/milestones/M21/M21.md` states that artifacts are merged to `main`, which is consistent with the repository default and inconsistent with what the publish workflows watch. Whichever way issue 1 is resolved, the contract language should follow it.

---

## Documentation structure and audience

The repository has roughly forty markdown documents totalling around nine thousand lines. Most are accurate. The problem is that almost none of them declare who they are for, so a reader cannot tell whether a section applies to them, and the three audiences are interleaved inside single files.

The three audiences worth separating are the standard user who has a robot and wants it running, the developer who is changing the code, and the one-time infrastructure setup that is performed once per account or per fleet and never again by anyone else.

### 27. The root README interleaves all three audiences

Verified. `README.md` opens with a kit list and a six-step software quick-start, which is standard-user material. Step 7 is optional inference, still user material. It then presents "Continuous bench watchdog", which instructs the reader to `sudo pip3 install krabby-bench` and pass `BENCH_SMTP_TO`, `BENCH_GITHUB_REPO`, and `BENCH_GITHUB_TOKEN` — one-time infrastructure setup for a bench machine, sitting inside the getting-started flow for someone who wants to drive a robot.

The closing "Further reading" table then mixes all three again: firmware setup and the CLI reference are user-facing, the locomotion image document is largely maintainer-facing, and the bench watchdog is infrastructure.

A reader following the README top to bottom cannot tell where their job ends.

### 28. Nothing routes a reader by intent

Verified. `docs/FOLDER_LAYOUT.md` describes where things live, which answers a different question from what a reader arriving at the repository actually asks. There is no top-level entry point that says: if you have a robot and want it running, read these three documents in this order; if you are changing code, start here; if you are standing up AWS for the first time, here.

This is the cheapest high-impact fix on the list, because it needs no reorganization of the underlying documents to be useful.

### 29. The installed package and the spoken name differ

Verified. The distribution is `krabby-launcher` (`krabby/pyproject.toml` line 6) and the console command it installs is `krabby`. Every document correctly says `pip install krabby-launcher`, and everyone in conversation says `pip install krabby`.

The reason for the mismatch is settled and worth recording so it stops being rediscovered: `krabby` on PyPI is held by an unrelated project that has been dead for about ten years. The name was unavailable, so the distribution took `krabby-launcher` while the console script kept the short name.

The gap remains a trap for a new user, who types what they heard and gets either nothing or the dead project. Three ways out, in rough order of value:

Request the name under PEP 541. PyPI's name-retention policy covers abandoned projects, and requests are filed as issues against the `pypi/support` repository. A decade of no releases is a reasonable case, though the process contacts the current owner first and transfer is discretionary rather than automatic. This is the only option that makes the spoken name and the real name agree.

Failing that, make the mismatch loud rather than quiet. The getting-started document should state the install command and say plainly that `pip install krabby` is a different, unrelated project, because a reader who guesses wrong currently gets no signal that they guessed.

Renaming to a third name that is available and memorable is the fallback, and costs a deprecation cycle across the docs, the publish workflow tag `krabby-v*`, and anything pinning the distribution.

The M23 grant documents themselves were written saying `pip install krabby` before this was checked, which is evidence of how easily it propagates.

### 30. `firmware/SETUP.md` still carries a milestone-task title and serves four audiences

Verified. The document is titled "Krabby-Uno Task 2: Six-Axis Leg Controller", which describes the milestone that produced it rather than what it is now, which is the canonical firmware reference.

Inside, it runs hardware wiring for Rev 3 boards (one-time build), a serial RX buffer patch to the Arduino core (developer environment setup), the telemetry wire protocol (developer reference), a usage guide covering auto-calibration and manual jog (user), the EEPROM address layout (developer), and the S3 firmware store bucket layout (infrastructure). Any one of those is fine; the mix means nobody can read the document straight through.

### 31. Enrollment is documented in two places

Verified. `fleet/ENROLL.md` covers device enrollment on the Orin, and `fleet/SETUP-FLEET.md` covers it again under "Enroll and SSH" alongside "One-time per robot (Orin)". Two documents drift; a reader cannot tell which is current, and the bring-up in Task 1 has to pick one.

### 32. `fleet/SETUP-FLEET.md` mixes account setup with per-robot operation

Verified. It opens with a deploy sequence for a fresh AWS account, which is done once ever, then covers per-robot enrollment and operator SSH, which is done per robot, then the MQTT topic scheme and a Playwright E2E, which are developer reference. coturn configuration and force-relay verification sit in the middle as infrastructure.

The natural split is a one-time infrastructure document, an operator document, and a protocol reference.

### 33. Two competing host-setup paths

Verified. `docs/JETSON_DEPLOYMENT.md` walks through SSH access, Docker installation, and the NVIDIA container toolkit by hand, while the root README says `sudo krabby install` writes the udev rule, adds the user to `dialout`, and installs the systemd unit. A reader has no way to tell whether the manual guide is prerequisite, alternative, or stale.

Establishing which one is current, and marking the other accordingly, is a decision that the Task 1 bring-up will force anyway.

### 34. `DEVELOPER.md` has no title and covers one half of development

Verified. The file begins at an H3 heading with no H1, and covers CUDA, Python 3.11, PyTorch, IsaacSim, IsaacLab, HAL editable installs, Docker GPU, and `make test`. That is the simulation and training environment. Nothing covers developing against a real robot, which is the other half and the one Task 1 exercises.

### 35. Image and package READMEs mix consumer and maintainer sections

Verified. `images/locomotion/README.md` opens with a canonical install path for a consumer, then continues into tag schemes, PyPI package pins, and pin-bumping procedure for a maintainer. `images/isaacsim/README.md` is maintainer-facing throughout, which is consistent but unmarked, and it documents building an image that is never published (see entry 4).

---

## Scope note for Task 1

### 36. A bench watchdog already exists

Verified. `bench/` contains the `krabby-bench` package, installed with `sudo krabby-bench install`, which runs as a systemd service polling ECR every 60 seconds for a new `mainline-latest` digest. On a new digest it runs `krabby firmware show` to discover ports, `krabby firmware update <channel> <port>` for each board, re-reads versions, asserts all three boards report the same version, and checks that version against the S3 manifest at `krabby-firmware-public`. It alerts by email or by opening a GitHub issue.

That overlaps substantially with the Task 1 bench harness. What it does not do is trigger on a commit rather than on a new image digest, install from a clean state rather than assuming `krabby-launcher` is already present, assert that a motor actually moved rather than that versions match, report to Discord, or check that the published packages and images install and start.

Task 1 should extend `krabby-bench` rather than build alongside it. The existing smoke test is the flash-and-verify stage already written; what is missing is the fresh-install stage, the motion assertion, the trigger, and the notification path.

---

## Not issues, recorded to prevent rediscovery

`firmware/.venv` and `firmware/arduino/Downloads/` both exist on disk and appear in local searches. Neither is tracked by git, so both are local artifacts rather than repository contents. They are noted here because they show up in repo-wide greps and have been mistaken for committed junk.
