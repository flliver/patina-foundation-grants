# Patina Foundation Grant — Krabby-Uno Milestone 23: Fleet Reliability and Quality of Life

## Grant Overview

Make the Krabby software stack work end to end from a standing start, and keep it working.

Today the pieces each work when driven by someone who already knows their quirks. The `krabby` pip package installs, the locomotion image runs, the firmware flashes, the fleet portal shows robots, teleop connects, cameras stream, and a Bluetooth controller drives the legs. Getting all of that running on a fresh Orin takes accumulated knowledge that lives in bits and pieces of .md files all over the code base, and no automated check tells anyone when one of those pieces stops working.

This milestone brings up a complete robot from bare hardware while recording every point of friction between the documented path and what a person actually has to do, and turns that same path into a public bench test on a real Orin with real motors. The bench runs on every commit and every published artifact, performing a fresh install, reflashing firmware, and confirming an attached actuator actually moves, then reporting pass or fail to Discord alongside the health of the published pip packages, images, and firmware. The friction record carries an effort estimate and an impact judgment on every entry, which makes it a backlog by construction, and the milestone spends a fixed 60-hour bucket working that backlog in priority order.

The milestone is deliberately open-ended in its second half. The bench test and the friction log are fixed deliverables; the backlog work is time-boxed rather than scope-fixed, because the value is in fixing the most friction per hour rather than in completing a list written before anyone had run the experiment.

## Why is this Important?

- Nothing currently reports that the stack is broken. The publish workflows build and upload artifacts, and the first signal that a package, image, or firmware build is unusable is a person trying to use it. A bench test that runs per commit turns that into a Discord message minutes after the break.
- Cold-start setup is the main barrier to adding a robot or an engineer. Every hour of undocumented fiddling is paid again by the next person, and the knowledge currently lives in whoever last did it.
- A scored backlog is what makes this work tractable for AI. Most friction items are small, self-contained, and well-specified once written down, which is exactly the shape of work an agent completes reliably.
- Confidence in every later milestone depends on it. Sim-to-real work, fleet growth, and firmware changes all rest on the assumption that a known-good robot can be produced on demand, and nothing currently proves that.
- It pairs with M21. M21 gives the firmware a simulator so it can be tested without hardware; M23 gives the whole stack a hardware bench so the parts a simulator cannot cover are tested too.

## Tasks

Total: ~20 working days, roughly 3-4 weeks part-time at 20-30 hours per week. Implementation detail and exact code locations live in the per-task files linked below; the acceptance criteria are here.

Task 2 is a time-boxed bucket rather than a fixed scope. Its acceptance turns on the bucket being worked in priority order with the remainder left documented, described in its criteria below.

### Task 1 - Cold-start bring-up, bench test, and build notifications -> [TASK-1-BENCH-TEST-AND-BRINGUP.md](TASK-1-BENCH-TEST-AND-BRINGUP.md)

#### Narrative

Task 1 builds a complete Krabby from bare hardware, recording every point of friction, then turns that same path into a bench test that runs continuously. The two halves are one task because automating the install is what forces every undocumented step into the open: a step that lives in someone's head survives a manual bring-up and fails a scripted one.

The manual pass runs from a bare Orin and unflashed MCUs through flashing, installing `krabby-launcher` from the public index, enrolling into the fleet, confirming the robot appears in the portal with live telemetry, verifying cameras, running a teleop session, and driving the legs with a Bluetooth controller. Every deviation from the documented path becomes a friction log entry carrying the symptom, the workaround, the component at fault, the cause where known, whether it blocks, what it costs and how often, a rough effort estimate, and whether an AI agent could work it. Those last four fields are what Task 2 packs its bucket from.

The bench half turns a second Orin with attached MCUs and at least one live actuator into a CI target. On every commit to the publishing branch and every published artifact, the harness resets to a known state, installs from the public index, reflashes firmware from the published artifact, brings the stack up, and drives a motor while asserting through telemetry that it moved. Results post to Discord with pass or fail, the triggering commit, and the stage that failed, so the message is actionable without opening the logs. Alongside it, cheaper checks report whether the published packages install cleanly, whether the locomotion image pulls and starts, and whether the firmware artifact matches the commit it claims.

The task closes by writing down the command sequence a working bring-up actually requires, including the undocumented steps, as the baseline Task 2 is measured against.

#### Acceptance Criteria

