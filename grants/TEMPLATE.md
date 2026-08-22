# Grant Template

Copy the two skeletons below into a new grant directory: the first becomes `OVERVIEW.md`, the second becomes each `TASK-N-SHORT-NAME.md`. Angle-bracket text is a placeholder. Write both to the rules in [GUIDE.md](GUIDE.md).

---

## Skeleton — OVERVIEW.md

```markdown
# Patina Foundation Grant — <Program> Milestone <NN>: <Title>

## Grant Overview

<What the milestone delivers, in one paragraph. State the outcome a reader
cares about before the mechanism.>

<Where things stand today and why that blocks the outcome.>

<How the milestone closes the gap. Describe the core abstractions and major 
design changes.>

<How the work builds up across the tasks, in one paragraph, so the reader
sees the arc before meeting the task list.>

<One sentence naming what exists at the end.>

## Why is this Important?

- <Concrete consequence of the current state, and what this milestone
  changes about it.>
- <What it unblocks that is impossible today.>
- <What it closes, completes, or de-risks in the wider system.>
- <What it enables for later milestones.>

## Tasks

Total: ~<N> working days, roughly <N>-<N> weeks part-time at <N>-<N> hours
per week. Implementation detail and exact code locations live in the
per-task files linked below; the acceptance criteria are here.

### Task 1 - <Short title> → [TASK-1-<SHORT-NAME>.md](TASK-1-<SHORT-NAME>.md)

#### Narrative

<What this task builds, in prose. Name the functions, files, and values
involved. Where a boundary with another task matters, say where that work
lives rather than what is absent here.>

<Second paragraph: the deliverable, and how someone confirms it works.>

#### Acceptance Criteria

- 1a — <Independently verifiable outcome, described in 1-2 sentenes, 'The X is able to Y.'>
- 1b — <...>
- 1c — <...>

#### Time estimate (~<N> days)

| Days | Sub-task title |
|------|----------------|
| <N> | <Sub-task> |
| <N> | <Sub-task> |

## FAQ

- <Question a grantee will actually ask?> <Answer in the same bullet.>
- <Question?> <Answer.>
```

---

## Skeleton — TASK-N-SHORT-NAME.md

```markdown
# Task <N> — <Short title>

Goal: <One or two sentences on what this task produces and how it is
confirmed.>

Outputs
- <Artifact this task leaves behind.>
- <Artifact.>
- <Tests, and what they assert.>

(Acceptance criteria and time estimate for this task live in
[OVERVIEW.md](OVERVIEW.md#task-<N>---<generated-anchor>).)

---

Note: exact line numbers below are from the repo at the time of
writing and may drift. Treat them as pointers to the right function.

---

## 1. <The seam — what existing code this task attaches to>

<Prose describing the current behavior, with a code excerpt where it saves
explanation.>

| <Interface> | <Called at> | <Responsibility here> |
|---|---|---|
| <symbol> | <file ~Lnn> | <what this task does about it> |

## 2. <The data — pin map, wire format, register set, schema>

<Table or excerpt of the concrete values, sourced from the real header or
spec rather than re-typed.>

## 3. <The build or harness this task stands up>

<What exists today, what this task adds, and the smallest version that
satisfies the goal.>

## 4. <Testing approach>

<What the tests drive, what they assert, and how a human confirms it by
eye where that applies.>
```
