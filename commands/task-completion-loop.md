---
description: "Unattended branch -> implement -> push -> PR -> automated review -> rework -> merge -> next-task loop, pluggable across VCS hosts and task trackers"
argument-hint: "[skip=<task-id>,...] [source=docs|jira|asana|monday|linear]"
allowed-tools: ["Bash", "Read", "Write", "Edit", "Grep", "Glob", "Skill"]
---

# Task Completion Loop

Unattended branch -> implement -> push -> PR -> automated review -> rework ->
merge -> next-task loop. Invoked explicitly only — never auto-triggered —
because it pushes branches and merges pull requests.

This command is a fixed 8-step control flow (below) over two swappable
adapters:

- **VCS adapter** — how PRs/MRs get opened and merged. Auto-detected from
  `origin`'s remote host: GitHub, Azure DevOps, or GitLab. Uses the host's
  CLI (`gh`/`az`/`glab`) when installed and authenticated, and falls back
  to that host's REST/GraphQL API directly (via `curl`) when the CLI isn't
  available. Full command sets for both modes are in **Appendix A**.
- **Task source adapter** — where tasks come from and how "done" is
  recorded. Defaults to this repo's `docs/planning/phase-*/sprint-*/task-*.md`
  checkbox docs; can instead point at Jira, Asana, Monday.com, or Linear.
  Full command sets for each are in **Appendix B**.

Steps 1 and 4 call into whichever task source adapter is selected; Steps 5
and 7 call into whichever VCS adapter is selected. The control flow itself
(branch, plan, implement, review, rework cap, merge, repeat) never changes.

## Arguments

Parse `"$ARGUMENTS"` before Step 0. Both are optional and can combine, e.g.
`skip=0.1.3 source=linear`.

- `skip=<task-id>,<task-id>,...` — task IDs to skip over in Step 1's scan,
  for tasks already known to be blocked on something outside this loop's
  control (see Stop Conditions). Skipped tasks are listed in the final
  report; they are never implemented or marked done by this loop.
