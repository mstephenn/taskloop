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
  use, same meaning as `/taskloop`. If omitted, resolved automatically per
  Appendix C of `commands/taskloop.md`; falls back to `docs` if nothing
  else resolves it.
- **No positional argument at all** (nothing left in `$ARGUMENTS` after
  removing any `source=` token): do not guess or invent a goal. Stop and
  ask the user: "What should I plan tasks for? Give me a description of
  the goal, or a path to a spec file." Wait for their reply before
  continuing — do not proceed with an assumed goal.

## Step 1 — Resolve the Task-Source Adapter

Resolve `<source>` per Appendix C of `commands/taskloop.md` (reads
`source=` from `$ARGUMENTS` first; if omitted, checks
`.taskloop/config.json`, then whether `docs/planning/` is already
established, then scans README/AGENTS.md/CLAUDE.md/CODEX.md, asking only
if none of those resolve it). Appendix C also validates required env vars for
non-`docs` sources and persists the resolution to `.taskloop/config.json`
so a choice made here is remembered by `/taskloop` too. If a required
variable is missing: **stop** and report exactly which variable is
missing — same pattern as `/taskloop`'s own Step 0. (This command has no
VCS adapter step — it never touches git remotes, branches, or PRs.)

## Step 2 — Resolve the Input

Strip any `source=` token from `$ARGUMENTS`. If nothing remains, stop and
ask the user per the Arguments section above.

Otherwise, run `test -f "<remaining-argument>"`:
- If it exists: `Read` the file in full — this is the spec text to
  decompose.
- If not: treat the remaining argument text itself as the goal
  description to decompose.

## Step 3 — Ask Clarifying Questions

This step always runs, whether the input from Step 2 is a one-line goal
or a fully written spec file — a detailed-looking input can still hide
ambiguity that would otherwise surface as a wrong or incomplete task
list. Never skip straight from Step 2 to Step 4 (Load the Existing
Backlog).

1. Read the Step 2 input and draft a candidate list of clarifying
   questions covering, at minimum: scope boundary (what's explicitly in
   vs. out), constraints or non-goals, priority/ordering across the work,
   and definition of done / acceptance criteria. Add more candidates if
   the input leaves anything else ambiguous (e.g. target users, affected
   surfaces, rollout expectations).
2. Drop any candidate the input already answers unambiguously — quote or
   paraphrase the answer already given so it's clear why that question
   was dropped.
3. Cap the remaining list at 5 questions. If more than 5 remain
   genuinely open, keep the 5 most decomposition-blocking ones (the ones
   most likely to change which tasks get created or what their
   acceptance criteria say) and drop the rest — do not ask more than 5.
4. If zero questions remain after step 2 (the input already answers all
   of them), state which questions were considered and why each is
   already answered, and proceed straight to Step 4 without waiting for
   a reply.
5. Otherwise, ask the user all remaining questions together in one
   message. Wait for their reply before proceeding to Step 4. Use their
   answers to inform decomposition in Step 5 — do not re-ask anything
   already covered by an answer here.

## Step 4 — Load the Existing Backlog

Call the selected adapter's existing **list open tasks** operation
(`commands/taskloop.md` Appendix B) — read-only, no new adapter code
needed. Keep the result for the dedup check in Step 6.

## Step 5 — Decompose the Input into Tasks

Walk the input structurally so nothing is silently dropped:

- **Spec/PRD file:** treat each distinct requirement, section, or feature
  bullet as a candidate task boundary. Go section by section — do not
  summarize past a section without producing at least one candidate task
  for it.
- **Free-text goal description:** identify each distinct piece of work
  required to deliver the stated goal (e.g. "add dark mode support" ->
  theme/state, styling, persistence, UI control).

Incorporate any answers gathered in Step 3 — they may narrow scope, add
constraints, or change acceptance criteria for individual candidates.

For each candidate task, produce:
- a short title
- a one-line goal
- a "why" (context)
- an explicit acceptance-criteria checklist (`- [ ]` items) stating
  exactly what "done" means for that task — this is what `/taskloop`'s
  own Step 4 will later read and satisfy.

## Step 6 — Cross-Check Against the Existing Backlog

Using the list loaded in Step 4: for each candidate task from Step 5, if
an existing open task's title/goal already substantially covers it, drop
that candidate. Note it (with the existing task's id) as "already
covered" for the final report — do not create a duplicate for it.

## Step 7 — Present the Proposed List and Confirm

Print the remaining candidates (after Step 6's dedup) as:

```
<task-id>  <title>  — <one-line goal>
```

one row per candidate. For the `docs` adapter, `<task-id>` is the real id
this command will assign, computed from Step 9's placement rules
(Appendix B's `docs` row), so it can be shown now. For `jira`/`asana`/
`monday`/`linear`, no id exists yet at this point — no ticket has been
created, and the tracker assigns the id only when Step 9 actually creates
it — so show the literal placeholder `(new)` in that column instead of a
task-id. Follow the rows with a line listing anything dropped as
"already covered" in Step 6. Ask the user to confirm:

- **Accept as-is** — proceed to Step 8.
- **Reject** — stop here. Nothing is created, no partial writes.
- **Edits** (e.g. drop a row, reword a title) — apply them to the list,
  then re-present the updated list and ask again.

Do not proceed past this step without an explicit accept.

## Step 8 — `docs` Adapter Only: Ask About Git Tracking

Skip this step entirely for `jira`/`asana`/`monday`/`linear` — go straight
to Step 9.

For `docs`, before writing any file:

1. Check `git ls-files docs/planning/ | head -1` — if it prints a path,
   `docs/planning/` is already tracked; skip to Step 9 and write files
   normally (tracked).
2. Otherwise check whether `.gitignore` already has an entry matching
   `docs/planning/` (a `docs/planning/`, `docs/planning`, or broader
   `docs/` line). If so, skip to Step 9 and write files as already
   covered by that ignore rule.
3. If neither is true — this is the first time this repo would get a
   `docs/planning/` tree — **ask the user**: "This repo doesn't have
   `docs/planning/` yet. Should the new task docs be committed normally,
   or added to `.gitignore` as local-only planning notes?" Do not infer
   the answer from any other `docs/` subdirectory's tracked/ignored
   status. Wait for their answer:
   - **Committed:** proceed to Step 9, do not touch `.gitignore`.
   - **Gitignored:** append a `docs/planning/` line to `.gitignore`
     (create `.gitignore` if it doesn't exist) before proceeding to
     Step 9.

## Step 9 — Create the Tasks

Call the selected adapter's **create task(s)** operation
(`commands/taskloop.md` Appendix B) once per approved candidate, in the
order presented in Step 7, respecting Step 8's answer for the `docs`
adapter. If a single creation call fails (ticket-tracker adapters only):
stop immediately, and in the report (Step 10) list exactly which tasks
were already created successfully before the failure and which one
failed. Do not continue past a failed creation, and do not roll back
tasks already created — they're valid on their own.

## Step 10 — Report

Report:
- every task created, with its final `<task-id>` (ticket trackers assign
  their own id; for `docs`, the id is the one this command assigned per
  the placement rules in Appendix B's `docs` row) — this is where the
  real ids for any `(new)` placeholder rows shown in Step 7 first appear
- everything skipped as "already covered" in Step 6, with the existing
  task id it matched
- for a first-time `docs` run only: whether the new docs were committed
  or gitignored per Step 8
