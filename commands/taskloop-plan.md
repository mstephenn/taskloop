---
description: "Decompose a goal description or spec file into tasks and create them via the selected task-source adapter, for backlogs that are empty or don't cover what's needed"
argument-hint: "[\"<goal description>\" | <spec-file-path>] [source=docs|jira|asana|monday|linear]"
allowed-tools: ["Bash", "Read", "Write", "Edit", "Grep", "Glob"]
---

# Task Loop — Plan

Decomposes a goal description or an existing spec/PRD file into discrete
tasks, then creates them via the selected task-source adapter — the same
`docs`/`jira`/`asana`/`monday`/`linear` adapters `/taskloop` already reads
from (Appendix B of `commands/taskloop.md`). Use this when `/taskloop`
reports "no remaining tasks", or when the backlog exists but doesn't cover
what you actually want done next.

This command never branches, pushes, or opens a PR/MR — it only adds tasks
to a task source. It does not edit, reprioritize, or delete any existing
task.

## Arguments

Parse `"$ARGUMENTS"` before Step 1.

- **Positional argument** — either free text (a goal/feature description)
  or a path to an existing file. Test with `test -f "<arg>"` (after
  stripping any `source=` token) to decide which: if it resolves to a real
  file, `Read` it in full as the spec text; otherwise treat the argument
  text itself as the goal description.
- `source=<docs|jira|asana|monday|linear>` — which task-source adapter to
  use, same meaning as `/taskloop`. Defaults to `docs` if omitted.
- **No positional argument at all** (nothing left in `$ARGUMENTS` after
  removing any `source=` token): do not guess or invent a goal. Stop and
  ask the user: "What should I plan tasks for? Give me a description of
  the goal, or a path to a spec file." Wait for their reply before
  continuing — do not proceed with an assumed goal.

## Step 1 — Resolve the Task-Source Adapter

Read `source=` from `$ARGUMENTS` (default `docs`). For a non-`docs`
source, confirm its required environment variables (Appendix B of
`commands/taskloop.md`) are all set. If any are missing: **stop** and
report exactly which variable is missing — same pattern as `/taskloop`'s
own Step 0. (This command has no VCS adapter step — it never touches
git remotes, branches, or PRs.)

## Step 2 — Resolve the Input

Strip any `source=` token from `$ARGUMENTS`. If nothing remains, stop and
ask the user per the Arguments section above.

Otherwise, run `test -f "<remaining-argument>"`:
- If it exists: `Read` the file in full — this is the spec text to
  decompose.
- If not: treat the remaining argument text itself as the goal
  description to decompose.

## Step 3 — Load the Existing Backlog

Call the selected adapter's existing **list open tasks** operation
(`commands/taskloop.md` Appendix B) — read-only, no new adapter code
needed. Keep the result for the dedup check in Step 5.

## Step 4 — Decompose the Input into Tasks

Walk the input structurally so nothing is silently dropped:

- **Spec/PRD file:** treat each distinct requirement, section, or feature
  bullet as a candidate task boundary. Go section by section — do not
  summarize past a section without producing at least one candidate task
  for it.
- **Free-text goal description:** identify each distinct piece of work
  required to deliver the stated goal (e.g. "add dark mode support" ->
  theme/state, styling, persistence, UI control).

For each candidate task, produce:
- a short title
- a one-line goal
- a "why" (context)
- an explicit acceptance-criteria checklist (`- [ ]` items) stating
  exactly what "done" means for that task — this is what `/taskloop`'s
  own Step 4 will later read and satisfy.

## Step 5 — Cross-Check Against the Existing Backlog

Using the list loaded in Step 3: for each candidate task from Step 4, if
an existing open task's title/goal already substantially covers it, drop
that candidate. Note it (with the existing task's id) as "already
covered" for the final report — do not create a duplicate for it.

## Step 6 — Present the Proposed List and Confirm

Print the remaining candidates (after Step 5's dedup) as:

```
<task-id>  <title>  — <one-line goal>
```

one row per candidate. For the `docs` adapter, `<task-id>` is the real id
this command will assign, computed from Step 8's placement rules
(Appendix B's `docs` row), so it can be shown now. For `jira`/`asana`/
`monday`/`linear`, no id exists yet at this point — no ticket has been
created, and the tracker assigns the id only when Step 8 actually creates
it — so show the literal placeholder `(new)` in that column instead of a
task-id. Follow the rows with a line listing anything dropped as
"already covered" in Step 5. Ask the user to confirm:

- **Accept as-is** — proceed to Step 7.
- **Reject** — stop here. Nothing is created, no partial writes.
- **Edits** (e.g. drop a row, reword a title) — apply them to the list,
  then re-present the updated list and ask again.

Do not proceed past this step without an explicit accept.

## Step 7 — `docs` Adapter Only: Ask About Git Tracking

Skip this step entirely for `jira`/`asana`/`monday`/`linear` — go straight
to Step 8.

For `docs`, before writing any file:

1. Check `git ls-files docs/planning/ | head -1` — if it prints a path,
   `docs/planning/` is already tracked; skip to Step 8 and write files
   normally (tracked).
2. Otherwise check whether `.gitignore` already has an entry matching
   `docs/planning/` (a `docs/planning/`, `docs/planning`, or broader
   `docs/` line). If so, skip to Step 8 and write files as already
   covered by that ignore rule.
3. If neither is true — this is the first time this repo would get a
   `docs/planning/` tree — **ask the user**: "This repo doesn't have
   `docs/planning/` yet. Should the new task docs be committed normally,
   or added to `.gitignore` as local-only planning notes?" Do not infer
   the answer from any other `docs/` subdirectory's tracked/ignored
   status. Wait for their answer:
   - **Committed:** proceed to Step 8, do not touch `.gitignore`.
   - **Gitignored:** append a `docs/planning/` line to `.gitignore`
     (create `.gitignore` if it doesn't exist) before proceeding to
     Step 8.

## Step 8 — Create the Tasks

Call the selected adapter's **create task(s)** operation
(`commands/taskloop.md` Appendix B) once per approved candidate, in the
order presented in Step 6, respecting Step 7's answer for the `docs`
adapter. If a single creation call fails (ticket-tracker adapters only):
stop immediately, and in the report (Step 9) list exactly which tasks
were already created successfully before the failure and which one
failed. Do not continue past a failed creation, and do not roll back
tasks already created — they're valid on their own.

## Step 9 — Report

Report:
- every task created, with its final `<task-id>` (ticket trackers assign
  their own id; for `docs`, the id is the one this command assigned per
  the placement rules in Appendix B's `docs` row) — this is where the
  real ids for any `(new)` placeholder rows shown in Step 6 first appear
- everything skipped as "already covered" in Step 5, with the existing
  task id it matched
- for a first-time `docs` run only: whether the new docs were committed
  or gitignored per Step 7