- `source=<docs|jira|asana|monday|linear>` — which task source adapter to
  use. Defaults to `docs` (this repo's `docs/planning/` tree) if omitted.
  Non-`docs` sources also require the environment variables listed for
  that adapter in Appendix B — if they're missing, this is a Step 0 stop
  condition, not a per-task failure.

## Guardrails (read first)

- Never commit to or push the default branch directly.
- Never force-push.
- Never delete branches other than the one just merged for the current task.
- No PR reviewer is set; `/code-review` is the actual quality gate, not a
  human approval.
- Rework is capped at 3 rounds per task. Exceeding the cap halts the
  **entire** loop (not just that task) and reports the stuck task +
  findings to the user.
- A task acceptance criterion the loop cannot satisfy autonomously (e.g.
  "reviewed and explicitly signed off" by a human, an unresolved external
  precondition) is a stop condition for that task: halt and escalate, do
  not mark it done and do not skip ahead — unless that task ID was passed
  in `skip=`, in which case move to the next candidate instead.
- Never print, log, or embed an API token in a commit message, PR/MR body,
  or branch name. Tokens are read from environment variables only (see
  Appendix A/B) and used solely in `Authorization`/`PRIVATE-TOKEN` headers.

## Step 0 — Resolve Adapters

Derive everything about the target repo, its VCS host, and its task source
from the environment; never hardcode an org/project/repo name or assume
one provider. Do this once per loop run and reuse the result — do not
re-detect per task.

1. **VCS adapter.** Run `git remote get-url origin` and match its host
   against Appendix A's detection table to pick `<vcs-provider>`. Then
   check whether that provider's CLI is installed *and* authenticated
   (commands in Appendix A); if so, `<vcs-mode> = cli`. If the CLI is
   missing or unauthenticated, check for that provider's API token env var
   (Appendix A); if present, `<vcs-mode> = api`. If neither the CLI nor a
   token is available, or the host doesn't match any known provider: **stop
   the loop** before Step 1 and report the remote URL / missing
   credential to the user — ask how to proceed rather than guessing.
2. **Task source adapter.** Read `source=` from `$ARGUMENTS` (default
   `docs`). For a non-`docs` source, confirm its required environment
   variables (Appendix B) are all set. If any are missing: **stop the
   loop** before Step 1 and report exactly which variable is missing.
3. Determine the default branch (`main` or `master`) via
   `git symbolic-ref refs/remotes/origin/HEAD` (or default to `main` if
   that's unset and `main` exists on the remote).

## Step 1 — Find the Next Task

Call the selected task source adapter's **list open tasks** operation
(Appendix B), which returns an ordered list of `(task-id, task-slug,
title, detail-ref)` tuples — already sorted in that adapter's natural
priority order, already excluding anything already done.

1. Remove any task whose `<task-id>` appears in `skip=`.
2. The first remaining task is this iteration's task. Fetch its full
   detail (goal, description/acceptance-criteria, any implementation
   notes) via the adapter's **get task detail** operation.
3. Stop condition: if the list is empty after removing skipped tasks,
   **stop the loop** — report "no remaining tasks" (listing anything
   skipped) to the user. This is the success/completion stop condition.

## Step 2 — Branch

```bash
git checkout <default-branch>
git pull origin <default-branch>
git checkout -b task/<task-id>-<task-slug>
```
Never skip the `pull` — branching from a stale local default branch risks
a PR that conflicts or silently drops someone else's merge.

## Step 3 — Plan

Before writing code, write a short plan using the same structure as Plan
Mode: problem/goal, approach, and files you'll touch and why. This plan is
**not** submitted for approval and does not block the loop — write it, then
proceed straight to Step 4. Fold it directly into the PR description
written in Step 5.

Keep it short (3-6 sentences) for small tasks; expand only as far as the
task's acceptance criteria actually require.

## Step 4 — Implement

1. Read the task detail fetched in Step 1 in full — goal, context/why, any
   implementation plan, and every acceptance criterion.
2. Implement the task: write the code and tests needed to satisfy every
   acceptance criterion.
3. Record completion via the task source adapter's **mark done** operation
   (Appendix B). For the `docs` adapter this means flipping each satisfied
   `- [ ]` to `- [x]` individually as you go; for ticket-tracker adapters
   (Jira/Asana/Monday/Linear) there is no per-criterion checkbox to flip —
   completion is recorded once, at the end of this step, as a single
   status transition/close on the ticket after every criterion in its
   description is satisfied.
4. If any acceptance criterion requires something the loop cannot do
   itself (external human sign-off, an unresolved precondition owned by
   someone else, an ambiguous spec gap) — **stop the loop** here, do
   **not** call the adapter's mark-done operation, and report the task +
   the specific unsatisfiable criterion to the user, noting it as a
   candidate for a future `skip=` argument. Do not move to the next task
   while this one is incomplete.

## Step 5 — Commit, Push, Open the PR

```bash
git add -A
git commit -m "feat: implement task <task-id> - <short description>"
git push -u origin task/<task-id>-<task-slug>
```

Call the selected VCS adapter's **create PR** operation (Appendix A, `cli`
or `api` mode per Step 0), with:
- title: `Task <task-id>: <short description>`
- body/description: the plan from Step 3, or a one-line summary for
  trivial tasks
- source branch: `task/<task-id>-<task-slug>`, target: `<default-branch>`

Capture the returned PR/MR identifier as `<pr-ref>` (its number/IID — not
its URL; Appendix A's merge operation needs the identifier).

No reviewer is added to the PR/MR — Step 6 below is the actual gate.

## Step 6 — Automated Review and Rework

1. Run the `/code-review` skill against the diff on
   `task/<task-id>-<task-slug>` (the PR's source branch). This step is
   identical regardless of adapter choice — it operates on the local git
   diff, not the hosted PR.
2. If `/code-review` reports no blocking findings: proceed to Step 7.
3. If it reports findings: fix them, commit, `git push` (same branch — the
   open PR/MR updates automatically on every VCS adapter), and re-run
   `/code-review`. This is one "rework round."
4. Track rework rounds for this task. After 3 rework rounds without a
   clean review: **stop the entire loop** (not just this task). Report the
   task, the PR/MR, and the latest `/code-review` findings to the user. Do
   not proceed to Step 7 and do not move to another task.

## Step 7 — Merge

Call the selected VCS adapter's **merge PR** operation (Appendix A) with
`<pr-ref>` from Step 5. A nonzero exit code, a "not mergeable"/conflict
response, or (for adapters where merge is queued rather than synchronous)
a polled failure status all mean the same thing here: stop the loop and
report to the user — do not retry merge indefinitely.

Then, regardless of adapter:
```bash
git checkout <default-branch>
git pull origin <default-branch>
git branch -d task/<task-id>-<task-slug>
```
(The remote branch is already gone — the merge operation above handled
deleting it on every adapter. This is local cleanup only.)

## Step 8 — Repeat

Go back to **Step 1 — Find the Next Task**. Continue the cycle until one of
the Stop Conditions below is hit.

## Stop Conditions

- **Success:** Step 1 finds no remaining undone, unskipped task from the
  selected task source. Report completion to the user with a summary of
  every task processed this run and every task skipped via `skip=`.
- **Unresolvable adapter:** Step 0 can't match the VCS remote to a
  supported provider, the matching CLI/API credential is unavailable, or a
  required task-source environment variable is missing. Halt before Step 1
  and tell the user exactly what's missing.
- **Stuck task:** a task exceeds 3 rework rounds (Step 6) or its merge
  fails (Step 7). Halt the entire loop immediately — do not start another
  task. Report the stuck task, its PR/MR, and the relevant findings/error
  to the user.
- **Unsatisfiable acceptance criterion:** Step 4 finds a criterion the loop
  cannot resolve autonomously. Halt the entire loop immediately. Report
  the task and the specific criterion to the user, and suggest re-running
  with `skip=<task-id>` if the user wants the loop to continue past it.

---

## Appendix A — VCS Adapters

### Detection

| Host pattern in `origin`'s remote URL | `<vcs-provider>` |
|---|---|
| `github.com` | `github` |
| `dev.azure.com`, `ssh.dev.azure.com`, `visualstudio.com`, or an alias resolving to one of these (check `git config --get remote.origin.url` and any `url.<base>.insteadOf` rewrites in `.gitconfig`) | `azuredevops` |
| `gitlab.com` or a self-hosted GitLab (verify with `glab repo view` succeeding, or `GITLAB_HOST`/`CI_SERVER_URL` if set) | `gitlab` |
| anything else | unrecognized — stop the loop |

Parse `<ORG>`/`<PROJECT>`/`<REPO>` (or the provider's equivalent
identifiers) from the remote URL itself — never from a saved CLI default
(`az devops configure -l` in particular can silently point at the wrong
org):
- `azuredevops`: `git@ssh.dev.azure.com:v3/{ORG}/{PROJECT}/{REPO}` (or an
  `insteadOf`-aliased form) — `{ORG}`, `{PROJECT}`, `{REPO}` are exactly
  those path segments, case-sensitive.
- `github`: `{OWNER}/{REPO}` from `github.com/{OWNER}/{REPO}` (or the SSH
  equivalent).
- `gitlab`: `{NAMESPACE}/{REPO}` (may include subgroups,
  `{GROUP}/{SUBGROUP}/{REPO}`).

### `github`

**CLI availability:** `command -v gh` and `gh auth status` both succeed.
**API fallback token:** `GH_TOKEN` or `GITHUB_TOKEN` env var (a PAT with
`repo` scope), used as `Authorization: Bearer $GH_TOKEN`.

| Operation | `cli` mode | `api` mode |
|---|---|---|
| Create PR | `gh pr create --title "..." --body "..." --base <default-branch> --head task/<task-id>-<task-slug>` — capture the printed number, or re-run with `--json number` | `curl -sS -X POST -H "Authorization: Bearer $GH_TOKEN" -H "Accept: application/vnd.github+json" https://api.github.com/repos/{OWNER}/{REPO}/pulls -d '{"title":"...","body":"...","head":"task/<task-id>-<task-slug>","base":"<default-branch>"}'` — `<pr-ref>` is the response's `.number` |
| Merge PR | `gh pr merge <pr-ref> --squash --delete-branch` (use `--merge` instead of `--squash` if `git log --merges -5` shows this repo prefers merge commits) — synchronous; nonzero exit = stop and report | `curl -sS -X PUT -H "Authorization: Bearer $GH_TOKEN" -H "Accept: application/vnd.github+json" https://api.github.com/repos/{OWNER}/{REPO}/pulls/<pr-ref>/merge -d '{"merge_method":"squash"}'`, then `curl -sS -X DELETE -H "Authorization: Bearer $GH_TOKEN" https://api.github.com/repos/{OWNER}/{REPO}/git/refs/heads/task/<task-id>-<task-slug>` to delete the branch — check `.merged == true` in the merge response before treating it as done |

### `azuredevops`

**CLI availability:** `az extension show --name azure-devops` succeeds
(run `az extension add --name azure-devops` if missing), and
`az repos list --organization https://dev.azure.com/<ORG>` succeeds
without a login prompt.
**API fallback token:** `AZURE_DEVOPS_PAT` env var, used as HTTP Basic
auth with an empty username: `-u :$AZURE_DEVOPS_PAT`.

| Operation | `cli` mode | `api` mode |
|---|---|---|
| Create PR | `az repos pr create --organization https://dev.azure.com/<ORG> --project <PROJECT> --repository <REPO> --source-branch task/<task-id>-<task-slug> --target-branch <default-branch> --title "..." --description "..." --output json` — `<pr-ref>` is `.pullRequestId`. `--project` is required even with a configured default. | `curl -sS -u :$AZURE_DEVOPS_PAT -X POST "https://dev.azure.com/<ORG>/<PROJECT>/_apis/git/repositories/<REPO>/pullrequests?api-version=7.1" -H "Content-Type: application/json" -d '{"sourceRefName":"refs/heads/task/<task-id>-<task-slug>","targetRefName":"refs/heads/<default-branch>","title":"...","description":"..."}'` — `<pr-ref>` is the response's `.pullRequestId` |
| Merge PR | `az repos pr update --organization https://dev.azure.com/<ORG> --id <pr-ref> --status completed --delete-source-branch true --merge-commit-message "..."` (note: no `--project` flag on this subcommand) — completion is **queued**, poll with `az repos pr show --organization https://dev.azure.com/<ORG> --id <pr-ref> --query "{status:status, mergeStatus:mergeStatus}"` every 2s up to 5x, wait for `status: completed` + `mergeStatus: succeeded` | `curl -sS -u :$AZURE_DEVOPS_PAT -X PATCH "https://dev.azure.com/<ORG>/<PROJECT>/_apis/git/repositories/<REPO>/pullrequests/<pr-ref>?api-version=7.1" -H "Content-Type: application/json" -d '{"status":"completed","lastMergeSourceCommit":{"commitId":"<head-commit-sha>"},"completionOptions":{"deleteSourceBranch":true,"mergeCommitMessage":"..."}}'` — also queued; poll the same endpoint with GET until `mergeStatus == "succeeded"` |

### `gitlab`

**CLI availability:** `command -v glab` and `glab auth status` both
succeed.
**API fallback token:** `GITLAB_TOKEN` env var, used as header
`PRIVATE-TOKEN: $GITLAB_TOKEN`. Self-hosted instances also need
`GITLAB_HOST` (defaults to `gitlab.com`).

| Operation | `cli` mode | `api` mode |
|---|---|---|
| Create MR | `glab mr create --title "..." --description "..." --source-branch task/<task-id>-<task-slug> --target-branch <default-branch> --yes` — capture the printed IID | `curl -sS -H "PRIVATE-TOKEN: $GITLAB_TOKEN" -X POST "https://${GITLAB_HOST:-gitlab.com}/api/v4/projects/{URL_ENCODED_NAMESPACE_REPO}/merge_requests" -d 'source_branch=task/<task-id>-<task-slug>&target_branch=<default-branch>&title=...&description=...'` — `<pr-ref>` is the response's `.iid` |
| Merge MR | `glab mr merge <pr-ref> --squash --remove-source-branch --yes` — synchronous; nonzero exit or a pipeline-blocked message = stop and report | `curl -sS -H "PRIVATE-TOKEN: $GITLAB_TOKEN" -X PUT "https://${GITLAB_HOST:-gitlab.com}/api/v4/projects/{URL_ENCODED_NAMESPACE_REPO}/merge_requests/<pr-ref>/merge" -d 'squash=true&should_remove_source_branch=true'` — check the response isn't a 405/406 (not mergeable) before treating it as done |

---

## Appendix B — Task Source Adapters

Every adapter exposes the same three operations used by Steps 1 and 4:
**list open tasks** (ordered, excludes done), **get task detail**, and
**mark done**.

### `docs` (default)

No environment variables required. Operates entirely on the local
checkout.

| Operation | Command / procedure |
|---|---|
| List open tasks | `find docs/planning -path '*/sprint-*/task-*.md' \| sort -t/ -k2 -k3 -V`, then for each file `grep -c '^- \[ \]' <file>` — a nonzero count means open. `<task-id>`/`<task-slug>` come from the filename `task-<task-id>-<task-slug>.md`. Order is filename sort order (phase, then sprint, then task, numeric). |
| Get task detail | `Read <task-doc-path>` in full — Goal, Context/Why, Implementation Plan, acceptance-criteria checklist. |
| Mark done | Edit the file: flip each satisfied `- [ ]` to `- [x]` individually, as each criterion is satisfied (not all at once at the end). This is the only place "done" is recorded for this adapter — there is no separate status field. |

### `jira`

**Required env vars:** `JIRA_BASE_URL` (e.g. `https://yourorg.atlassian.net`),
`JIRA_EMAIL`, `JIRA_API_TOKEN`, `JIRA_PROJECT_KEY`. Optional:
`JIRA_JQL` to override the default open-tasks query.
Auth: HTTP Basic with `-u "$JIRA_EMAIL:$JIRA_API_TOKEN"`.

| Operation | Command / procedure |
|---|---|
| List open tasks | `curl -sS -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -G "$JIRA_BASE_URL/rest/api/3/search" --data-urlencode "jql=${JIRA_JQL:-project=$JIRA_PROJECT_KEY AND statusCategory != Done ORDER BY Rank ASC}" --data-urlencode "fields=summary,status,description"` — order is the JQL's `ORDER BY` (Rank ascending by default, i.e. the team's prioritized backlog order). `<task-id>` = the issue key (e.g. `PROJ-123`); `<task-slug>` = a slugified `summary`. |
| Get task detail | `curl -sS -u "$JIRA_EMAIL:$JIRA_API_TOKEN" "$JIRA_BASE_URL/rest/api/3/issue/<task-id>"` — treat `.fields.description` (Atlassian Document Format) as the goal/acceptance-criteria text; render its `content` blocks to plain text/markdown before reading. |
| Mark done | Look up the transition ID for "Done" (or the project's terminal status) via `curl -sS -u "$JIRA_EMAIL:$JIRA_API_TOKEN" "$JIRA_BASE_URL/rest/api/3/issue/<task-id>/transitions"`, then `curl -sS -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -X POST "$JIRA_BASE_URL/rest/api/3/issue/<task-id>/transitions" -H "Content-Type: application/json" -d '{"transition":{"id":"<transition-id>"}}'`. One transition per task, at the end of Step 4 — Jira issues have no native per-criterion checkbox to flip incrementally. |

### `asana`

**Required env vars:** `ASANA_ACCESS_TOKEN` (Personal Access Token),
`ASANA_PROJECT_GID`.
Auth: `Authorization: Bearer $ASANA_ACCESS_TOKEN`.

| Operation | Command / procedure |
|---|---|
| List open tasks | `curl -sS -H "Authorization: Bearer $ASANA_ACCESS_TOKEN" "https://app.asana.com/api/1.0/projects/$ASANA_PROJECT_GID/tasks?completed_since=now&opt_fields=name,gid,notes"` — the API returns incomplete tasks in the project's manual/board order already. `<task-id>` = the task `gid`; `<task-slug>` = a slugified `name`. |
| Get task detail | `curl -sS -H "Authorization: Bearer $ASANA_ACCESS_TOKEN" "https://app.asana.com/api/1.0/tasks/<task-id>?opt_fields=name,notes,subtasks"` — treat `.data.notes` as the goal/acceptance-criteria text; if `.data.subtasks` is non-empty, each subtask is one acceptance criterion — fetch and read each. |
| Mark done | `curl -sS -H "Authorization: Bearer $ASANA_ACCESS_TOKEN" -X PUT "https://app.asana.com/api/1.0/tasks/<task-id>" -d 'data[completed]=true'`. If the task has subtasks, mark each satisfied subtask complete individually as it's finished (`PUT .../tasks/<subtask-gid>` the same way) before completing the parent at the end of Step 4. |

### `monday`

**Required env vars:** `MONDAY_API_TOKEN`, `MONDAY_BOARD_ID`,
`MONDAY_STATUS_COLUMN_ID` (the board's status-column ID, found via the
`columns` field on a board query). Monday's API is GraphQL over a single
endpoint (`https://api.monday.com/v2`), not REST — every call below is a
POST with a `query`/`mutation` string in the JSON body.
Auth: `Authorization: $MONDAY_API_TOKEN` (no `Bearer` prefix).

| Operation | Command / procedure |
|---|---|
| List open tasks | `curl -sS -H "Authorization: $MONDAY_API_TOKEN" -H "Content-Type: application/json" -X POST https://api.monday.com/v2 -d '{"query":"query { boards(ids: [$MONDAY_BOARD_ID]) { items_page { items { id name column_values(ids: [\"$MONDAY_STATUS_COLUMN_ID\"]) { text } } } } }"}'` — filter out items whose status column `text` is the board's "Done" label; remaining order is the array order returned (board item order). `<task-id>` = item `id`; `<task-slug>` = a slugified `name`. |
| Get task detail | Same query shape scoped to one item ID, requesting `name` plus the board's description/long-text column (its column ID is board-specific — resolve once via a `columns { id title }` query and treat whichever column holds free text as the detail field). |
| Mark done | `curl -sS -H "Authorization: $MONDAY_API_TOKEN" -H "Content-Type: application/json" -X POST https://api.monday.com/v2 -d '{"query":"mutation { change_simple_column_value(board_id: $MONDAY_BOARD_ID, item_id: <task-id>, column_id: \"$MONDAY_STATUS_COLUMN_ID\", value: \"Done\") { id } }"}'` — one mutation per task, at the end of Step 4; Monday status columns have no native per-criterion sub-checklist unless the board uses a checklist column, which is board-specific and out of scope for this default mapping. |

### `linear`

**Required env vars:** `LINEAR_API_KEY`, `LINEAR_TEAM_KEY` (e.g. `ENG`).
Linear's API is GraphQL at `https://api.linear.app/graphql`.
Auth: `Authorization: $LINEAR_API_KEY` (no `Bearer` prefix).

| Operation | Command / procedure |
|---|---|
| List open tasks | `curl -sS -H "Authorization: $LINEAR_API_KEY" -H "Content-Type: application/json" -X POST https://api.linear.app/graphql -d '{"query":"query { team(id: \"$LINEAR_TEAM_KEY\") { issues(filter: { state: { type: { nin: [\"completed\",\"canceled\"] } } }, orderBy: priority) { nodes { identifier title description } } } }"}'` — order is priority-then-team-default (Linear's `orderBy: priority`, ascending = most urgent first). `<task-id>` = `identifier` (e.g. `ENG-42`); `<task-slug>` = a slugified `title`. |
| Get task detail | Same query scoped to one issue (`issue(id: "<task-id>") { title description comments { nodes { body } } }`) — treat `description` as the goal/acceptance-criteria text; if the description references a checklist, Linear renders markdown checkboxes inside `description` itself (`- [ ]`) — these can be read the same way as the `docs` adapter's checkboxes for judging what's satisfied, but are not individually flippable via this adapter; only the whole-issue state transition below is written back. |
| Mark done | `curl -sS -H "Authorization: $LINEAR_API_KEY" -H "Content-Type: application/json" -X POST https://api.linear.app/graphql -d '{"query":"mutation { issueUpdate(id: \"<task-id>\", input: { stateId: \"<done-state-id>\" }) { success } }"}'` — resolve `<done-state-id>` once per run via `query { team(id: \"$LINEAR_TEAM_KEY\") { states { nodes { id name type } } } }`, picking the node with `type: \"completed\"`. One mutation per task, at the end of Step 4. |
