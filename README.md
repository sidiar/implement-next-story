# implement-next-story

A story pipeline on [Claude Code](https://docs.claude.com/en/docs/claude-code): it takes
the next story off a sprint board and turns it into a pull request, spawning three fresh
subagents in turn — *create the story*, *implement it*, *review it on a different model* —
and stops with the PR open for one human to merge. It implements nothing itself; it
orchestrates, reads artifacts off disk, and refuses to guess. Two epics can run as
parallel lanes with dependency gates between them, and every run is timed and
token-counted per phase.

What the three phases *do* is the project's **adapter**. Two ship: `bmad`, for
[BMad Method](https://github.com/bmad-code-org/BMAD-METHOD) v6, which the skill was
written on; and `plain`, which needs nothing but a git repo and a Markdown plan. Writing
a third is an afternoon — [`adapters/CONTRACT.md`](adapters/CONTRACT.md) is the whole
agreement.

Terms: an *epic* is a group of *stories* (units of work with acceptance criteria), and
`sprint-status.yaml` is the board that lists each story's status — BMad's file, whose
format the skill claims as its own. A *lane* is this skill's word for one epic being
worked by one session.

## What it did on a real project

Thirty stories of [Game of Life Studio](https://github.com/sidiar/game-of-life-studio), a
Next.js/TypeScript app, 2026-08-26 → 2026-09-15, measured by the skill itself from the
runtime's own transcripts — not estimated. The numbers are in each story file in the source
project; this is the aggregate. The project consumes the skill as a git subtree and has kept
running it since — [DESIGN.md](DESIGN.md) records what the runs after 2.0.0 changed.

| | Median per story | Notes |
| --- | ---: | --- |
| Time, start → PR open | **67 min** | 22 uninterrupted runs, range 43–106 min |
| &nbsp;&nbsp;re-entry guard | 25 s | no agent; git and two scripts |
| &nbsp;&nbsp;create | 10 min | one Opus agent |
| &nbsp;&nbsp;implement | 26 min | one agent, Sonnet or Opus per story |
| &nbsp;&nbsp;review + PR | 32 min | one agent plus BMad's three parallel review hunters |
| Output tokens | 180 k | 9 % of them the orchestrator's own |
| Cache-read tokens | 82.6 M | 97 % of the 85 M median total; priced at a fraction of the input rate |
| Subagents per run | 6 | create, dev, review, and the review's three hunters |

Thirteen stories were implemented on Sonnet and reviewed on Opus. Seventeen were
implemented on Opus — more than the design expects, because the middle epic was the
simulation engine — and reviewed on Fable for the ten stories after 2026-09-09, when that
pairing was introduced; the seven before it were reviewed on Sonnet, which is the weakness
the pairing fixed. Eight of the thirty runs paused mid-way — a usage-limit reset, a laptop
closed for the night, once for three days while the owner decided a spec conflict — which
is why the report distinguishes *active* time from wall clock: the two most recent paused
runs took 4 h 44 and 4 h 30 on the clock and 74 and 61 minutes of work.

## Five decisions that carry it

- **The PR is the state machine.** `main` is the source of truth for what is done; the set
  of open PRs is the source of truth for what awaits a human. A story branch's `done` is a
  proposal; it becomes fact when the PR merges. Nobody edits the status file by hand.
  ([why](DESIGN.md#the-pr-is-the-state-machine))
- **The re-entry guard fails closed.** Every run starts by asking, in order: is this the
  right working tree for the lane, is the tree free, is there an open PR for this lane, a
  dangling `story/*` branch, a half-written story file, an unanalysed epic pair, a gate not
  yet satisfied? Any yes stops the run and hands back. That is what makes it safe to fire
  on a loop — and the checks that matter most are re-run right before the irreversible
  step. ([why](DESIGN.md#the-re-entry-guard-fails-closed),
  [the incident](DESIGN.md#one-lane-per-working-tree))
- **The working tree names the lane.** A git worktree called `lane-epic-4` *is* epic 4's
  lane: `git worktree list` is the record, so there is nothing to configure and nothing to
  remember. A session in that worktree needs no `--epic`; the primary checkout refuses
  `--epic 4` while that worktree exists. Each tree also carries a lock in its git dir, so
  a second session landing in the same tree stops at Step 0 instead of colliding later.
- **Never the same model twice.** The reviewer is derived from the implementer by a lookup
  table, not chosen — `[models.review]` in the project's TOML, shipped as Sonnet → Opus,
  Opus → Fable, and refused by the script if any row pairs a model with itself. The premise
  is that a model reviewing its own output tends to re-run the reasoning that produced the
  bug; the rule became a table after the split silently collapsed once.
  ([why](DESIGN.md#never-the-same-model-twice))
- **Gates are rows, not memory.** A cross-epic dependency exists only as a line in
  `lane-gates.yaml`, versioned on `main` and checked by a script. Rows are proposed by the
  agents that notice them, approved by the human, and ride the story's PR.
  ([why](DESIGN.md#gates-are-rows-not-memory), [the sync step](DESIGN.md#step-s--the-sync-that-has-conflicted-twice))
- **Measured, not estimated.** Every phase reports the tokens the API's `usage` blocks
  recorded for every agent that started inside it, and active time with idle gaps listed so
  the figure is auditable against the wall clock.
  ([why](DESIGN.md#measured-not-estimated), [active vs wall clock](DESIGN.md#active-time-not-wall-clock))

[DESIGN.md](DESIGN.md) tells which of these came from an incident and which were designed
in anticipation and have not fired yet.

## Install

The repo root is the plugin, and `SKILL.md` at the root is its single skill, so it installs
three ways; pick one.

**Marketplace** — the repo is also its own one-plugin marketplace:

```
/plugin marketplace add sidiar/implement-next-story
/plugin install implement-next-story@implement-next-story
```

**Copy-in** — clone (or `git subtree add`, which is how the source project consumes it)
into `.claude/skills/implement-next-story/` in your project. The manifest makes Claude Code
load it as a project-scope plugin the next session, after the workspace trust prompt;
nothing else to install.

**Try it first** — `claude --plugin-dir path/to/implement-next-story` loads it for one
session.

**Configure:** copy `implement-next-story.example.toml` to `implement-next-story.toml` at
your project root. Name the adapter (`[adapter] name = "bmad"` or `"plain"`, or `dir =`
for one of your own), say which model plays which role (`[models]` — the example carries
the shipped pairing and its reasons), and point `[paths]` at your artifacts. Add a
`[[sync.rules]]` entry for each file whose merge conflicts can be resolved mechanically
without a human; the example carries the three from the source project. Nothing is
defaulted: a missing table stops the run before it starts.

**Run:** with `sprint-status.yaml` holding at least one `backlog` story, say

```
implement the next story
```

or `implement the next story --epic 4` to name the lane. It runs the guard, then
create → dev → review, opens the PR, prints the stats table, and stops. Say it again for
the next story, or put it on `/loop` (Claude Code's recurring-run mode) — the guard makes
repeated firing safe. To run a second epic in parallel, open a git worktree named
`lane-epic-N` in a fresh session and say the same phrase there — the worktree's name is
the lane, so `--epic` is only needed the first time, before the worktree exists;
SKILL.md's *Lanes* section has the recipe.

It is built to run unattended, with permission prompts bypassed: it creates branches,
pushes them, and opens PRs without asking. It never merges, never pushes to `main`, and
never resolves a review finding that needs a human decision — those open the PR as a
draft.

`fixtures/` is a three-epic toy board you can point the scripts at without a real project:

```
python3 lane-gates.py --root fixtures list
python3 lane-gates.py --root fixtures adapter     # the adapter.md the config names
python3 lane-gates.py --root fixtures reviewer opus
python3 lane-gates.py resolve            # which lane this checkout serves
python3 lane-gates.py lock status        # who, if anyone, is running in it
python3 -m unittest
```

## Adapters

An adapter is one Markdown file with five sections: what it requires, and the spawn-prompt
text for *Create*, *Implement* and *Review*, plus notes for the owner. The orchestrator
reads it once per run and appends each section to its own instructions when it spawns
that phase. Everything else — the board format, the story-file trailers, the three triage
buckets `patch` / `defer` / `decision-needed` and the `[Review][Decision]` marker the
draft-PR rule counts, which phase may write which status — is the skill's, and
[`adapters/CONTRACT.md`](adapters/CONTRACT.md) says so section by section.

- **`bmad`** — invokes BMad Method v6's `bmad-create-story`, `bmad-dev-story` and
  `bmad-code-review` by name, and scripts every one of the review's five halts so it runs
  with nobody at the keyboard. The thirty stories above ran on it.
- **`plain`** — no framework: Create fills a story template from the epic's entry in your
  plan; Implement works the tasks and runs whatever `[adapter.plain] check` lists;
  Review diffs the branch against the acceptance criteria and triages in the skill's
  vocabulary. The proof that the seam is real, and the way to use the skill without BMad.
- **Yours** — `[adapter] dir = "docs/my-adapter"`; `tests/test_adapters.py` checks the
  shape, and a story run on a throwaway repo checks the rest.

### The `plain` adapter's first run

[`implement-next-story-on-plain`](https://github.com/sidiar/implement-next-story-on-plain)
is a throwaway project — a stdlib Python todo CLI, `add` implemented, `list` / `done` /
`remove` in the backlog, `python3 -m unittest` as the check, a two-line CI workflow. Its
first story went through the skill on 2026-09-17, the day the adapter was written, with
the orchestrator following `SKILL.md` and nothing done by hand between phases. The
hand-back, as reported:

> **Lane 2, story 2.2 `list-items`** — status on the branch: `done`. PR #1, not draft, CI
> green on both pushes. Adapter `plain`; implemented on **Sonnet**, reviewed on **Opus**
> (`lane-gates.py reviewer sonnet`). Files: `todo.py`, `test_todo.py`, the story file, the
> board. Review: 1 patch applied (a missing test for AC 3's second clause), 1 deferred
> (temp-dir leak, pre-existing from story 2-1), 0 decisions. Proposed gate rows: none.
>
> | Phase | Model | Active | Output tokens |
> |---|---|---:|---:|
> | Step 0 — re-entry guard | — | 9 s | 1,050 |
> | Step 1 — create | opus-5 | 1 m 25 s | 6,157 |
> | Step 2 — implement | sonnet-5 | 2 m 13 s | 12,085 |
> | Step 3 — review + PR | opus-5 | 2 m 49 s | 10,855 |
> | **Total (create → PR ready)** | | **6 m 37 s** | 30,147 |

Six and a half minutes for a thirty-line story is the floor, not the norm — the thirty
stories above took an hour each on a real codebase. What the run checked is the seam:
`[adapter]` resolution, the `[models]` lookup, the `[Review][Decision]` done-check, the
two-commit shape, CI as a finding, the stats — every rule the orchestrator applies held
with a method that has no tooling of its own. The reviewer also found a place it could
have raised a decision (`list extra` still lists) and correctly did not, because the
story's dev notes allowed it.

## What it depends on

The skill itself:

- **Claude Code** (verified 2026-09) — subagents with a per-agent model choice, the
  recurring-run mode, git worktrees, and the session/subagent transcript layout under
  `~/.claude/projects/`, which `story-run-stats.py` reads to count tokens. That layout is
  undocumented and may change with any release.
- **GitHub and the `gh` CLI** — PRs, draft PRs, `gh pr checks --watch`. No GitLab support.
- **Python ≥ 3.11**, standard library only (`tomllib`, no PyYAML — both scripts carry
  strict readers for exactly the file shapes they accept).
- **Models** — whichever Claude Code aliases `[models]` names. The shipped table is
  Sonnet for dev and sync, Opus for create and escalated dev, and Fable (Anthropic's
  largest as of 2026-09, at roughly twice Opus's per-token price) reviewing only the
  Opus-implemented stories. Tiers change; the table is yours to edit, and the *Models*
  section of `SKILL.md` is the part to re-read when they do.

The `bmad` adapter, in addition:

- **BMad Method v6** (written against 6.8.0) — the three skills and the
  `sprint-status.yaml` file it writes natively. The review's `dismiss` bucket is dropped;
  its other three are the skill's vocabulary as written.

## Not in v2

Listed so the omission reads as a decision:

- **Other forges.** Six `gh` calls and draft-PR-as-signal, all in `SKILL.md`; a `[forge]`
  table is the shape when someone needs GitLab.
- **Other board formats.** `sprint-status.yaml`'s shape is the skill's; a method that keeps
  its board elsewhere mirrors it here. `read_sprint_status` in `lane-gates.py` is the
  one-function boundary if a second file format ever exists.
- **Runtime abstraction.** Subagents with per-agent models, worktrees, transcripts —
  that coupling is the product, not a seam.

## Attribution

Written on and for [BMad Method](https://github.com/bmad-code-org/BMAD-METHOD) (MIT),
whose skill names and menu text the `bmad` adapter quotes and whose board format the skill
adopted. This repository is not part of BMad and is not
endorsed by it. Extracted, with history, from the project it was written for. MIT licence.