- 1a - A complete robot is brought up from a bare Orin and unflashed MCUs, appears in the fleet portal with live telemetry rather than a stale registration, streams cameras, completes a teleop session, and responds to a paired Bluetooth controller.
- 1b - A friction log records every deviation from the documented path, each entry carrying symptom, workaround, component, cause where known, whether it blocks, cost with a note on recurrence, effort in hours, and a marking of AI-suited, human, or hardware access.
- 1c - Items solved in under a minute are logged, since frequently hit small friction is what the cost-with-recurrence field exists to surface.
- 1d - A bench Orin with MCUs and at least one attached actuator is registered as a CI target and reachable from the pipeline without manual intervention between runs.
- 1e - The harness resets the Orin to a known state and installs `krabby-launcher` from the public package index with no manual steps, failing loudly if the documented install path has drifted.
- 1f - The harness reflashes the MCU firmware from the published artifact and confirms the flashed version matches it.
- 1g - The harness brings the stack up and drives at least one attached actuator, asserting motion through telemetry rather than through the absence of errors.
- 1h - Each run posts a Discord notification carrying pass or fail, the triggering commit, the failing stage where applicable, and a link to the run.
- 1i - A health check reports whether the published `krabby-*` packages install cleanly into a fresh environment, whether the locomotion image pulls and starts, and whether the published firmware artifact corresponds to the commit it claims.
- 1j - The harness runs on commit to the publishing branch and on artifact publication, and the branch it watches is the branch the publish workflows actually use.
- 1k - A failing run is reproducible by hand from the notification alone, with the commands the harness ran recorded in the run output.
- 1l - The commands a working bring-up actually requires are captured as a single ordered sequence, with the gap to the few-commands goal stated as a number.

#### Time estimate (~6 days)

| Days | Sub-task title |
|------|----------------|
| 0.5 | Cold-start bring-up by hand: Orin flash, `krabby` install, enrollment, portal, teleop, Bluetooth controller |
| 0.5 | Camera configuration and the remaining fiddly bits that resist a first pass |
| 1 | Bench harness: Orin as a CI target, fresh install, firmware reflash, bringup, motion assertion |
| 1 | Discord notifications and artifact health checks |
| 3 | Friction log and TODO consolidation, with effort estimates and AI-suitability marking, plus the working command sequence |

### Task 2 - Work the 60-hour improvement bucket -> [TASK-2-IMPROVEMENT-BUCKET.md](TASK-2-IMPROVEMENT-BUCKET.md)

#### Narrative

Task 2 orders the friction log and the known issues into one prioritized list, packs a 60-hour bucket from the top of it, and works that bucket top down.

Ordering is quick because Task 1's log already carries the scoring fields; the known issues need the same four fields added before they can sit alongside it. Priority is impact over effort, where impact combines how often friction is hit with what it costs each time, so a thirty-second annoyance on every bring-up outranks a two-hour problem hit once a year. The M21 hand-offs, covering publication of the IsaacSim image, absent-MCU detection, GPU validation, and IoT resume, are scored the same way as everything else and may legitimately fall below the line.

The bucket is time-boxed rather than scope-fixed. An item that runs over is finished, split so the valuable part lands, or returned with a revised estimate and a note on what the original missed, and actual hours are recorded either way so the next milestone estimates better. Each completed item is verified against the friction it targeted, which usually means repeating the relevant bring-up step and confirming it got shorter or disappeared.

The task closes with a cold-start bring-up compared against Task 1's command sequence, and a bench run confirming the improvements did not break the path they were meant to smooth.

#### Acceptance Criteria

- 2a - Every friction-log entry and every entry in `KNOWN-ISSUES.md` appears in one ordered list or carries an explicit out-of-scope marking with a reason, with known issues scored on the same fields the friction log uses.
- 2b - The list is ordered by impact over effort, with impact accounting for frequency as well as severity, and items that block bring-up weighted above items that slow it.
- 2c - The cut line is marked and the packed total is within 10 percent of 60 hours, with hardware-access items grouped and prerequisites kept on the same side of the line.
- 2d - 60 hours of work is spent against the list, worked in priority order, with actual hours recorded per item including returned ones and any departure from the order recorded and justified.
- 2e - Each completed item is verified against the friction it targeted, by repeating the relevant bring-up step and showing it got shorter, simpler, or absent, or by demonstrating the described behavior no longer occurs.
- 2f - Items that ran over their estimate are finished, split so the valuable part lands, or returned with a revised estimate and a note on what the original missed.
- 2g - The list is left current, with completed, returned, out-of-scope, and remaining items each distinguishable, and items below the line retained as the handoff to the next milestone.
- 2h - A closing cold-start bring-up is performed and compared against Task 1's command sequence, stating the change as numbers.
- 2i - The Task 1 bench test passes at the close of the bucket.

