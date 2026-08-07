<!-- urkon:managed — installed by `urkon init`; edits are overwritten on the next run.
     Remove this marker line to take ownership of the file and urkon will leave it alone. -->

# Urkon workflow — BMAD loop

This repository runs urkon's **BMAD** loop: an eight-phase flow driven by `bmad:*` labels,
rather than the core `spec`/`tests`/`code` loop. There is no `phase:*` label in this
repository — if you are looking for one, you are reading the wrong mode's documentation.

Read this with the two files installed alongside it:

- `.urkon/mechanics.md` — the mode-independent machinery: sandboxing, retries,
  multi-instance ownership, identities, PR conventions, and the hard limits. All of it
  applies here unchanged.
- `.urkon/agent-policy.md` — the rules the dispatched agent follows.

The issue tracker and its project board are the single source of truth for planning and
status.

## The eight phases

Each phase is one user⇄agent iteration, selected by the item's `bmad:*` label, which is also
the prompt selector. The first four are **planning** phases that run on the epic issue; the
last three are **implementation** phases that run per sub-issue.

| Phase label          | Board status   | What the agent produces |
|----------------------|----------------|-------------------------|
| `bmad:brief`         | Brief          | Product brief (`bmad-product-brief`) |
| `bmad:prd`           | PRD            | PRD (`bmad-prd`) |
| `bmad:architecture`  | Architecture   | Architecture doc (`bmad-architecture`) |
| `bmad:epics`         | Epics          | Epics and stories (`bmad-create-epics-and-stories`) |
| `bmad:story`         | Story          | Story artifacts, plus the linked **sub-issues** |
| `bmad:dev`           | Dev            | Implementation, per sub-issue |
| `bmad:qa`            | QA             | Tests and quality work, per sub-issue |
| `bmad:review`        | Review         | Review fixes, per sub-issue |

Artifacts are written under the planning path configured in `.urkon/settings.toml`, not
inline in the issue.

## Anytime triggers

Four labels are **not** phases and do not move the board. They can be applied to an issue or
a PR at any point, and are consumed at pickup like any other trigger:

| Label                   | Runs |
|-------------------------|------|
| `bmad:correct-course`   | `bmad-correct-course` — assess a deviation, propose the adjustment |
| `bmad:retrospective`    | `bmad-retrospective` — capture what worked and what to change |
| `bmad:brainstorm`       | `bmad-brainstorming` — generate and structure options |
| `bmad:setup`            | One-off bootstrap: commit the installed BMAD framework files |

## The planning PR

The epic **issue #N stays the live item** through all four planning phases; `urkon advance N`
drives phase→phase on that issue. One long-lived planning PR carries the planning artifacts:

| Phase | PR behaviour | Reference line | Closes the epic? |
|-------|--------------|----------------|------------------|
| `bmad:brief` | **Opens** the planning PR on `feat/N-<slug>` | `Refs #N` | no |
| `bmad:prd`, `bmad:architecture`, `bmad:epics` | **Finds** the open planning PR (its body references `#N`) and pushes to its branch — never a new PR | `Refs #N` | no |
| after the epics review | the user **merges** the planning PR | — | **no** — the epic stays open |
| `bmad:story` | its **own** new PR; creates and links sub-issues `#M` | `Refs #N` | no |
| `bmad:dev` → `bmad:qa` → `bmad:review` | a per-story code PR, per sub-issue `#M` | `Closes #M` | closes that sub-issue |

Two rules follow from this, and they are the ones most easily got wrong:

- **Planning PRs use `Refs #N`, never `Closes #N`.** Merging one must leave the epic open —
  the next planning phase needs it, and so does `bmad:story`.
- **Only a sub-issue's code PR uses `Closes #M`.** The epic is an umbrella; nothing closes it
  automatically. The user closes it by hand once its sub-issues are merged.

## Labels (state machine)

Beyond the `bmad:*` phase and trigger labels above, the loop reads the same state families as
core mode. Unknown labels are always preserved.

| Label               | Who sets                        | Meaning |
|---------------------|---------------------------------|---------|
| `agent:review`      | User only                       | **The dispatch trigger** — "agent, your turn". Consumed by the watcher at pickup. |
| `agent:debug`       | User (issue or PR)              | The debugging trigger; see `.urkon/mechanics.md`. |
| `agent:in-progress` | Watcher                         | A headless run is working this item now. |
| `agent:errored`     | Watcher                         | A run hit an error; durable auto-resume marker. |
| `agent:retrying`    | Watcher                         | Auto-resume in progress. |
| `agent:<name>`      | User only                       | Agent selector (`agent:claude`, `agent:amp`, …). |
| `human:review`      | Agent; user may also set/remove | The agent finished a turn, **or** is blocked on a decision. |

The `bmad:*` label is the phase marker and prompt selector; it is never a dedup key. Dedup
rides entirely on the trigger being consumed at pickup.

**Blocked on a decision.** The no-assumptions rule holds in headless runs: the agent never
guesses. On any decision — including a BMAD elicitation — it posts a comment listing concrete
options, sets `human:review`, and stops. A blocked item is simply `human:review` with no
`agent:review`, so it sits idle until the user answers and re-adds the trigger.

## Roles & gates

- **User** owns `bmad:*` phase transitions (via `urkon advance` or by hand) and the
  `agent:review` trigger, and **merges PRs**. The agent never merges and never advances a
  phase on its own.
- **Agent** runs the phase's BMAD workflow, writes the artifacts, opens and updates the PR,
  and stamps a `bmad:*` label only at creation.
- **Watcher** (`urkon watch`) polls, runs the label choreography, moves the board, dispatches
  the agent, and streams the trace. It does no git and no PR work.

If the required BMAD skill or command for a phase is unavailable in the agent's environment,
it must **not** improvise: it reports that and stops.

## Board

Status options: **Backlog → Brief → PRD → Architecture → Epics → Story → Dev → QA → Review →
Done**.

The watcher moves the board at each choreography step. A phase's working status is the column
named in the table above; the anytime triggers make no board move; a merged PR's bookkeeping
moves the item to Done.
