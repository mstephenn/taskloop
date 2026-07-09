# taskloop

[![Claude Code](https://img.shields.io/badge/Claude%20Code-plugin-5A32FB)](https://github.com/mstephenn/taskloop)
[![automation](https://img.shields.io/badge/automation-ci--cd-blue)](https://github.com/mstephenn/taskloop)

A Claude Code plugin providing `/taskloop`: an unattended
branch -> implement -> push -> PR -> automated review -> rework -> merge ->
next-task loop, pluggable across VCS hosts and task trackers. Also
provides `/taskloop-plan`, for generating tasks when the backlog is empty
or doesn't cover what's needed yet.

## What it does

1. Finds the next open task from the selected **task source** — this
   repo's `docs/planning/phase-*/sprint-*/task-*.md` checkbox docs by
   default, or Jira, Asana, Monday.com, or Linear via `source=`.
2. Branches, implements it, and records completion back in the task
   source (checkbox flip for `docs`; a status transition/close for the
   ticket trackers).
3. Commits, pushes, and opens a PR/MR via the selected **VCS adapter** —
   GitHub, Azure DevOps, or GitLab, auto-detected from the repo's `origin`
   remote, with org/project/repo identifiers parsed from the remote URL
   rather than hardcoded. Uses each host's CLI (`gh`/`az`/`glab`) when
   installed and authenticated, and falls back to that host's REST/GraphQL
   API directly when the CLI isn't available.
4. Runs `/code-review` against the diff as the actual quality gate (no
   human reviewer is assigned). Reworks up to 3 rounds on findings.
5. Merges the PR/MR and repeats, until every task is done or the loop hits
   a stop condition (stuck task, merge failure, or an unresolvable
   VCS/task adapter).

A task that's already marked blocked by its source (e.g. a `docs` task doc
with a `Status: BLOCKED` header, a Jira issue in a "Blocked" status, a
labeled Asana/Linear/Monday item) — or one discovered mid-implementation
to need something outside the loop's control (human sign-off, an
unresolved external precondition) — is never implemented or marked done.
It's recorded to a local `.taskloop/blocked-tasks.json` file
and the loop moves straight to the next task, instead of halting and
making you re-pass `skip=` for it on every future run. Bring one back into
scope deliberately with `unblock=<task-id>`.

See `commands/taskloop.md` for the exact step-by-step contract
the command follows — the numbered steps are the fixed control flow;
Appendix A (VCS adapters) and Appendix B (task-source adapters) hold the
per-provider command sets so switching providers never changes the steps
themselves.

## Install

```
/plugin marketplace add mstephenn/taskloop
/plugin install taskloop@taskloop-marketplace
```

## Usage

```
/taskloop
/taskloop skip=0.1.3
/taskloop source=linear
/taskloop skip=ENG-12,ENG-19 source=linear
/taskloop unblock=0.1.3

/taskloop-plan "add dark mode support"
/taskloop-plan docs/specs/dark-mode.md
/taskloop-plan "add dark mode support" source=linear
```

## Requirements

**Task source** (`source=`, default `docs`):
- `docs` — a repo with `docs/planning/phase-*/sprint-*/task-*.md` task
  docs using `- [ ]` / `- [x]` acceptance-criteria checkboxes. No
  credentials needed.
- `jira` — `JIRA_BASE_URL`, `JIRA_EMAIL`, `JIRA_API_TOKEN`,
  `JIRA_PROJECT_KEY` env vars.
- `asana` — `ASANA_ACCESS_TOKEN`, `ASANA_PROJECT_GID` env vars.
- `monday` — `MONDAY_API_TOKEN`, `MONDAY_BOARD_ID`,
  `MONDAY_STATUS_COLUMN_ID` env vars.
- `linear` — `LINEAR_API_KEY`, `LINEAR_TEAM_KEY` env vars.

**VCS host** (auto-detected from `origin`'s remote — only the matching row
is needed):
- GitHub — `gh` CLI (authenticated), or `GH_TOKEN`/`GITHUB_TOKEN` as an API
  fallback.
- Azure DevOps — `az` CLI with the `azure-devops` extension
  (authenticated), or `AZURE_DEVOPS_PAT` as an API fallback.
- GitLab — `glab` CLI (authenticated), or `GITLAB_TOKEN` as an API
  fallback.

An unresolvable VCS host, missing CLI/token, or missing task-source
credential halts the loop before it starts, rather than guessing or
partially running.

Also required: a `/code-review` skill or slash command available in the
session (this plugin does not ship one — it assumes the target repo or
another installed plugin provides it).

## Guardrails

- Never commits to or pushes the default branch directly.
- Never force-pushes.
- Never prints, logs, or embeds an API token in a commit message, PR/MR
  body, or branch name.
- Rework is capped at 3 rounds per task; exceeding it halts the whole loop,
  not just that task. This is deliberately kept a full halt (never
  auto-skipped) — a task failing review repeatedly is a loop/process
  problem worth surfacing, not an external precondition to remember past.
- A task that's blocked (pre-marked or discovered mid-implementation) is
  never marked done and never implemented — recorded to blocked memory and
  skipped instead, automatically, on every run until explicitly unblocked.
