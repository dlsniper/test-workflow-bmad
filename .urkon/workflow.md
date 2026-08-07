<!-- urkon:managed — installed by `urkon init`; edits are overwritten on the next run.
     Remove this marker line to take ownership of the file and urkon will leave it alone. -->

# Urkon workflow (Git-driven development method)

This is the loop urkon drives in this repository, between the human owner and the coding
agent urkon dispatches. The agent's own rules of conduct are in `.urkon/agent-policy.md`;
this file describes the loop those rules sit inside.

The issue tracker and its project board are the single source of truth for planning and
status.

The model has three loops and three label families: `phase:*` (phase marker + prompt
selector), `agent:*` (agent state), and `human:*` (human state).

GitHub is the default tracker **and** forge; JIRA is an alternate tracker paired with a
GitHub forge (see [JIRA tracker](#jira-tracker-urkon_trackerjira)). Identity is split per
axis: `URKON_TRACKER_USER` (tracker board/label authority) / `URKON_TRACKER_AGENT` (the bot
— issue ownership assignee) and `URKON_FORGE_USER` (Label Guard, merges PRs, and a PR
ownership assignee — the agent opens PRs under it) / `URKON_FORGE_AGENT` (the bot — PR
ownership assignee). When
`tracker=github` the forge pair defaults to the tracker pair (same GitHub logins) unless
set explicitly; under `tracker=jira` the forge pair (GitHub logins) must be set explicitly
and the tracker pair are JIRA `accountId`s.

All repo-specific values (repo URL, identities, board number) live in
`.urkon/settings.toml` (committable) and the repo Actions variables — this doc uses the
config names (`URKON_TRACKER_USER`/`URKON_TRACKER_AGENT`, `URKON_FORGE_USER`/`URKON_FORGE_AGENT`,
`URKON_REPO`) rather than hardcoded logins.

## Three loops, three workflows

There are three **loops** — `spec`, `tests`, `code` — each a user⇄agent iteration.
The **spec** loop runs at the **issue** boundary; the **tests** and **code** loops run
at the **PR** boundary (one PR per issue). The user composes them into one of three
**workflows** by choosing the next phase manually at each hand-off:

- `spec → tests → code`
- `spec → code → tests`
- `spec → code`

There is no upfront declaration of the path; it emerges from which `phase:*` label the
user sets next.

## Roles & gates

- **User (`URKON_TRACKER_USER` on the tracker / `URKON_FORGE_USER` on the forge)** owns
  phase transitions (`phase:*`) and the `agent:review` trigger, and **merges PRs**. The agent
  never merges and never advances a phase on its own.
- **Agent (`URKON_TRACKER_AGENT` on the tracker / `URKON_FORGE_AGENT` on the
  forge)** plans, writes specs/tests/code, opens & updates the
  PR, and reviews it. It stamps `phase:*` **only at creation** (issue create, PR open).
  It does **not** move the Projects board — that is the watcher's (below).
- The **Watcher** (`urkon watch`) is deterministic: it polls, runs the label
  choreography, **moves the Projects board Status through the tracker provider at each
  step**, dispatches the agent headless, and streams the trace. It does **no git and no PR
  ops** — those are the agent's.

## Labels (state machine)

Three families, plus nothing else that this workflow reads. Unknown labels are always
preserved.

| Label               | Who sets                     | Meaning / next action |
|---------------------|------------------------------|-----------------------|
| `phase:spec`        | User; agent at issue creation | Spec loop active on the issue. Also the prompt selector. |
| `phase:tests`       | User; agent at PR open        | Tests loop active on the PR. Prompt selector. |
| `phase:code`        | User; agent at PR open        | Code loop active on the PR. Prompt selector. |
| `agent:review`      | User only                   | **The review-loop trigger.** "Agent, your turn." Consumed by the watcher at pickup. |
| `agent:debug`       | User (issue or PR)          | **The debugging trigger** (independent of `agent:review`). Runs the debugging workflow; consumed by the watcher at pickup. See [Debugging workflow](#debugging-workflow-agentdebug). |
| `agent:in-progress` | Watcher (agent)              | A headless run is actively working this item now. Added at pickup, removed at cleanup. |
| `agent:errored`     | Watcher (agent)              | A headless run hit an error. Durable auto-resume marker; kept on hand-back. See [Auto-resume](#auto-resume-on-transient-failure). |
| `agent:retrying`    | Watcher (agent)              | Auto-resume in progress (waiting to retry a failed run). Held across attempts, removed on recovery or hand-back. See [Auto-resume](#auto-resume-on-transient-failure). |
| `agent:<name>`      | User only                   | **Agent selector** (e.g. `agent:claude`, `agent:amp`). Routes the item to that agent; absent ⇒ `URKON_AGENT_CLI_NAME`. Persists across phases (never stripped). One per registered agent, created by `urkon init`. Label Guard reverts an agent-added selector. |
| `human:review`      | Agent; user may also set/remove | The agent finished a turn **or** is blocked on a user decision (see below). Best-effort removed by the watcher at pickup. |

`phase:*` is both the phase marker and the prompt selector; it is **never** a dedup key.
Dedup rides entirely on the trigger being consumed at pickup — the watcher fires only when a
trigger (`agent:review` or `agent:debug`) is present and the item is not already
`agent:in-progress`. When both triggers are present, `agent:debug` wins and the debug turn
consumes **both** triggers, so the review loop does not auto-fire afterward (the user re-adds
a trigger to continue).

**Blocked on a decision.** The no-assumptions hard rule holds in headless runs: the agent
never guesses. When it hits any decision it (1) posts a comment listing concrete options,
(2) sets `human:review`, (3) stops. There is **no separate blocked label** — a blocked
item is just `human:review` with no `agent:review`, so it sits idle until the user
answers and re-adds `agent:review`. The user disambiguates "done" from "blocked" by
reading the comment.

## Uniform watcher choreography

Every dispatch — regardless of phase — follows the same steps:

1. **Trigger.** Item (issue or PR) has a trigger (`agent:review` or `agent:debug`), is not
   `agent:in-progress`, and the in-flight guard is free.
2. **Pickup.** Consume the trigger (abort if it can't be removed), best-effort remove
   `human:review`, add `agent:in-progress`, then move the board Status to the phase's
   **working** state (`phase:spec`→Spec, `phase:tests`→Tests, `phase:code`→Code; a debug run
   makes no board move).
3. **Run.** Dispatch the selected prompt — the item's `phase:*` label picks the spec,
   tests, or code prompt; `agent:debug` picks the debug prompt (see
   [Debugging workflow](#debugging-workflow-agentdebug)).
4. **Cleanup.** Remove `agent:in-progress` (cancellation-immune, pass or fail), add
   `human:review`, then move the board Status to the phase's **review** state
   (`phase:tests`→Tests review, `phase:code`→Code review; Spec has no separate review
   column, so it stays Spec; a debug run makes no board move).
5. **Merge trigger.** A merged PR that still carries workflow labels (and not `bookkept`)
   dispatches the bookkeeping prompt; its cleanup adds the permanent, human-visible
   `bookkept` marker, **removes all workflow labels**, and moves the board Status to
   **Done**. GitHub's search index lags label writes, so the watcher also keeps an
   in-process at-most-once set of bookkept PRs — the durable dedup is the `bookkept`
   label, the in-process set closes the search-lag window so bookkeeping never re-runs.

All board moves go through the tracker provider; a provider with no board
configured returns `ErrUnsupported` and the step is a clean no-op, so the loop still runs
without a board.

Only one urkon instance runs per repo, and an in-process per-item in-flight guard means
the same issue/PR is never worked twice at once. Distinct items run independently, and
each dispatch executes in its **own git worktree** rather than a single shared tree — see
[Sandboxed runs & worktree isolation](#sandboxed-runs--worktree-isolation-urkon_sandbox).

Each loop keeps its own agent session keyed by the **feature id** (the issue number `N`):
the spec loop resumes `spec/N`, and the implementation loops (tests + code) share one
feature-stable `code/N` session that spans the issue→PR boundary. The **first** run of a
session is *cold* — the full initial prompt; every later run *resumes* it with a terse
**continuation** prompt. Bookkeeping resumes the feature's last session (`code/N`, else
`spec/N`). A resume id the agent rejects as expired falls back to a cold run. See
the resume behaviour described above.

Sessions key by **feature id** while worktrees fan out by **`(feature, agent)`** (see
[Sandboxed runs](#sandboxed-runs--worktree-isolation-urkon_sandbox)): switching the
`agent:<name>` selector mid-issue creates a second worktree while the session store still
keys on the feature — this interaction is intended.

## Sandboxed runs & worktree isolation (`[sandbox]`)

Every headless dispatch runs in its **own git worktree**, and optionally inside a
container backend — configured under the `[sandbox]` table of `.urkon/settings.toml`.

**Worktree per `(feature, agent)` (always on).** One worktree lives under
`.worktrees/<feature>-<agent>/` (e.g. `.worktrees/5-claude/`), created on the first
dispatch for that pair (detached at the remote's default branch, fetched first) and
removed at post-merge bookkeeping. It replaces the old single shared working tree, so
distinct features — and distinct agents on the same feature — never contend on one tree.
Urkon's own state (`.urkon/`) stays at the base dir on the host; only the agent's **cwd**
is the worktree.

**Backends.** `[sandbox].backend` selects the run-isolation runtime:

| backend | behaviour |
|---------|-----------|
| `none` (default) | Host passthrough — runs on the host exactly as before, only pinning the run's cwd to its worktree. |
| `docker` / `podman` / `rancher` / `openshell` | Container runtimes. **Skeletons for now:** selecting one emits a one-time warning and falls back to host passthrough, unless `require_backend = true`, which **fails `urkon watch` closed at startup**. |

**Env & secrets.** A container backend takes **zero host env**; its environment comes
solely from a per-sandbox **`.env.sandbox`**, generated by urkon under the worktree (already
git-ignored) from the committed `[sandbox.env]` template. Each template value's
`${HOST_VAR}` references resolve from the real host environment at generation time, so
secrets (e.g. `GH_TOKEN`) are never committed. Inspect the fully-resolved result with
`urkon sandbox secrets --feature N --agent claude` (`--mask` redacts values). GPG key and
git config reach a container via **read-only path mounts** (`[sandbox].paths_ro`, default
`~/.gnupg` + `~/.gitconfig`), not env, so signed, Verified commits still work inside.

The host-passthrough default (`none`) ignores the rendered env and mounts — it keeps
today's inherited environment — so the only observable change under `none` is that each
dispatch uses its own `(feature, agent)` worktree.

## Auto-resume on transient failure

A headless run can fail for a transient reason — a usage/rate limit, a network timeout,
`error_max_turns`/`error_max_budget_usd`, a render error. Instead of handing straight back
to the user, the watcher **waits and resumes the same session in-process**, bounded so it
can never wait or retry forever. Runs that recover are invisible to the user; runs that
exhaust the bounds hand back with a clear signal. Auto-resume is **always on** (the bounds
are the only control) and applies to every triggered phase/review/debug run. Bookkeeping is
excluded — its labels persist on failure, so it already re-fires on the next poll.

**Retry decision (opt-out).** Any non-stale run that returns an error is retryable *unless*
the agent adapter marks it fatal — reserved for a known-fatal config/binary
problem retrying can never fix (e.g. the agent CLI is not on `PATH`, or a `401`/`403` auth
error). A stale resume id still takes precedence and re-runs cold, unchanged.

**Agent-global limit backoff.** A rate/usage limit (`429`/`529`) is a per-account condition:
it affects *every* session sharing that agent's account, not just the one item. So when a run
reports a usage limit, the watcher records an agent-level deadline and **every**
run for that agent — new dispatches and rehydrations alike — waits until it passes before
starting. One item's limit thus backs off all of that agent's work together, instead of each
item independently re-hitting the limit. The per-item bounds still apply on top; the gate only
synchronizes *when* the agent's runs happen.

**Wait math.** Before each attempt the watcher waits `LimitReset − now` when the provider
reported a best-effort reset time, else exponential backoff `min(base·2^(n−1), cap)`; a
**buffer** is added either way so the estimate and the LLM backend have slack. The reset time
is not dependably emitted in headless mode, so backoff is the common path.

**Bounds.** The **cumulative-wait ceiling** (`URKON_RESUME_MAX_WAIT`, default 6h) is the
primary bound: before each sleep, if `cumulative + next_wait` would exceed it, the watcher
hands back. A high **attempt backstop** (`URKON_RESUME_MAX_ATTEMPTS`, default 50) guards a
runaway loop. All knobs live in the `[resume]` table of `.urkon/settings.toml` with
`URKON_RESUME_*` env overrides (`buffer` 1m, `backoff_base` 30s, `backoff_max` 30m).

**Labels.** On the first retryable failure the watcher adds both `agent:errored` and
`agent:retrying` and begins waiting; `agent:retrying` persists across attempts. On **clean
recovery** it removes both (a green run leaves no trace) and runs normal cleanup. On
**hand-back** it removes `agent:retrying`, keeps `agent:errored`, adds `human:review`, and
posts a terse comment quoting the failure and why it gave up. Both labels are agent-managed
reserved `agent:*` state (never parsed as `agent:<name>` selectors) and are stripped from a
merged PR by post-merge bookkeeping.

**Cross-restart persistence.** A retry episode is durably recorded in `.urkon/runs/resume.json`
(keyed `<kind>-<number>`, atomic-write like `sessions.json`) holding episode metadata only —
kind, number, phase (the loop is derived from the phase slug, never a stored enum int), agent,
feature id, attempt count, cumulative wait, and the absolute next-resume deadline. The session id
is **not** duplicated; it stays source-of-truth in `sessions.json`. The record is written when the
first wait begins, updated each attempt, and deleted on recovery or hand-back. `SIGTERM`/ctx-cancel
mid-wait aborts the in-memory run but leaves the record and labels in place (unless the record
could not be persisted, in which case the item is handed back rather than stranded). **At the start
of each poll pass** the watcher rehydrates each pending record: it re-fetches and re-validates the
item (still present, still `agent:in-progress` + `agent:retrying`; a completed run, a stripped
merged PR, or a user who cleared the labels is discarded; a record whose phase/agent no longer
resolves is handed back rather than left stalled), reconstructs the task without re-running pickup,
and re-enters the resume loop at the stored attempt/cumulative — sleeping until the deadline, or
resuming immediately if it already passed during downtime. Rehydrating each pass (not only at
startup) also lets a record whose re-fetch failed transiently retry on a later pass; a record whose
episode is already running holds the in-flight lock and is skipped, so there is no double pickup.

## Multiple instances on one repo (assignee ownership)

Several watchers can run against **one** repo, each with its own tracker/forge identities,
without stealing each other's work. Ownership is expressed through the **assignee** field on
each axis (GitHub Assignees; JIRA's single-slot assignee). Ownership keys off the axis that
owns the item: an **issue** is owned iff `URKON_TRACKER_AGENT` (the bot) is its **sole**
assignee; a **PR** is owned iff `URKON_FORGE_AGENT` **or** `URKON_FORGE_USER` is its sole
assignee (comparison is case-insensitive). The bundled agent opens PRs under `URKON_FORGE_USER`
(`--assignee @me`), so the human forge login is a valid PR ownership key alongside the bot;
this is why **each instance must have its own distinct `URKON_FORGE_USER`** (separate machines,
separate logins) — a shared `URKON_FORGE_USER` would collapse the PR gate. On the issue axis,
assign the **bot** (`URKON_TRACKER_AGENT`) to route an item to an instance; the human
`URKON_*_USER` logins additionally keep board/label/merge authority. Issue-ownership gating is
**always on** when `URKON_TRACKER_AGENT` is set, PR-ownership
gating when `URKON_FORGE_AGENT` is set — a single instance simply owns everything it claims.

**Ordering (critical).** Ownership is resolved **before** the consume-at-pickup step, so a
non-owning instance never consumes the trigger (which would starve the winner). Per candidate:
resolve/claim ownership → only if owned → pickup (consume trigger) → work.

- **Issues** (keyed on `URKON_TRACKER_AGENT`). Sole-owner → own. Unassigned → **auto-claim**:
  assign `URKON_TRACKER_AGENT`, wait `URKON_CLAIM_SETTLE` (default 10s), re-fetch, and proceed
  only if it took. Anything else (co-assignee, or another bot) → **skip silently** (trace
  only; trigger + assignees left intact). Auto-claim applies to every trigger family
  (`agent:review`, `agent:debug`, bmad anytime-triggers).
- **Race resolution.** If the post-settle re-fetch shows another bot also assigned, the
  **lowest login wins** (case-insensitive); losers remove their own assignment (retried a
  bounded number of times) and skip. The settle window widens convergence but this remains
  **best-effort mutual exclusion**, not an atomic lock: if a loser's de-assign fails
  permanently the item can stay multi-assigned (both instances then skip it) until a human
  clears the assignees.
- **PRs** (keyed on `URKON_FORGE_AGENT` or `URKON_FORGE_USER`). Assigned → owned iff sole-
  assignee is either login (the agent self-assigns via `--assignee @me` = `URKON_FORGE_USER`).
  Unassigned → resolve the linked issue (`Closes #N`, or `Closes KAN-42` under JIRA — via the PR
  branch/title key); owned iff that **issue** is sole-`URKON_TRACKER_AGENT` (the linked-issue
  check is a tracker-axis concern), and the watcher **self-heals** by assigning the PR to
  `URKON_FORGE_AGENT`. Unassigned with no linked issue → skip (re-evaluated next poll).
  Post-merge bookkeeping is gated by the same PR rule, so only the owning instance strips a
  merged PR's labels.

**Assignability guard.** GitHub's `addAssignees` silently drops a non-assignable user, so
`watch` validates `URKON_FORGE_AGENT` is assignable on the forge at startup (hard-fail), and a
claim that does not take after the settle re-fetch is surfaced and skipped. Under a JIRA
tracker `watch` also verifies acli's ambient auth targets the configured site/project as the
tracker agent (`VerifyAccess`). Auto-resume is unaffected — it runs on local per-instance
state, never a cross-instance tracker query.

## The loops

### Spec loop (issue is the live item)

1. The user or the agent creates an issue tagged `phase:spec` (the agent stamps it at creation).
   The issue body captures the spec directly.
2. When the issue is ready for the agent, the user sets `agent:review`.
3. The watcher picks it up (consume `agent:review`, drop `human:review`, add
   `agent:in-progress`) and runs the spec prompt. The agent refines the spec on the issue and
   captures its current state as a single **`## Full specification`** comment — the
   finalized spec the implementation loops build from.
4. On finish, cleanup sets `human:review`. User reviews.
5. **Iterate:** user re-adds `agent:review` → back to step 3.
6. **Advance:** user removes `phase:spec` and sets `phase:tests` **or** `phase:code`
   (plus `agent:review`) — this both starts the next loop and picks the workflow variant.

### Tests loop and Code loop (PR is the live item)

The two PR-bound loops are identical in choreography; they differ only by which prompt
runs and what the agent produces.

1. **First non-spec run.** Triggered by `agent:review` on the issue carrying `phase:tests`
   (or `phase:code`). The watcher reads the issue's **`## Full specification`** comment and
   **injects it by value** into the cold prompt, so the agent works **from that spec** without
   re-reading the whole issue thread; this is the spec→next-phase hand-off. If no
   `## Full specification` comment exists yet (spec phase skipped), the prompt's guard body
   has the agent write one from the issue first. When the work is green the agent
   **opens the PR itself** (`Closes #<issue>` in the body, or `Closes KAN-42` under JIRA) — this
   is the agent's git/PR job, not
   the watcher's — and stamps the new PR with the matching `phase:*` **and** `human:review`. The
   agent must set `human:review` on the PR itself because the watcher's cleanup labels only the
   item it picked up (the issue), not the PR the agent created out-of-band. From here the **PR is
   the live item**; the issue waits to be closed on merge — GitHub auto-closes a linked GitHub
   issue, but under JIRA GitHub cannot auto-close the issue (the `Closes KAN-42` line is a human
   breadcrumb), so urkon itself drives the JIRA issue to Done + closes it on merge. There is
   **no** issue↔PR label sync.
2. **Review.** User reviews the PR (code comments and/or PR comments) and sets
   `agent:review` on the PR.
3. **Iterate.** Watcher picks up the PR, the agent addresses the feedback, pushes to the same
   branch, replies per thread, and finishes with `human:review`. Back to step 2.
4. **Advance (tests → code).** User swaps the `phase:*` label on the PR (remove
   `phase:tests`, add `phase:code`) plus `agent:review`. Same PR, same branch — **one PR
   per issue**. The `spec → code` and `spec → code → tests` variants are just different
   orders of setting these labels.
5. **Merge.** When satisfied, the user **merges** the PR. The merge trigger dispatches the
   bookkeeping prompt, which strips the workflow labels and stamps `bookkept`. On a GitHub
   tracker the `Closes #<issue>` reference auto-closes the issue; under JIRA GitHub cannot,
   so urkon drives the linked JIRA issue to the mapped Done status and closes it via acli.

**Code review is this loop, not a separate one.** There is no review-only label or prompt:
the user's PR review is addressed by the same `agent:review` dispatch (steps 2–3 above). The
tests and code prompts instruct the agent to address the user's PR review feedback per thread on
each PR turn.

## Debugging workflow (`agent:debug`)

An **independent** trigger, separate from the three loops: `agent:debug` on an **issue or a
PR** runs a debugging pass — reproduce, root-cause, and post a terse findings comment. It
needs no `phase:*`; the user just files (or reuses) an item, labels it `agent:debug`, and the
watcher dispatches the debug prompt. It is polled by its own queries and consumed at pickup
exactly like `agent:review` (its removal is the dedup); when both triggers are present,
`agent:debug` wins.

- **Deliverable.** After reporting, the agent delivers the fix as a PR that then follows the
  regular loop. On an **issue** it opens a fix PR (`feat/<key>-<slug>`, `Closes #<issue>` — or
  `Closes KAN-42` under JIRA)
  stamped `phase:code` + `human:review`, so the PR enters the code loop for user review. On a
  **PR** it pushes the fix to that PR's existing branch and replies per thread.
- **No board move.** Debugging has no board column, so a debug run leaves the Projects board
  Status untouched; any fix PR it opens enters the board through the normal code loop.
- **Cleanup.** Like a review turn, cleanup drops `agent:in-progress` and sets `human:review`
  (the agent reported and/or opened a fix PR — the user acts next). It also consumes a leftover
  `agent:review` if the user set both, so the debug turn claims the item fully and nothing
  auto-fires the review loop next poll.

## Projects board (`URKON_TRACKER_GITHUB_PROJECT_NUMBER`)

Status field options: **Backlog → Spec → Tests → Tests review → Code → Code review → Done**.

The **watcher** keeps the board in sync through the tracker provider at each
choreography step (pickup, cleanup, merge bookkeeping) — the GitHub provider resolves the
board's Status field + option IDs once, adds the item to the board if absent, then sets the
single-select value. GitHub's built-in Projects automations only cover item-added /
item-closed / PR-merged — they cannot map labels → Status. A board move never blocks the
loop: a failure is logged, and a provider with no board configured no-ops via
`ErrUnsupported`.

State → Projects Status mapping:

| State                                   | Projects Status |
|-----------------------------------------|-----------------|
| (created)                               | Backlog         |
| `phase:spec`                            | Spec            |
| `phase:tests`, agent working / PR opened | Tests           |
| `phase:tests` + `human:review`          | Tests review    |
| `phase:code`, agent working             | Code            |
| `phase:code` + `human:review`           | Code review     |
| (PR merged)                             | Done            |

## External orchestrator (`urkon watch`)

So the user doesn't have to verbally hand off every transition, the **external** watcher
drives the loop. It is the `urkon watch` command, run from this repo's root (polling the tracker every ~2 min) so the repo's own agent instructions, skills, and settings all load, and dispatching the agent headless when a trigger fires. The GitHub **labels are the queue** — the watcher holds no local state.
It reads its two secret tokens from the environment and every other knob from
`.urkon/settings.toml` at the base dir (`--dir`, default cwd), a CLI flag, or a `URKON_*`
env var (precedence: flag > env > settings.toml > built-in default). `Ctrl-C`/`SIGTERM`
cancels any in-flight run and stops the loop.

The watcher runs the selected agent adapter **in-process**: it execs
`claude -p … --permission-mode bypassPermissions` and renders its `--output-format stream-json`
trace itself (the Claude-specific `--permission-mode` flag lives in the adapter, not the loop).
The adapter is selected internally — there is no user-facing subcommand to invoke it.
`URKON_AGENT_CLAUDE_BIN` overrides the executable; `URKON_AGENT_CLAUDE_PERMISSION_MODE` the flag.

Each headless run streams its full turn-by-turn trace to a stable per-PR/issue file under
`.urkon/runs/` (`<kind>-<number>.log`, e.g. `pr-38.log`), appended across runs so `tail -f`
survives re-runs. The watcher's own heartbeat prints to the console and is teed to
`.urkon/watch.log`. (Both paths are relative to `--dir`.) All urkon on-disk artifacts live
under the one `.urkon/` directory, alongside the committable `.urkon/settings.toml`.

## JIRA tracker (`URKON_TRACKER=jira`)

GitHub is the default tracker **and** forge. Setting `URKON_TRACKER=jira` selects a JIRA
issue tracker driven by the **acli** Atlassian CLI (**≥ 1.3.22**) as a subprocess. JIRA
hosts no PRs, so the **only supported pairing is `tracker=jira` + `forge=github`** — the
tracker is JIRA, the forge stays GitHub.

- **Env:** `URKON_TRACKER_JIRA_SITE`, `URKON_TRACKER_JIRA_PROJECT`, `URKON_TRACKER_JIRA_BOARD` (precedence:
  flag > env > `settings.toml [tracker.jira]`). Auth is **acli-ambient** (`acli jira auth login`) —
  urkon never reads, stores, or passes a JIRA token. `URKON_GH_TOKEN` / `URKON_REPO` and the
  explicit `URKON_FORGE_USER` / `URKON_FORGE_AGENT` (GitHub logins, no inference) stay
  required for the GitHub forge; the tracker pair are JIRA `accountId`s. Setting any
  `URKON_TRACKER_JIRA_*` while `URKON_TRACKER != jira` is a hard error.
- **`settings.toml [tracker.jira]`:** `urkon init` writes this block (replacing any prior
  one, other lines/comments preserved). It holds the `[tracker.jira]` `site`/`project`/`board`
  scalars plus `[tracker.jira.status_map]` and `[tracker.jira.bmad_status_map]` subtables
  mapping **each** pipeline stage to a JIRA status **name**. Every stage must be mapped (JIRA
  statuses are user-owned; urkon never authors or backfills them); an unmapped stage is a loud
  validation error. The non-done mapped statuses are the "open" query set.
- **Startup guard:** `watch` verifies acli's ambient auth targets `URKON_TRACKER_JIRA_SITE` as
  `URKON_TRACKER_AGENT` and the project is reachable; drift hard-fails with guidance
  (`acli jira auth switch`).
- **Linkage & close:** the branch/title carry the JIRA key (`feat/KAN-42-slug`,
  `KAN-42: <desc>`); `Closes KAN-42` in the body is a human breadcrumb — GitHub does not
  auto-close a JIRA issue, so urkon drives the JIRA issue to the mapped Done status and closes
  it on merge (see [PR conventions](#pr-conventions)).
- **Label authority** on JIRA issues is convention-only (no GitHub-Actions guard); the forge
  side (PRs) keeps Label Guard + PR Lint (see [Label authority](#label-authority-enforced-by-label-guard)).

## PR conventions

On a forge that runs the installed checks these are enforced server-side by the `PR Lint`
workflow; on one that does not they hold as conventions of the loop. Either way the agent
follows them.

- **Branch prefix:** `feat/<key>-<slug>` — one branch carrying every PR-bound phase
  (tests and/or code) of an issue; no stacking. `<key>` is the feature id: the issue number
  under GitHub (`feat/123-slug`) or the JIRA key under JIRA (`feat/KAN-42-slug`).
  `plan/<key>-<slug>` for docs / design-note PRs (no code/tests). All target `main`.
- **Title:** `ISSUE-ID: Short description` — accepts **both** the numeric form
  (`123: Solves config parsing`) and the JIRA-key form (`KAN-42: Solves config parsing`).
- **Body:** a medium-but-terse description of what the PR does, then two blank lines, then a
  final line — `Closes` accepts **both** numeric and JIRA-key forms:

  ```
  Closes #123
  ```

  or, under JIRA:

  ```
  Closes KAN-42
  ```

  On a GitHub tracker the literal `Closes #123` form both renders as a clickable reference
  **and** auto-closes the issue on merge. Under JIRA `Closes KAN-42` is a **human
  breadcrumb** only — GitHub does not auto-close a JIRA issue, so urkon itself drives the JIRA
  issue to Done + closes it on merge. `PR Lint`, where installed, accepts both forms in the title and body.

## Identities (user vs agent)

Identity is split per axis (see the intro): the **tracker** pair
(`URKON_TRACKER_USER` / `URKON_TRACKER_AGENT`) owns issues and tracker board/label authority,
and the **forge** pair (`URKON_FORGE_USER` / `URKON_FORGE_AGENT`) owns PRs and Label Guard
authority. When `tracker=github` the forge pair defaults to the tracker pair (same GitHub
logins) unless set explicitly; under `tracker=jira` the tracker pair are JIRA `accountId`s and
the forge pair (GitHub logins) must be set explicitly.

On the GitHub **forge**, the agent acts as a dedicated **`URKON_FORGE_AGENT`** machine account (a
repo collaborator with Write), authenticated via its fine-grained PAT — the watcher reads it
from `URKON_GH_TOKEN` in the environment and exports it as `GH_TOKEN` for itself and every headless
child. So the agent's PRs/labels show as **`URKON_FORGE_AGENT`**, clearly distinct from the user's
**`URKON_FORGE_USER`**. On a JIRA **tracker**, acli runs as the tracker agent
(`URKON_TRACKER_AGENT`, acli-ambient auth) and assigns `URKON_TRACKER_AGENT` to claim issue
ownership. **Commits are still authored as the (forge) user** (git `user.email` = user's, and
GPG-signed so they show *Verified*) — only the API identity is the agent. This split is what
lets Label Guard *machine-enforce* the gate (it no longer relies on a shared token).

### Operator setup (one-time)
Export `URKON_USER_TOKEN` (a user PAT) in the environment, then run `./urkon init` (creates the
repo, labels, repo variables, the agent collaborator, and the board + Status options, links
the board to the repo, grants `URKON_TRACKER_AGENT` **Writer** access on the board, and
scaffolds `.urkon/settings.toml` if absent), and complete its printed manual steps:
1. Create the `URKON_FORGE_AGENT` GitHub account (`init` adds it as a **collaborator (Write)**
   once it exists).
2. As `URKON_FORGE_AGENT`, create a **fine-grained PAT** scoped to this repo: Contents, Issues,
   Pull requests, Projects (read/write). Export it as `URKON_GH_TOKEN`.
3. In the board's **Project → Settings**, set **Default repository** to this repo. GitHub
   exposes no API for this setting, so `init` cannot automate it; it backs the board's
   built-in auto-add / draft-issue workflows.
4. Register the user's **GPG signing key** on the user account, and set the repo git config:
   `git config user.name/user.email` (user's), `commit.gpgsign true`, `user.signingkey <id>`.

## Label authority (enforced by `Label Guard`)

With distinct identities, `Label Guard` (GitHub Actions, on the **forge**) enforces a real
matrix on PR labels (reverting violations). The user and agent logins come from the repo
Actions variables `vars.URKON_FORGE_USER` / `vars.URKON_FORGE_AGENT`.

| Labels | Who may change |
|--------|----------------|
| `phase:spec`, `phase:tests`, `phase:code` (phase transitions) | `URKON_FORGE_USER` only, **except** the agent may **add** a `phase:*` label on an item it just created (issue create / PR open) — the initial stamp. All later transitions are user-only. |
| `agent:review` (the trigger) | `URKON_FORGE_USER` adds; `URKON_FORGE_AGENT` may **remove** (consume-at-pickup), never add |
| `agent:in-progress` | `URKON_FORGE_AGENT` only |
| `human:review` | `URKON_FORGE_AGENT` or `URKON_FORGE_USER` |
| anything, by anyone else | reverted |

`URKON_FORGE_USER` and `github-actions[bot]` (the revert step itself) bypass the checks.

**JIRA issue-side caveat.** Label Guard is GitHub Actions and covers only the **forge** (PRs).
JIRA has no GitHub-Actions equivalent, so label authority on **JIRA issues is
convention-enforced only** — there is no server-side guard reverting a violating issue-side
label change (urkon's own code still never advances a `phase:*` transition). The forge side
(PRs) keeps both Label Guard and PR Lint.

## Quality gates

This repository's own build, lint, and test commands are the gates — the agent runs them
locally before pushing, and CI must be green before the owner merges. Urkon does not
define or impose them: the agent discovers this repo's conventions the usual way (its
`AGENTS.md`/`CLAUDE.md`, `Makefile`, or CI workflow definitions) and uses whatever the repo
already prescribes. Do not invent a build system the repo does not have.

## Hard limits

The agent never: `git merge`, push to the default branch, force-push, delete branches or
tags, advance a `phase:*` transition on its own, add `agent:review`, or add a
`Co-Authored-By` (or any AI-attribution) trailer. On any decision — however small — it
posts a comment listing concrete options and stops rather than guessing. See
`.urkon/agent-policy.md` for the full set.
