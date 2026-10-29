# Task 2 — Work the 60-hour improvement bucket

Goal: order the friction log and known issues into one prioritized list, pack a 60-hour bucket from the top of it, and work that bucket, verifying each completed item against the friction it was meant to remove and closing with a cold-start bring-up that states how much shorter the path became.

Outputs
- A single ordered list combining the Task 1 friction log with `KNOWN-ISSUES.md`, scored by value per unit of effort, with the 60-hour cut line marked.
- 60 hours of work against that list, logged per item and worked in priority order.
- Each completed item verified against the bring-up step it targeted.
- The list left current, showing what was completed, what was returned with a revised estimate, and what remains as the handoff to the next milestone.
- A closing cold-start bring-up compared against Task 1's command sequence.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-2---work-the-60-hour-improvement-bucket).)

---

Note: this task is time-boxed rather than scope-fixed. The commitment is the hours and the ordering; the number of items completed is an outcome. Its inputs are the friction log from [TASK-1-BENCH-TEST-AND-BRINGUP.md](TASK-1-BENCH-TEST-AND-BRINGUP.md) and [KNOWN-ISSUES.md](KNOWN-ISSUES.md).

---

## 1. Building the ordered list

Task 1's friction log already carries the fields this needs: blocking, cost with a note on whether it recurs, effort in hours, and a marking of AI-suited, human, or hardware. Entries in `KNOWN-ISSUES.md` carry a location and a rationale but no scores, so they need the same four fields added before they can be ordered alongside the log.

The M21 hand-offs enter here too: publishing the IsaacSim image, detecting an absent MCU and offering the simulator, GPU capability validation before launch, and IoT-based resume. They are scored the same way as everything else, and it is a legitimate outcome for them to fall below the cut line.

Priority is impact over effort, where impact combines how often the friction is hit with what it costs each time: a thirty-second annoyance on every bring-up costs more per year than a two-hour problem hit once. Weight items that block bring-up above items that merely slow it, since a blocker stops a new robot or a new engineer entirely.

Sort by priority, then sanity-check the top of the list by hand; an ordering that looks wrong at the top usually means a frequency judgment needs revisiting.

## 2. Packing the bucket

Pack top down against the effort estimates until the total reaches roughly 60 hours, and mark the line; within 10 percent is close enough given the precision of the estimates.

Two adjustments are worth making by hand: group hardware-access items, since the setup cost is real and the estimates usually omit it, and check whether any item just below the line is a prerequisite for one above it, in which case both move or neither does.

Leave everything below the line in the document with its scores intact; the remainder is what the next milestone starts from.

## 3. Working order

Work top down; departing from the ordering discards the reason the scoring was done.

Two departures are legitimate when recorded: hardware-access items can be grouped so one bench session covers several, and an item can be pulled forward when a later item depends on it and the packing pass missed the dependency.

If an item near the top turns out to be a bad idea on closer inspection, record why, mark it out of scope, and move on rather than quietly skipping it.

AI-suited items are worth batching: running several through an agent in one pass is more efficient than interleaving them with work that needs the bench. Each still needs its verification step from section 5.

## 4. When an estimate is wrong

An item that runs over its estimate is finished, split, or returned. Finish it if the overrun is modest and the item is nearly done. Split it if part of it delivers value on its own: land that part and return the rest to the list as a new entry with its own estimate. Return it if it turns out substantially larger or depends on something unavailable, with a revised estimate and a note on what the original missed — returning early beats spending the bucket discovering it.

Record actual hours against every item, including returned ones; that record is what tells the next milestone whether these estimates were systematically optimistic.

## 5. Verifying a fix removed friction

An item is complete when the friction it targeted is gone, which is a different claim from the code having changed.

For items from the friction log, verification means repeating the relevant bring-up step and confirming it is shorter, simpler, or absent. The log entry records the symptom and the cost, so the verification compares against a number that already exists.

For items from `KNOWN-ISSUES.md`, verification means demonstrating the behavior the issue described no longer occurs. The unassigned contact sensor is verified by reading `self.contact_sensor` and getting a sensor; the absent per-commit test run is verified by a failing test blocking a commit.

Note where a fix is also covered by the Task 1 bench test, since those items gain a standing regression check and need no separate future verification.

Update the friction log entry as well as the list entry. The log is what the closing bring-up is measured against, and an entry fixed but not marked will be counted twice.

## 6. Closing measurement

End the bucket by bringing up a robot cold, following the current documentation, and writing the command sequence the way Task 1 did.

Compare the two sequences directly: how many commands, how many undocumented steps, how many manual interventions, and how long it took. State the result as numbers, including the case where it barely moved.

Run the Task 1 bench test at the close as well. The bucket changes the install path, the firmware path, and the bringup path, which are exactly what the bench asserts, so a green run is what shows the improvements did not break the thing they were smoothing.

## 7. Verify

- Every friction-log entry and every known issue appears in the ordered list or carries an explicit out-of-scope marking with a reason.
- Each item carries blocking, cost with recurrence, effort in hours, and a marking of AI-suited, human, or hardware access.
- Picking any three AI-suited items at random, each has enough detail in the entry alone to be worked without asking a question.
- The list is ordered by priority, the cut line is marked, and the packed total is within 10 percent of 60 hours.
- 60 hours are logged against items, with actual hours recorded per item including returned ones.
- The work followed priority order, with any departure recorded and justified.
- Each completed item has a verification note showing the targeted friction is gone, measured against the log entry or the known-issue description.
- Over-running items were finished, split with the valuable part landed, or returned with a revised estimate and a note on what the original missed.
- The list is current: completed, returned, out-of-scope, and remaining items are each distinguishable.
- A closing cold-start bring-up is recorded and compared against Task 1's sequence, with the change stated as numbers.
- The Task 1 bench test passes at the close of the bucket.
