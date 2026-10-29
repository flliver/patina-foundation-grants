# Task 1 — Cold-start bring-up, bench test, and build notifications

Goal: bring up a complete Krabby from bare hardware while recording every point of friction, then turn that path into a bench test on a real Orin that runs on every commit and every published artifact, performing a fresh install, a firmware reflash, a stack bringup, and a real motor movement, and reporting pass or fail to Discord alongside the health of the published packages, images, and firmware.

The two halves are one task because automating the install is what forces every undocumented step into the open: a step that lives in someone's head survives a manual bring-up and fails a scripted one.

Outputs
- A complete robot brought up from a bare Orin and unflashed MCUs through to portal, cameras, teleop, and a Bluetooth controller.
- A friction log with one entry per deviation from the documented path, carrying symptom, workaround, component, cause, whether it blocks, and what it costs.
- A bench Orin registered as a CI target, resettable to a known state between runs without a person touching it.
- A four-stage bench harness covering install, flash, bringup, and motion, each reporting its own pass or fail.
- A Discord notification carrying result, triggering commit, failing stage, and a link to the run.
- Artifact health checks for the published packages, the locomotion image, and the firmware artifact.
- The command sequence a working bring-up actually requires, and the distance from there to the few-commands goal.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-1---cold-start-bring-up-bench-test-and-build-notifications).)

---

Note: line numbers are pointers to the right file and may drift. Issues recorded before this milestone are in [KNOWN-ISSUES.md](KNOWN-ISSUES.md) and do not need rediscovering, though confirming they still reproduce is useful.

---

## 1. What CI does today

Three workflows exist in `.github/workflows/`:

| Workflow | Trigger | Produces |
|---|---|---|
| `publish-firmware.yml` | push to `mainline` or `release/**`, daily 07:00 UTC | Arduino firmware to S3 via OIDC role |
| `publish-locomotion.yml` | push to `mainline` or `release/**`, `locomotion-v*` tags, daily 07:30 UTC | production locomotion image to ECR Public |
| `publish-packages.yml` | `<pkg>-v*` version tags | `krabby-*` packages to PyPI, running that package's tests first |

Tests run only when a version tag is pushed, so ordinary commits carry no test signal. All three workflows watch `mainline` and `release/**` while the repository default branch is `main`, so which branch this harness should follow needs settling before anything is wired up (see KNOWN-ISSUES entry 1).

Nothing reports results anywhere today, which is why the Discord piece belongs here rather than in a follow-on.

## 2. The first bring-up, by hand

The bring-up is done once: build the robot from nothing, recording as you go, since a log reconstructed afterward loses the small items it exists to capture.

The documented path is roughly: flash the Orin with its base image, `pip install krabby-launcher`, `krabby install` to pull the locomotion image and set up services, `krabby firmware` to flash the MCUs, `krabby enroll` to provision IoT identity and enable the agent, and `krabby run` to start the stack. From there the robot should appear in the fleet portal, stream cameras, accept a teleop session, and respond to a paired Bluetooth controller. The entry points are `krabby/install.py`, `krabby/firmware.py`, `krabby/enroll.py`, `krabby/run.py`, `krabby/agent.py`, and `krabby/_docker.py`, with the fleet side in `fleet/cli/` and `fleet/service/`.

Treat that description as a hypothesis: establishing what the path actually is, in order, with everything it silently assumes, is half the point.

Stages to work through and record:

Orin from bare. Flashing or imaging, first boot, network, account and SSH setup, and whatever has to be true before `pip install krabby-launcher` runs at all. Note the Python version and how it got there.

Package install. `pip install krabby-launcher` from the public index, then `krabby install`. Record anything needing sudo, any environment variable that turns out to be required, and anything that fails once and succeeds on retry.

Firmware. `krabby firmware` against three MCUs. Record how boards are addressed, whether ordering or cabling matters, whether role election completes, and how long it takes to know it worked. Port discovery is in `firmware/mcu_port.py` and honors `KRABBY_MCU_PORT`.

Enrollment and fleet. `krabby enroll`, then confirm the robot appears in the portal. Distinguish appearing in the registry from reporting live telemetry, since a stale registration that looks correct is a friction item in its own right.

Cameras, teleop, controller. Confirm each camera streams, run a teleop session end to end with both control and video, then pair a Bluetooth controller and drive the legs. The camera catalog and driver selection are around `hal/server/jetson/front_camera_factory.py`. Note that the gamepad path is recorded as tested on macOS only (KNOWN-ISSUES entry 18) and that gamepad launch is pinned to the Jetson backend (entry 19), so this stage is a likely source of entries.

### The friction log format

One entry per deviation, kept short and specific — a log that takes an hour per item will not get written.

| Field | Content |
|---|---|
| Stage | Which bring-up stage it happened in |
| Symptom | What the operator saw, in the words they would search for |
| Workaround | What actually got past it |
| Component | The package, service, or document at fault |
| Cause | The underlying reason, where known; left empty where not |
| Blocking | Whether bring-up stops here, or it merely costs time |
| Cost | Rough minutes lost, and whether it recurs on every bring-up |
| Effort | Rough hours to fix it properly, to the half hour |
| Who | AI-suited, human, or needs hardware access |

Fill the last four fields in as you write each entry; they are what let Task 2 order the list and pack its bucket. Blocking and cost give the impact side — a thirty-second workaround hit on every bring-up outranks a half-hour problem hit once a year, which is why the cost field notes recurrence — and effort gives the other.

