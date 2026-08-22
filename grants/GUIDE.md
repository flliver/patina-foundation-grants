# Grant Document Guide

How to write a Patina Foundation grant so a grantee can pick it up, understand the whole job from one document, and start work without asking questions.

## The document set

A grant lives in one directory under `grants/<Program>/<MilestoneNN-Short-Name>/` and contains:

- `OVERVIEW.md` — the whole grant. Someone who reads only this file understands the goal, why it matters, every task, and what done means for each.
- `TASK-N-SHORT-NAME.md` — one per task, numbered in execution order, describing how to implement that task against the actual codebase.
- `AUDIT-LOG.md` — posting date, grantor, steward, assigned grantee, and notes on any partially completed work.

## OVERVIEW.md structure

The sections run in this order.

* Grant Overview: States what the milestone delivers and how it works. A reader finishing this section should be able to describe the design to someone else.
* Why is this Important? Makes the case for spending the money. Each point states a concrete consequence rather than a virtue.
* Tasks: A task is roughly a week of work part time, can be independently completed (potentially with a dependency on an earlier task). Each task subsection links to its task file and carries three parts:
- Narrative — what this task builds and how it fits the milestone, in prose.
- Acceptance Criteria — a lettered list keyed to the task number. Generally this should be 1-2 sentences maximum, and be an easily understood pass/fail result. The combination of all AC should fully validate the task (and milestone).
- Time estimate — a heading carrying the total days, then a table of days against sub-task titles. Tasks land around a week of part-time work; anything much larger usually warrants being split into multiple tasks.
* FAQ: Is a number list of questions and answers, as if they were asked by a grantee. They will typically clarify conceptual models, major redesign reasoning, answer questions about why one component or technology was selected vs another, highlight major dependencies, or clarify potential gotchas. Each FAQ is formulated as a question a grantee would ask, followed by its answer.

## Task document structure

A task document opens with a short goal statement and the outputs it produces, pointing back to the overview for acceptance criteria and time estimate instead of repeating them.

The body is numbered sections, 1 through as many as the task needs, each covering one key detail of the design, all the way down to specific lines of code and suggestions of restructures needed. i.e. the classes being modified, the pin map or wire format, the host build setup, the test approach. Real code excerpts, file paths, and tables belong here wherever they save explanation.

The purpose of this document is to give an initial set of guiding design choices to the AI models that will be performing the work, as well as to give the grantee strong suggestions on how to perform necessary refactors, with the intention of reducing wasted effort on designs and redesigns where intentions were not clear.

This should also clarify where any other milestones may overlap, and any long term design principles to keep in mind that may impact future milestones (i.e. define why a particular interface should be used and why, deep dive into pertintent details about technology choice and setup, etc.)

Where line numbers appear, note once that they point at a function rather than a literal address, since they drift as the code changes.

## Dividing content between the two

The overview answers what and why. The task document answers how.

A grantee reading only the overview knows what to build and how the work will be judged. A grantee reading a task document learns which file to open, which function to change, and what the existing code does today.

When the same fact appears in both, cut it from the task document and let the overview own it.

## Writing style

Write in complete sentences and connected paragraphs. The document should read as a complete english narrative, with complete, but concise reasoning. Use lists when items are genuinely parallel, e.g. acceptance criteria. 

State what a task does. A sentence whose content is the absence of something gives the reader nothing to act on, e.g here is an anti-pattern:

    Nothing on the read side is in Task 1. There is deliberately no
    position feedback, no closed-loop P-controller, and no calibration.

The exception to this is where the reader would normally assume something should be implemented, but we're intentionally not implementing it. Instead, just state what needs to be done once, in the task appropriate for that work.

Say a thing once. Announcing a count and then re-listing the items doubles the length and halves the trust, e.g. the following is an anti-pattern:

    Build just two things: the RobotCore game loop and the output pin
    calls. ... Scope: SimulatedRobotConfig + SimulatedActuator +
    RobotCore, one board, 6 motors.

Fold the specifics into the sentence that introduces them:

    Task 1 builds the RobotCore game loop together with the Arduino
    output calls: pinMode, digitalWrite, analogWrite, and millis for
    the loop clock.

* Leave out bolded text. 
* Avoid the constructions that read as machine-written. The worst offenders are `just...`, a `Scope:` prefix standing in for a sentence, and the "X, not Y" contrast where a plain positive statement works.
* Prefer concrete numbers, real symbol names, and actual paths over descriptions of them.
* Avoid duplication and restatement across narrative, AC, and tasks. After writing the full doc, review for duplicity and move each concept to where it makes the most sence to be.

## Conventions

* Task files are named `TASK-N-SHORT-NAME.md`, capitals with hyphens.
* Acceptance criteria are lettered within their task: 1a through 1f for Task 1, 2a onward for Task 2.
* Time estimates use whole and half days in a two-column table, with the section heading carrying the total.
* Task headings in the overview include the arrow link to the task file. Keep heading text stable once written, since task files link back to the generated anchor.
