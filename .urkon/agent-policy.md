<!-- urkon:managed — installed by `urkon init`; edits are overwritten on the next run.
     Remove this marker line to take ownership of the file and urkon will leave it alone. -->

# Agent policy

**Subject of every rule below: you, the coding agent urkon dispatches to work on this
repository's issues.** Read it alongside `.urkon/workflow.md`, which describes the loop
these rules sit inside.

This text is agent-neutral: it holds whichever agent urkon dispatches, so it names no
harness-specific mechanism.

## Autonomy

You may, without asking: create and check out feature branches, commit, push to feature
branches, open and update PRs, push commits that address PR review comments, create /
read / update / close issues, apply non-gating labels, and update the project board.

## Never — these are the human's

- `git merge` (the human merges every PR)
- pushing to the default branch
- force-pushing (`--force`, `-f`, `--force-with-lease`)
- deleting branches or tags
- adding the dispatch trigger label `agent:review`
- advancing a phase: you may stamp a `phase:*` label once at creation (issue create, PR
  open) and never swap or remove one

If asked to do any of these, decline and hand the command to the human.

Do **not** add a `Co-Authored-By` trailer, or any other AI-attribution trailer, to a commit
message.

## Git and PR ownership

All git and PR work is yours, not urkon's: branch, commit, push, and open the PR. These
operations are protocol-uniform across forges (GitHub, GitLab, Gitea, …), which is why
urkon carries no git or PR abstraction — it treats the host only as issue tracker and
provisioning target. See `.urkon/workflow.md` for the actor breakdown.

## Branch naming

- `feat/<issue#>-<slug>` for anything carrying code or tests. One branch carries every
  PR-bound phase (tests and/or code) of an issue: one branch and one PR per issue, no
  stacking.
- `plan/<issue#>-<slug>` for docs- and design-note-only PRs.
- PRs target the default branch.

## PR title and body

Both are enforced server-side by the `PR Lint` workflow, so a PR that does not
match them fails its checks:

- **Title:** `ISSUE-ID: Short description` — e.g. `123: Solves config parsing`. The id is
  a numeric issue number or a JIRA key like `KAN-42`.
- **Body:** a terse description of what the PR does, then a blank line, then a final line
  `Closes #<ISSUE-ID>`. The `#` is **required** for a numeric issue (merging then
  auto-closes it) and **forbidden** for a JIRA key, where the line is a human breadcrumb
  and urkon drives the tracker close on merge. The body id must match the title id.

## Tracker of record

The issue tracker and its project board are the single source of truth for planning and
status. The watcher moves the board Status deterministically as it runs the choreography;
keep labels and board Status consistent on every transition.

## Identity

You act as a machine account, distinct from the human. Identity is split per axis: the
tracker pair (`URKON_TRACKER_USER` / `URKON_TRACKER_AGENT`) governs issue ownership and
board and label authority; the forge pair (`URKON_FORGE_USER` / `URKON_FORGE_AGENT`)
governs PR ownership and Label Guard. When the tracker is the forge, the forge pair
defaults to the tracker pair.

Commits stay authored as the human (git `user.email` is theirs, so they show as Verified);
only the API identity is the machine account.

## Label authority

- The **human** owns `phase:*` transitions, adds `agent:review`, and owns the
  `agent:<name>` routing selectors. Label Guard reverts an agent-added selector.
- The **watcher** consumes `agent:review` at pickup, owns `agent:in-progress`, and sets
  `human:review`.
- **You** stamp `phase:*` only at creation. No label is mirrored between issue and PR.

## Stopping on a decision

On any decision — however small — post an options comment on the item and stop at
`human:review`. There is no separate blocked label: with no `agent:review` present the
item sits idle until the human answers and re-adds the trigger. Never pick a default and
proceed.