Mark an entry AI-suited when it is self-contained, verifiable without the robot in hand, and detailed enough to be worked from the entry alone — a typo, a missing default, an error message that does not say what to do, a documentation path that no longer matches the code. Mark it human when it needs judgment or a decision that is not the contractor's to make, and hardware when it needs the bench or a robot.

Record even items solved in under a minute.

## 3. The bench Orin as a CI target

`bench/` already contains `krabby-bench`, a systemd watchdog that polls ECR for a new `mainline-latest` digest and, on finding one, flashes every attached board, re-reads the versions, asserts all three match, and checks them against the S3 manifest, alerting by email or GitHub issue. That is the flash-and-verify stage already written and running.

What this task adds to it is a commit trigger rather than a digest poll, a fresh-install stage rather than an assumption that `krabby-launcher` is present, a motion assertion rather than a version comparison, a Discord path, and the artifact health checks in section 6. Extending `krabby-bench` is cheaper than building alongside it and avoids two bench services disagreeing about robot state (see KNOWN-ISSUES entry 36).

The bench needs an Orin with the MCUs attached over their normal serial links and at least one actuator physically connected and free to move without damaging anything. A single leg on a stand is enough; the test asserts motion, not gait.

Registering it as a self-hosted GitHub Actions runner is the path of least new infrastructure, since the existing workflows are already GitHub Actions and already assume OIDC role access rather than stored credentials. Label it so only bench jobs target it.

Between runs the Orin has to return to a state where the install stage genuinely tests a fresh install rather than an upgrade over the last one. Options in rough order of fidelity: a scripted teardown removing the `krabby-launcher` install, its state directory, and its containers; a container or filesystem snapshot restored per run; or a full reflash. A scripted teardown is the pragmatic starting point, with the caveat that it tests what teardown remembers to remove. Whatever is chosen, record it, because the gap between the reset and a truly bare machine is the gap where install bugs hide.

The state to clear lives around `krabby/_state.py`, `krabby/install.py`, and `krabby/_docker.py`.

## 4. The four stages

Each stage reports its own result so a failure names where it happened. The friction log from section 2 is the specification for what these stages have to do without human help.

Install. Install `krabby-launcher` from the public package index exactly as the documentation tells an operator to, with no editable installs and no path into the working tree. If the documented command stops working, the harness fails here, which is the signal that the getting-started path has broken.

Flash. Reflash the MCU firmware from the published artifact using `krabby firmware` (`krabby/firmware.py`), then confirm the version on the board matches the artifact. The firmware reports version over the serial `V` command and `parse_ver_reply` in `firmware/krabby_mcu.py` already parses the reply, so the check is a comparison rather than new parsing. On a three-board robot the leader aggregates all three versions into one reply.

Bringup. Start the stack the way an operator does, through `krabby run` (`krabby/run.py`), and wait for the MCU to be present and telemetry to flow. `krabby/telemetry.py` already reports `mcu_present` in `collect_health()`, which gives this stage a definite success condition rather than a sleep.

Motion. Drive at least one actuator and assert through telemetry that it moved. Assert on reported position or hall count changing in the commanded direction, rather than on the command being accepted, because a command that is accepted and does nothing is exactly the failure this bench exists to catch. The Jetson SDK currently jogs rather than sending position targets (KNOWN-ISSUES entry 6), so drive the path the robot actually uses.

## 5. Notifications

A Discord webhook is the smallest thing that works and needs no bot to maintain. The message should carry the result, the triggering commit with a short subject, the stage that failed where applicable, and a link to the run.

Post on both success and failure: a channel that only speaks when something breaks leaves silence ambiguous between healthy and not running. If per-commit success proves noisy, collapse consecutive successes rather than suppressing them.

Failure messages should be actionable from the message alone, which means naming the stage and including the command that failed.

## 6. Artifact health checks

Separate from the bench run, and cheaper, these answer whether what has been published is usable:

- Packages: install each published `krabby-*` package from the public index into a clean environment on the target architecture and import it. Publishing succeeds and installing fails more often than is comfortable, particularly on aarch64.
- Locomotion image: pull `public.ecr.aws/t7t7b3i3/krabby-locomotion` at the channel tags operators actually use (`release-latest`, `mainline-latest`) and confirm the container starts.
- Firmware artifact: confirm the published artifact exists for the commit it claims and that its version string matches.

These run on ordinary hosted runners and do not need the bench, so they belong in a separate job that does not queue behind hardware.

## 7. The command sequence

Close by writing the sequence of commands a working bring-up requires, including the undocumented ones, in order. This is the baseline the milestone is measured against: Task 2 ends by producing the same sequence again and comparing, with the gap to the few-commands goal stated as a number.

## 8. Verify

- A robot brought up from bare hardware reaches a state where it appears in the portal with live telemetry, streams cameras, completes a teleop session, and responds to a Bluetooth controller.
- The friction log has an entry for every deviation, with blocking and cost filled in on each.
- Trigger a bench run from a commit and observe all four stages execute in order, with the run reaching the motion stage and the actuator visibly moving.
- Break each stage deliberately — a bad package version, a mismatched firmware artifact, a stopped service, a disconnected motor — and confirm the run fails at that stage and the Discord message names it.
- Confirm a passing run and a failing run are distinguishable in Discord without opening the run, and reproduce a failure by hand from the notification alone.
- Run the harness twice in succession and confirm the second run is genuinely a fresh install rather than an upgrade over the first.
- Confirm the artifact health checks fail when pointed at a package version known to be broken or absent.
- The command sequence is written down in order, with undocumented steps marked as such and the gap to the few-commands goal stated as a number.