#### Time estimate (~14 days)

| Days | Sub-task title |
|------|----------------|
| 1 | Score the known issues onto the friction log's fields; order the combined list; pack and mark the 60-hour cut line |
| 11 | Work the bucket top down, 60 hours logged per item |
| 1 | Verify completed items against the friction they targeted |
| 1 | Closing cold-start bring-up, list reconciliation, bench run, comparison against Task 1 |

## Information

- Repositories: `krabby-research` (the `krabby` package, HAL, firmware, fleet client, images, and CI workflows), `patina-foundation-grants` (this grant), `krabby-contracts` (the milestone ICA).
- CI as it stands: `.github/workflows/publish-firmware.yml` (firmware to S3), `publish-locomotion.yml` (locomotion image to ECR Public), and `publish-packages.yml` (packages to PyPI on version tags). All three trigger on `mainline` and `release/**`, while the repository default branch is `main`.
- The `krabby` CLI: `krabby/install.py`, `krabby/run.py`, `krabby/update.py`, `krabby/firmware.py`, `krabby/enroll.py`, `krabby/agent.py`, `krabby/telemetry.py`, `krabby/_docker.py`, `krabby/_state.py`, and `krabby/__main__.py`.
- Fleet and teleop: `fleet/cli/`, `fleet/service/`, and the M10 IoT Core path (telemetry, teleop signaling, Secure Tunneling notifications).
- Firmware and MCU: `firmware/arduino/`, `firmware/krabby_mcu.py`, `firmware/mcu_port.py`, `firmware/Makefile`, and the published firmware artifact path used by `krabby firmware`.
- Known issues carried in from earlier work: [KNOWN-ISSUES.md](KNOWN-ISSUES.md) in this folder, which is ordered alongside Task 1's friction log at the start of Task 2.
- Hardware: a bench Orin with MCUs and at least one attached actuator to serve as the CI target, and a second complete robot for cold-start bring-up experiments, so the harness is not disturbed once it is running.

## Looking ahead

M23 leaves behind a bench test that proves the stack works on real hardware, a documented and much shorter path from bare Orin to working robot, and a scored backlog whose remainder is ready to be worked.

The natural successors are the items below Task 2's cut line, and the fleet-simulation work M21 deferred: publishing the IsaacSim image so a machine with no attached krab can pull it, detecting an absent MCU and offering the simulator instead, validating GPU capability before launch, and IoT-based resume when a simulated robot restarts. Those items enter this milestone's backlog as candidates rather than commitments, and their priority is decided by the same rubric as everything else.

## FAQ

- Why are the bring-up and the bench test one task? Because automating the install is the thing that finds the friction. A manual bring-up lets an operator paper over an undocumented step without noticing, and a scripted one fails on it. Doing them together also means the harness is running by the time Task 2 starts changing the install path.
- Why a real Orin with real motors rather than a simulated one? M21 covers the simulated path. What the bench catches is the class of failure a simulator cannot see: a firmware artifact that does not flash, a package that fails to install on aarch64, a service that starts in the wrong order on real hardware, a motor that does not move because a connector spec changed.
- What counts as a friction-log entry? Anything the operator had to do that the documentation did not tell them to do, anything the documentation told them to do that did not work, or anything that required setup steps other than 'use the self-documenting krabby package to set it up'. Small items count, and frequently hit small items are the ones the scoring rubric is designed to surface.
- Why is Task 2 time-boxed instead of scoped to a list of fixes? Because the list comes out of Task 1, after the experiment that reveals what the real friction is. Committing in advance to a fixed set of fixes would mean committing before anyone knows which ones matter. The hours and the ordering are the commitment.
- What happens to backlog items that are not reached? They stay in the ordered list below the cut line, with their scores, and become the starting point for the next milestone.
- How does an AI agent work these items? Task 1's friction log marks an entry AI-suited when it is self-contained and specifiable from the entry alone, which most small friction fixes are. Items needing hardware access or judgment about the physical robot are marked for a human.
- Which branch should the bench test watch? The one the publish workflows use. Today those are `mainline` and `release/**` while the default branch is `main`, and reconciling that is itself a known issue and a backlog candidate.
- Where are the full task details? In this folder: `TASK-1-BENCH-TEST-AND-BRINGUP.md` and `TASK-2-IMPROVEMENT-BUCKET.md`, with the pre-existing issues in `KNOWN-ISSUES.md`.
