# Changelog

Dates are the dates the change landed in the source project; the skill was extracted
with its history on 2026-09-15.

## Unreleased

Docs only. `DESIGN.md` records what the source project's runs after 2.0.0 changed: *The
board and the vocabulary are the skill's* is now *observed* for the `bmad` adapter (story
3.19 walked the `decision-needed` → draft → owner → second review → `done` path as 1.1.1
wrote it; 4.12 ran the no-decision half and died on a usage limit in Step 2); *Step S* has
conflicted twice, both times in the project's `deferred-work.md` and resolved by its rule,
the two shipped rules still unexercised; the first gate row was proposed and not needed; a
doubled TOML from the 2.0.0 merge stopped both lanes at Step 0 (fail-closed, as designed);
a hand-finished story is where the review-model lookup can be less than the truth. README
links the source project.

## 2.0.0 — 2026-09-17

**Breaking:** `implement-next-story.toml` needs two new tables, `[adapter]` and
`[models]`; a run stops before Step 0 without them. Copy both from
`implement-next-story.example.toml` — for a BMad project that is `[adapter] name = "bmad"`
and the shipped model table unchanged.

The method is pluggable. Everything BMad-specific in `SKILL.md` — the three skill
invocations, the review's halt answers, the auto-discovery warning, the non-recursive
glob, the resume path for a draft PR — moved verbatim into `adapters/bmad/adapter.md`,
one file with five sections that the orchestrator reads once and appends to its own
instructions when it spawns each phase. `adapters/CONTRACT.md` is what an adapter must
provide, derived from what the orchestrator reads and checks: the board format and its
transition table (adapters own their status transitions, the orchestrator verifies), the
story-file trailers, the three triage buckets and the `- [ ] [Review][Decision]` marker
line the done-check counts, and the rule that every halt a tool presents has a scripted
answer. `adapters/plain/` is the second adapter — no framework, a git repo and a
Markdown plan — and the proof the seam is real: its first run, on a throwaway todo CLI,
went create → implement → review → PR in six and a half minutes with nothing done by
hand between phases (README, *The `plain` adapter's first run*).

Found on the way: `bmad-code-review` halts five times, not four — a checkpoint in its
first step and a chunking offer for large diffs — and the `bmad` adapter now answers both.

Models are roles. `[models]` names which alias plays `create`, `dev`, `dev_escalated` and
`sync`, and `[models.review]` pairs each dev model with its reviewer; `lane-gates.py`
refuses a table that pairs a model with itself, and `reviewer DEV` does Step 3's lookup.
`SKILL.md` refers to roles throughout; its one *Models* section holds the shipped pairing
and the Fable craft note, dated.

Scripts: `lane-gates.py adapter` resolves the adapter file; the board-format literals are
named constants; `story-run-stats.py`'s phase titles are the skill's steps (`create`,
`implement`, `review + PR`), the marker unchanged so re-runs still replace. Tests 44 → 61.

## 1.1.1 — 2026-09-16

Step 3 drives `bmad-code-review` by script rather than by the reviewer's judgement. Its
last step halts four times for a human; the skill answered one (the patch menu) and left
the reviewer to infer the rest — which it did correctly in the five draft-PR runs so far,
but "Start the next story" at the final menu is auto-discovery, the other lane's story
on this branch, and nothing said not to. Now every halt has its answer, and the reason
the story file must be passed is stated: it is what keeps `decision-needed` findings
from being silently reclassified.

Step 3 also no longer writes the story's status — `bmad-code-review` does that itself
(`done`, or `in-progress` with decisions left as action items), and the two instructions
disagreed on the with-decisions case; the reviewer followed the skill and left `review`
in the five runs above. The orchestrator now checks the result instead: `done` ⇔ no
`decision-needed` findings, STOP on a mismatch; draft iff not `done`. Also written down:
how a draft PR reaches `done` (the owner answers the decision items in the story file,
then runs dev-story and code-review on the branch) — the skill never resumes one.

## 1.1.0 — 2026-09-16

The working tree names the lane, and a tree can be held by one run at a time. After two
terminals in the primary checkout were told `--epic 3` and `--epic 4` while epic 4's
worktree sat idle beside them:

- `lane-gates.py resolve [--epic N]` — the lane this working tree serves, from
  `git worktree list`: a worktree named `lane-epic-N` is epic N's, a bare call inside it
  needs no flag, and the primary checkout refuses `--epic N` while that worktree exists.
  Step 0 runs it instead of reasoning about the board by eye.
- `lane-gates.py lock acquire | release | status` — one lock per working tree, in its git
  dir (`.git/` or `.git/worktrees/<name>/`), never tracked. Step 0 takes it, every exit
  releases it, and a second session in the same tree stops at Step 0 with the holder's
  session, epic, story and last activity. A holder silent for an hour is taken over.
- Step 2's re-check now also asks `lock status`.
- `fixtures/`-independent tests for both, on throwaway git repos with real worktrees.

## 1.0.0 — 2026-09-15

First standalone release. The skill as it ran thirty stories in its source project,
decoupled from that project:

- Paths and project-specific sync rules come from `implement-next-story.toml`
  (`[paths]`, `[[sync.rules]]`), read by `SKILL.md` and by `lane-gates.py`; no config is an
  error, not a guess.
- Script invocations resolve through `{skill_dir}` — `${CLAUDE_PLUGIN_ROOT}` as a plugin,
  `.claude/skills/implement-next-story` copied in.
- The human role is "the owner", defined once.
- `## Adapter surface` names every BMad and runtime seam.
- Plugin and marketplace manifests, `fixtures/`, `tests/`, `README.md`, `DESIGN.md`.

## Before extraction

- 2026-09-14 — `story-run-stats.py` reports *active* time with idle gaps excluded and
  listed, beside the raw wall clock.
- 2026-09-13 — one lane per working tree: the worktree recipe and the pre-Step-2 tree
  check, after two lanes launched in one checkout.
- 2026-09-13 — lanes (`--epic N`) and mechanical cross-epic gates: `lane-gates.yaml`,
  `lane-gates.py`, *Opening a lane*, Step S (sync with `main`), the merge-desk rules.
- 2026-09-09 — Opus-implemented stories are reviewed on Fable; the review model is a
  lookup from the dev model.
- 2026-08-26 — every phase is timed and token-counted from the runtime's transcripts
  (`story-run-stats.py`).
- 2026-08-26 — the review model is derived from the dev model, never the same one.
- 2026-08-26 — first version: Step 0 re-entry guard, create → dev → review in fresh
  subagents, the PR as the state machine, the commit gate moved to a merge gate.
