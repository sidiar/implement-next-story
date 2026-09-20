# Design notes — why each rule is there

`SKILL.md` states the rules; this file records where they came from, section by section
in the order the README lists them. Each is labelled: **incident** (something went wrong
and the rule is the fix), **observed** (the rule was designed up front and the runs since
bear it out), or **anticipated** (designed up front and never yet exercised). The source
project is [Game of Life Studio](https://github.com/sidiar/game-of-life-studio), a
Next.js/TypeScript app; thirty stories went through the skill between 2026-08-26 and
2026-09-15, and it kept running there after the extraction — the project took 2.0.0 on
2026-09-18 (its PR #54) and two stories have gone through the `bmad` adapter under the
contract since, 3.19 (PR #58) and 4.12 (PR #59), both that day. Where a section's label
changed because of those two runs, the paragraph that changed it is dated.

## The PR is the state machine

*Designed, observed over thirty merges.* The first version of the skill replaced a *commit* gate with a *merge* gate. Until then the
project's rule was "nothing is committed without an explicit go-ahead", which is the right
rule for a human pair and the wrong one for a delegated agent: an agent that stops to ask
before every commit is not delegated. The first commit's message puts the resolution this
way:

> Sprint status is then maintained by the merge itself: the file is versioned, so `done`
> on a branch means "implemented and reviewed" while `done` on main means "merged". Nobody
> hand-edits it, and a PR closed unmerged correctly leaves the story as backlog to be
> redone.

Two questions, two signals. "What is done?" is answered by `main`. "What awaits a human?" is
answered by the set of open PRs. Reading the status file on a branch, or treating an open
PR as done, each gives a wrong answer to the other question, so the rule is stated as a
prohibition: don't. Thirty-eight merged PRs later, the status file has never been
hand-edited on `main`; the closed-unmerged case has not come up yet.

## The re-entry guard fails closed

*Anticipated, in part.* Step 0 was written for a failure mode that had not yet happened: a
usage-limit halt.

> It stops on a dirty tree, an open story PR, an orphaned `story/*` branch, or a
> status-order anomaly — the last two being the states a usage-limit halt leaves behind,
> which the open-PR check alone cannot see. It never self-recovers; interrupted runs are
> reported and left for [the owner].

Thirty runs later, the dangling-branch and partial-story-file checks have never tripped.
They stay because the cost of a false negative (silently redoing or skipping a story) is
much higher than the cost of the check. The open-PR check, by contrast, trips on most
runs — it is what turns "implement the next story" into a no-op while a PR waits.

One bug in that check was caught at design time rather than in production: `gh pr list
--head` takes an exact branch name, so `--head story/` matches nothing and would *fail
open*, letting a second story start on top of an unmerged one. The skill filters on a
prefix with `--jq` instead, and the note stays in `SKILL.md` so nobody simplifies it back.

What did interrupt runs was different. Story 3.7 halted on a benchmark budget it could
not meet without touching something the spec forbade; it raised the conflict, recommended
one resolution, and stopped. The resolution was the owner's to make, and the run's review
phase reads 65 h 45 m on the clock because it resumed three days later, once they had made
it. In story 3.11 the dev subagent hit a usage limit before its bookkeeping; after the
reset the same orchestrator session spawned a second one to verify the diff and close the
story, which is why the run kept its marks and its report shows the 3 h 30 m reset as an
excluded gap rather than as work. Neither produced wreckage — the HALT rule ("HALT
conditions belong to the BMad skills; surface the reason and stop") did its job — but
3.7's number is why *Active* exists (below).

*Observed, 2026-09-18.* The same rule holds one level down, in the config. 2.0.0 reached
the source project as two commits — the subtree pull and the project's new `[adapter]` /
`[models]` tables — and each had added the tables, so the merge (PR #54) declared every
one of them twice. `tomllib` rejects a duplicate table, `lane-gates.py` reads the config
before it reads anything else, and every run on both lanes stopped at Step 0 with the
parser's error and the line number:

> `not valid TOML: Cannot declare ('adapter',) twice (at line 19, column 9)` —
> `lane-gates.py analysed` / `check` read the config, so every `implement-next-story` run
> stops at Step 0 — on both lanes.

The fix was one PR (#57, delete the second block) an hour later. *No config is an error,
not a guess* was written for a missing table; a doubled one is the same case, and a lenient
reader that took the first block would have run on it and never said so.

## One lane per working tree

*Incident.* On 2026-09-13 the skill became lane-aware, and two lanes were launched
the same afternoon from the same checkout. Both passed Step 0 — the tree was clean each
time it was checked — and the first lane's `feat:` commit (story 3.8, PR #26) swept in the
second lane's status lines (`epic-4: in-progress`, `4-1: ready-for-dev`), which lane 4's
create-story had just written into the working tree they shared. The fix — PR #25, opened
before either story PR and merged after both — is the worktree recipe in *Lanes* and the
check right before Step 2 branches:

> Two lanes launched bare in the same checkout pass each other's Step 0 unnoticed — the
> tree is still clean when the second one checks — and the first lane's `feat:` commit
> then sweeps in the second lane's story-status lines. That happened today with 3-8 and
> 4-1.

The lesson generalises: a guard that runs once at the start is a guard against the state at
the start. Anything that must hold at the moment of an irreversible step (branching,
committing) is re-checked at that moment, cheaply, even though it "just passed".

*Second incident, same shape.* On 2026-09-16 the recipe was followed — epic 4 had run in
`lane-epic-4` for days — and then two terminals were opened in the primary checkout and told
`--epic 3` and `--epic 4`. Nothing in the skill knew that epic 4 *had* a worktree: the
mapping lived in the owner's head, and the pre-Step-2 check only fired once both runs had
written into the tree. The fix has two parts, both in `lane-gates.py`:

- `resolve` derives the lane from the tree. `git worktree list` already records every
  `lane-epic-N` worktree, so that is the source of truth — nothing to configure, and it
  goes away with the worktree. A `lane-epic-4` session needs no flag; the primary refuses
  `--epic 4` while `lane-epic-4` exists. (`lane-gates.yaml` was the obvious home and the
  wrong one: it is committed and shared, a worktree path is local to one clone.)
- `lock` closes the window `resolve` cannot see — two sessions in the *same* tree for
  epics that have no worktree yet. One file per working tree, in its git dir so no project
  has to gitignore it, naming the session that holds it. The second Step 0 stops before
  writing anything. A holder silent for an hour across its transcripts is presumed dead
  and taken over, so a run killed by a usage limit does not hold the tree hostage; a holder
  with no transcript at all is not presumed anything, and the owner decides.

The pre-Step-2 re-check stays. The lock is a file; the tree is the evidence.

*The dead-holder case, observed 2026-09-18.* Story 4.12's run hit a usage limit in Step 2
— the dev subagent stopped with one e2e red and its record unwritten, and the orchestrator
session went with it — the first run killed mid-phase since the lock existed. The owner
finished the story by hand in the same tree (a second dev pass on Opus, then the review on
Opus per the table, then the PR), and the dead session's lock sat there throughout. Nothing
was harmed: the lock guards *runs*, and no run tried the tree. The PR's hand-back notes the
lock as the one piece of wreckage — `lock release --force` before the next run — which is
the designed recovery; the hour-of-silence takeover would have done the same at the next
Step 0 without anyone typing it. What the run did not leave behind is a stats table: Step 4
never ran, so 4.12 is the one story since 2026-08-26 whose cost is not on record.

## Never the same model twice

*Incident, then a second one.* Two commits, two weeks apart.

The first version hard-coded the reviewer as Opus and read the implementer from the story
file. The bug was obvious once seen and invisible until then:

> Step 3 hardcoded Opus, so an Opus-implemented story got an Opus review and the two-model
> split silently collapsed.

The fix derived the reviewer as the *complement* of the dev model — which created the second
problem. Stories are escalated to Opus precisely because they are architecture-shaping, the
ones later stories inherit a pattern from. The complement rule sent exactly those diffs to
Sonnet, the weakest available reviewer:

> The complement rule sent architecture-shaping stories — the ones escalated to Opus
> precisely because they set a pattern later stories inherit — to the weakest available
> reviewer. Route those to Fable instead.

Hence the table rather than a rule: `sonnet → opus`, `opus → fable`. Fable sits only on the
Opus row so it never touches the default path, and Step 3 makes the orchestrator say the
derivation out loud before spawning — *"Step 2 ran on X, so Step 3 spawns Y"* — because a
collapsed split is silent and a stated one is not. Ten stories have run Opus + Fable so far.

Since 2.0.0 the table is the project's, `[models.review]` in its TOML, and the derivation is
a script call (`lane-gates.py reviewer`) that refuses a table pairing any model with itself.
The move was prompted by a question, not an incident: the three names were going to be
outdated, and a skill-shipped default would go stale silently — exactly the failure the
table exists to make loud. So the project owns the choice, dated, in its own repo.

One impression rode along, from those ten runs rather than from a comparison: Fable seemed
to do worse under step-by-step prescription than Opus or Sonnet, so its review prompt is
deliberately shorter — the goal, the branch, the two hard rules, and the method left to it.

*Observed, 2026-09-18.* The lookup has run twice in anger, both on the Sonnet row (3.19
and 4.12, Sonnet → Opus); 3.19's second review — after the owner's decision, on the branch
— was also on Opus. 4.12 shows the rule's blind spot: the Sonnet dev pass died on a usage
limit, the owner's resume pass was Opus, and the review was Opus because the story's
`Dev Model:` line still said Sonnet. For the resume's own diff — one e2e fix — the split
collapsed, and nothing said so. The table keys on who the story *says* implemented it; a
hand-finished story is where that can be less than the truth.

## The board and the vocabulary are the skill's

*Anticipated when written; observed since 2026-09-18 — for `bmad`.* This section was
written before anything had run under the contract; the two runs that have (below) each
exercised a different half of it.

v2 made the three phase prompts pluggable and deliberately left two things fixed. The
**board format** — `sprint-status.yaml`, one `key: status` line per story under one
mapping — because *the PR is the state machine* needs `done` on `main` to mean *merged*,
which needs the board to be a versioned file in the repo; an issue tracker cannot be that,
and a second file format has not appeared. The **triage vocabulary** — `patch` / `defer` /
`decision-needed` and the `- [ ] [Review][Decision]` marker line — because the draft-PR
rule, the done-check and the hand-back are all written in those words; an adapter maps its
tool's vocabulary onto them rather than bringing its own. Both were BMad's; both are now
claimed as the skill's, with BMad noted as writing them natively.

The contract's other rule came from a review of the BMad path before anything moved: the
review tool halts five times for a human, and the skill had scripted one of them; the
reviewer inferred the rest correctly in every run — and one of the options it could have
inferred was "start the next story", which is the other lane's story on this branch. So
*an adapter states the answer to every halt its tool presents* is a contract obligation,
and "the subagent will work it out" is named as the defect it is. The `plain` adapter is
the second implementation, written so the seam is not imagined; whether it holds is a
question for its first run — which happened the same day, on a throwaway todo CLI
(README, *The `plain` adapter's first run*): one story, create → implement → review → PR,
no hand between phases, one patch, one deferral, no decision. One run on a thirty-line
story does not make the section *observed*; it does mean the seam is no longer imagined.

*Observed, 2026-09-18.* The `bmad` adapter ran twice under the contract on the source
project, the day after 2.0.0 landed there. Story 3.19 (PR #58) walked the whole vocabulary:
the review raised one `decision-needed` finding — whether `Escape` in the fullscreen stage
should stop the run or exit the stage — wrote it as the `- [ ] [Review][Decision]` line the
done-check counts, left the story short of `done`, and the orchestrator opened the PR as a
draft, an hour and a quarter after Step 0 (Sonnet dev, Opus review, four agents in
Step 3, 340 k output tokens, no idle gap). What 1.1.1 wrote down as the path from draft to
`done` then happened as written, by the owner, on the branch: the answer recorded in the
story file nine minutes after the hand-back, a dev pass for it, a second review on Opus
(twelve more patches, four more deferrals, no new decision), the box ticked, status `done`,
the PR undrafted an hour after it opened, merged that evening. Story 4.12 (PR #59)
exercised the other half — no decision, eleven patches, one deferral, `done` at the
review's hand — and the failure case besides: the run died on a usage limit in Step 2
(*One lane per working tree*, above), and finishing it by hand meant reading the board,
the trailers and the buckets exactly as the contract states them, which is the test of a
vocabulary that it survives the tooling that usually writes it.

So the section is *observed* for `bmad`, on two stories. The seam between the orchestrator
and the adapter has one adapter's evidence at production scale and one adapter's evidence
on a toy; the claim that the board and the buckets are the *skill's* rather than BMad's is
still, in practice, a claim about BMad's runs.

## Gates are rows, not memory

*Anticipated.* When the second epic opened, the two epics' dependencies were analysed once, by hand, and
written down as three rows in `lane-gates.yaml` with an `analysed` entry recording the pair
and the date. `lane-gates.py` reads the rows at every Step 0 and answers *open* or *gated*
with exit codes, so the orchestrator never reasons about a dependency table — it runs a
command. The alternative, remembering dependencies in the skill text or the session, was
rejected in one line: *if it is not a row, it is not a gate.*

No run has yet reached a gated story — the gated ones sit late in their epic — so the rows
have been read on every Step 0 since and have never said *gated* to a live run. And that
first analysis was committed directly to `main`, before the skill required a PR for it;
*Opening a lane* is the same event with the PR the rules now demand.

The proposal half of the rule fired once, on 2026-09-17: story 3.18's create-story noticed
that it was lifting four components out of files the preview panel (4.15) would consume
next, proposed `4-15 requires 3-18` with the reason, and the review confirmed it from the
diff; the PR (#55) carried the row as text, *not written — owner's call*. The owner did not
add it, and did not need to: epic 3 closed on `main` the next day, so the row could never
have said *gated*. The shape held — agents propose, the human decides, the file only changes
through a PR — and the file is unchanged since 2026-09-13.

## Step S — the sync that has conflicted twice

*Anticipated, then observed for one of the three rules.* Five story PRs had been synced with
`main` after the other lane merged first (#26, #33, #35, #36, #37) by the time of the
extraction. All five were clean merges; the conflict rules had never been exercised.
The step existed anyway, because the repo has no branch protection and the failure it guards
against is structural: two PRs can each be green against the `main` they were cut from and
merge to a red `main`. The two mechanisms named in the skill — a gate that measures the
whole tree, and an identifier both branches minted sequentially — are the ones the source
project's CI actually has (a per-route bundle budget; a numbered register of spec
resolutions). They were identified by reading the gates, not by being burned. Two rules
ship with the skill; the project-specific ones live in `implement-next-story.toml` so the
skill stays honest about which resolutions are universal.

*Observed, 2026-09-18 and 19.* Three more syncs in the two days after 2.0.0 landed, and two
of them conflicted, both in the same file: `deferred-work.md`, the project's running list
of what each story chose not to do, which every story appends to and which git therefore
cannot merge when two lanes append at once. Story 3.18's branch synced after 4.11 merged
(#55 after #56) and 4.12's after 3.19 (#59 after #58), and both resolved it by the project's
own rule — *main's hunks first, ours after, nothing dropped* — the first with the diff
verified in both directions and `spec:check` and `prettier --check` green on the merge
commit, as Step S asks. The two rules that ship with the skill still have not fired:
`sprint-status.yaml` auto-merged cleanly both times, so the keep-both rule was never needed,
and no conflict touched a file the rules do not name, so the abort case is untested. The
sequential-identifier rule (`architecture.md`'s M numbers) has not been reached either. The
step is *observed* for exactly the thing the project put in its TOML, and *anticipated* for
what the skill ships — which is the right way round: the rule that fired is the one the
owner wrote for the file they knew would conflict.

## Active time, not wall clock

*Incident.* The first stats table had one time column, mark-to-mark. Story 3.7's 65-hour review phase
made it useless as a measure of work, and the next two runs after the fix showed the
general case: 3.11 read 4 h 44 m on the clock and 74 minutes active; 4.4 read 4 h 30 m and
61 minutes.

> Wall clock between marks counted usage-limit resets and sleep as work. Active time drops
> any >15 min silence across session + subagent transcripts; excluded gaps are listed
> under the table.

The 15-minute threshold is empirical: the longest silence real work produced in thirty
runs (a CI poll, a long tool call) was about ten minutes; the shortest pause worth excluding
was thirty. The footnote lists every excluded gap so an Active figure is auditable against
its wall clock rather than trusted.

## Measured, not estimated

*Designed, observed.* Token counts are the `usage` blocks the API reported on every
assistant turn, summed
per phase over every subagent transcript that *started* inside the phase window. That
attribution rule is what lets `bmad-code-review`'s three parallel hunters count against
Step 3 instead of against nothing, and it works only because phases never overlap.

The orchestrator's own turns are counted too, and broken out: about 9 % of output tokens
across thirty runs. Keeping it there is a design constraint, not an accident — the
subagents report tersely, the orchestrator never reads a transcript (it reads the story
file's `Dev Model:` line and the printed stats table, nothing larger), and Step 4
(writing the stats) is done in-line because a fourth agent would add its tokens to the
number it was reporting. The skill's value is that its own context stays small across an
entire epic.
