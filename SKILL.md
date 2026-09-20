---
name: implement-next-story
description: Implement exactly one story from sprint-status.yaml end to end — create, implement, review, as the project's adapter defines them — each in a fresh subagent with its own model. Runs as a per-epic lane (`--epic N`) so two epics can proceed in parallel. Stops at review for approval; safe to re-run. Use when the user says "implement the next story".
---

# Implement Next Story

Orchestrates **one** story through three phases — create, implement, review — as the
project's **adapter** defines them (`adapters/CONTRACT.md`; `bmad` ships with the skill).
Does not implement anything itself — it spawns subagents and reads their artifacts off
disk.

One invocation = one story, ending with a PR open for the owner to merge. Repetition is
the caller's job — normally just asking again.

**The owner** is the one human who merges and decides. Nothing reaches `main` without
them, and every STOP below hands back to them.

**Arguments:** `$ARGUMENTS` — optionally `--epic N`. See *Lanes* below.

## Project configuration

The project declares where its artifacts live, and which adapter drives the three
phases, in `implement-next-story.toml` at the repo root (template:
`implement-next-story.example.toml`, in this skill's directory). Read it once, at the
start of Step 0; the `{…}` names used below are its `[paths]` keys, and `lane-gates.py`
reads the same file, so nothing is declared twice.

| Key | Meaning |
| --- | --- |
| `{status_file}` | the board, `sprint-status.yaml` — this skill's format, which BMad writes natively (`adapters/CONTRACT.md` §3); read on `main`, always |
| `{stories_dir}` | where Create writes `{story_key}.md`; story files for every epic in progress sit flat here |
| `{gates_file}` | `lane-gates.yaml`, the cross-epic dependency gates — see *Dependency gates* |
| `{epics_file}` | the epics file, read only when *Opening a lane* |
| `[adapter]` | `name = "bmad"` (shipped: `{skill_dir}/adapters/<name>/adapter.md`) or `dir = "…"` (project-local, relative to the repo root) — exactly one |
| `[models]` | which model alias plays which role — `{models.create}`, `{models.dev}`, `{models.dev_escalated}`, `{models.sync}` — and `[models.review]`, the reviewer for each dev model. See *Models* below |
| `[[sync.rules]]` | the project's own conflict rules for *Step S* — not a path |

No file, or a key missing → STOP before Step 0 and say which; do not guess a layout.

**The adapter file.** Resolve it by script, not by eye, at the same time as the TOML:

```bash
python3 {skill_dir}/lane-gates.py adapter   # prints the adapter.md path
```

Exit 1 is a bad `[adapter]` (missing, both keys, neither) and exit 2 a path with no
`adapter.md` → STOP before Step 0 with the script's output, as for a missing path. Read
the file once and hold it for the run. Its `## Create`, `## Implement` and `## Review`
sections are the spawn-prompt text Steps 1–3 append — whole, verbatim, placeholders
substituted — after the *Subagent instructions* below. Its `## Requires` and `## Notes`
are for the owner.

**Scripts:** `lane-gates.py` (gates, lane resolution, the per-tree lock) and
`story-run-stats.py` (timing and tokens) live in this skill's directory, written `{skill_dir}` in every command below. Installed as a
plugin that is `${CLAUDE_PLUGIN_ROOT}`; copied in, it is
`.claude/skills/implement-next-story`. For stats, call `mark` at each boundary as you
go — a mark you skip is a phase you cannot reconstruct afterwards — and `report` at the
end. See *Run stats* below.

---

## Lanes — one epic per session

A **lane** is one epic worked by one session. Two lanes run in parallel when their epics
are independent enough (a UI epic beside a data-layer epic, say — see *Dependency
gates*). Everything in this skill that used to say "the epic in progress" now means
**this lane's epic**, and every guard is scoped to it: another lane's open PR, dangling
branch, or partial story file is **none of this lane's business** — never report it as a
blocker, never touch it.

- **The working tree names the lane; `--epic N` only confirms it.** A worktree whose
  directory is `lane-epic-N` *is* epic N's lane — `git worktree list` is the record, and
  Step 0 reads it (`lane-gates.py resolve`). In such a worktree a bare call resolves to N,
  and `--epic M` with M ≠ N is refused. In the primary checkout, `--epic N` is refused
  when a `lane-epic-N` worktree exists (that epic lives there — open the session there),
  and a bare call takes the one in-progress epic that has **no** lane worktree. Two
  in-progress epics and neither housed → STOP and ask for `--epic`, because then the
  primary cannot tell which one is its own. Fail closed; the flag is cheap.
- **One lane per working tree, enforced by a lock.** The primary checkout holds one
  lane; every other lane runs in its own worktree. Step 0 takes a per-tree lock
  (`lane-gates.py lock acquire`) that lives in the tree's git dir — `.git/` for the
  primary, `.git/worktrees/lane-epic-N/` for a worktree — so it is never committed and
  needs no `.gitignore` line. A second session arriving in the same tree finds the lock
  and STOPs at Step 0, before anything is written, instead of colliding minutes later.
  The skill never creates worktrees itself, so the session for an additional lane has to
  be *in* one before Step 0 runs. The recipe, three separate prompts in a fresh session
  opened in the primary checkout:

  ```
  use a worktree named lane-epic-N
  ! npm ci                       # or whatever installs this project's dependencies
  implement next story --epic N
  ```

  The worktree line goes first and alone — folded into the skill call, Step 0 can fire
  before the tree switch lands. The install because a fresh worktree has no dependencies
  installed, and the dev agent's first typecheck or test run would fail for reasons
  unrelated to the story.
  Every later run of that lane goes in that same worktree session — and needs no
  `--epic`, the directory name says it. The lane that was opened first stays bare in the
  primary checkout. Before `resolve` and the lock existed, two lanes launched bare in the
  same checkout passed each other's Step 0 unnoticed (the tree is still clean when the
  second one checks), and the first lane's `feat:` commit then swept in the second lane's
  story-status lines — which is how this recipe came to be written down.
- Branch names are the lane's namespace: `story/{E}-*`. The lane-scoping query used
  throughout is `startswith("story/{E}-")` — the trailing dash matters, or lane 3 would
  match `story/30-*` one day and, more to the point, `story/3-` would not match `story/4-`.
- A worktree cannot check out `main` while the primary tree holds it. Step 0 handles
  that; Step 2 branches from `origin/main`, never from a local `main`, for the same reason.

---

## Step 0 — Re-entry guard (always run first)

Mark the clock before anything else, so a run that stops at the guard is still measured:

```bash
python3 {skill_dir}/story-run-stats.py mark step0
```

**Get current.** `git fetch origin` first. A dirty working tree means the owner has work in
progress: STOP, do not stash or clobber it. Then land on current `main`:

```bash
git checkout main && git pull --ff-only   # primary tree
# If git refuses because `main` is checked out in another worktree, this tree is a lane
# worktree — use the detached form instead. Same content, no branch to fight over:
git switch --detach origin/main
```

Every read of `{status_file}` below is from this checkout — i.e. from `main`.

**Resolve the lane** `{E}` from the working tree and the board — never by eye:

```bash
python3 {skill_dir}/lane-gates.py resolve            # or: resolve --epic N
```

Exit 0 prints `LANE {E} — why`; repeat that line to the owner and use `{E}` below.
Exit 2 is `WRONG_TREE` (epic N has a worktree of its own, or this worktree belongs to
another epic — the output names where N runs) or `AMBIGUOUS` (two in-progress epics and
neither has a worktree — ask for `--epic N`) → STOP with the script's output. Exit 1 →
STOP, the tree or the board could not be read.

**Lock the tree** for this run, so a second session landing in the same working tree
stops here rather than after both have written:

```bash
python3 {skill_dir}/lane-gates.py lock acquire --epic {E}
```

Exit 2 is `BUSY`: another session owns this tree — the output names it, its epic and
story, and when it was last active. STOP with that output and do not touch the tree;
the fix is a worktree for one of the two lanes, or `lock release --force` if the owner
knows that session is gone. (A holder silent for an hour is taken over automatically,
with a note in the output — a run killed by a usage limit does not hold the tree
forever.) Exit 1 → STOP.

**The lock is released at every exit of this run** — the end of Step 5, the end of
Step S, and every STOP in between, including the ones in this guard below:

```bash
python3 {skill_dir}/lane-gates.py lock release
```

Releasing is the last thing a run does; do not skip it on the STOP paths, or the next
run in this tree waits an hour for a session that has already ended.

Then, in order — every check scoped to `story/{E}-*`:

- **An open PR for this lane** →

  ```bash
  gh pr list --state open --json number,url,isDraft,headRefName,baseRefName \
    --jq '[.[] | select(.headRefName | startswith("story/{E}-"))]'
  ```

  `--head` takes an exact branch name, not a prefix — `--head story/` matches nothing and
  would fail open, starting a second story on top of an unmerged one.

  If the PR's branch is **behind `origin/main`** — another lane merged since it was
  cut — run **Step S (sync)** and stop there. Check with:

  ```bash
  git merge-base --is-ancestor origin/main origin/{headRefName} && echo current || echo behind
  ```

  Otherwise STOP: report the PR and that it is waiting on the owner. One open story PR
  **per lane** at a time. If in a loop, `ScheduleWakeup` with `noop: true`.

- **A `story/{E}-*` branch without an open PR** → STOP: a previous run died partway, most
  likely on a usage limit. Report the branch, whether it has commits, and whether it is
  pushed. Recovery is the owner's call — delete the branch to redo the story, or push and
  open the PR by hand (finishing it on a stronger model than the story's `Dev Model:`
  line names means updating the line before the review — *Models*, below). Never resume
  into it and never delete it yourself.

  ```bash
  git branch -a --list '*story/{E}-*'
  ```

- **The first `backlog` story of epic {E} is not the lowest-numbered incomplete one in
  epic {E}** → STOP. A story sitting at `ready-for-dev` or `in-progress` with no branch
  means Step 1 died mid-write, leaving a partial story file. Report it; do not skip past
  it to the next `backlog` story, which is what a naive scan would do.
- **The lane pair is unanalysed** →

  ```bash
  python3 {skill_dir}/lane-gates.py analysed --epic {E}
  ```

  Exit 2 means another epic is in progress and `{gates_file}` has no `analysed` entry
  for the pair → go to **Opening a lane** below; no story starts until the owner has approved
  that analysis. Exit 1 is a malformed file → STOP and report it.
- **No `backlog` story left in epic {E}** → STOP. Report the epic is complete. If in a
  loop, `ScheduleWakeup` with `stop: true`.
- **Otherwise** the first `backlog` story of epic {E}, top to bottom, is the candidate —
  and it still has to pass its gate:

    ```bash
    python3 {skill_dir}/lane-gates.py check {story_key}
    ```

    Exit 2 prints the prerequisite, its status on `main`, and why → STOP with that output.
    Do **not** skip to a later story: order within a lane is still order. If in a loop,
    `ScheduleWakeup` with `noop: true` — the other lane will clear it. Exit 1 → STOP, the
    file is broken. Only exit 0 (`OPEN`) makes the candidate this run's target; record it
    on the lock so a `BUSY` seen from elsewhere says which story this tree is on:

    ```bash
    python3 {skill_dir}/lane-gates.py lock acquire --epic {E} --story {story_key}
    ```

**Why the PR and not the status field:** `{status_file}` is versioned, so it says
different things on different refs. On a story branch, `done` means *implemented and
reviewed* — a proposal. On `main`, `done` means *merged*, because merging is the only
thing that writes it there. `main` is therefore the source of truth for epic state, and
the set of open PRs is the source of truth for "is something awaiting approval." Two
questions, two signals — don't try to answer one with the other.

A PR closed without merging is a rejected story: its `done` never reaches `main`, the
story stays `backlog` there, and the next run redoes it. That is correct, not a bug.

**Why the extra checks:** a run halted by a usage limit stops wherever it was, with no
wind-down. The open-PR check alone misses every state before the PR exists — a partial
story file, an unpushed commit, a pushed branch. Each of those would otherwise cause the
next run to silently skip a story or redo one. Assume any interruption is a limit hit
and leave the wreckage for the owner rather than guessing at recovery.

This guard is what makes the skill safe to fire repeatedly. Never skip it.

### Dependency gates — `{gates_file}`

Cross-epic dependencies that story order inside one lane cannot express live in
`{gates_file}`, a cross-epic artifact versioned on `main` like everything else.
`lane-gates.py` reads it; nobody reads it by eye at Step 0. Its shape and the meaning of
`requires` (a story key that must be `done`, or `epic-N` meaning every `N-*` story is
`done` — never the `epic-N: done` row, which only flips in the owner's manual close-out
commit) are documented in the file's header; `lane-gates.example.yaml` in this skill's
directory is the template.

The file has a lifecycle, and every step below plays a part in it:

- **Created — Opening a lane** (next section), one `analysed` entry per epic pair.
- **Updated — by the steps that see dependencies.** Step 1's Create, Step 3's
  Review, and a Step S sync that aborted on a code conflict each propose rows; the owner
  approves; the row rides the story's PR to `main` (edit `{gates_file}` on the branch).
- **Revisited — on two triggers.** When a lane hits a gate, re-read the row before
  reporting it as the reason (is the prerequisite still the right one?), and after any
  `correct-course` that touches an in-progress epic, since rows cite story keys that may
  have moved. Both are the owner's to call; the run just names the trigger.

Never enforce a dependency from memory: if it is not a row, it is not a gate.

### Opening a lane

Entered from Step 0 when `lane-gates.py analysed` exits 2: epic {E} is about to run beside
an in-progress epic {M} that it has never been analysed against. This is the dependency
analysis, done once per pair, and it is **not** part of any story run — no stats marks, no
story branch, nothing implemented.

First, `gh pr list --state open --head lane/{E}-gates`: an open PR means the analysis is
done and waiting on the owner's merge → STOP (`noop: true` in a loop). Do not analyse twice.

Otherwise spawn a subagent, `model: "{models.create}"`, `subagent_type: "general-purpose"`, and tell it to:

1. Read both epics' stories in `{epics_file}` and every existing `analysed`/`gates`
   entry in `{gates_file}`.
2. For each story of epic {E}, decide whether it **needs** an artifact a still-open story
   of epic {M} will create (a hook, a module, a semantic it must follow rather than define),
   or **reshapes a surface** an open {M} story is also reshaping (the same component, the
   same route budget). Ground each finding in the code as it is on `main` — name the file.
   Do the same in the other direction: {M} stories that depend on {E}.
3. Report, tersely: the proposed `gates` rows in the file's exact shape (story, requires,
   why — one sentence each), the stories found free, and any overlap that is *not* a gate
   but will need a Step S sync (so the owner knows what to expect). Write nothing to the file.

Hand the proposal to the owner and STOP. Once they approve, the run that follows adds the
rows and the `analysed` entry (`lane`, `against`, `date`, `approved_by` naming the owner)
to `{gates_file}` in a single `chore: open lane for epic {E} (gates vs epic {M})` commit
on a branch `lane/{E}-gates`, opens a PR, and STOPs — the owner merges it like any other, and
the first `--epic {E}` story run after that finds the pair analysed. If in a loop,
`ScheduleWakeup` with `noop: true` until then.

## Step 1 — Create the story (`{models.create}`)

```bash
python3 {skill_dir}/story-run-stats.py mark step1
```

Spawn a subagent: `model: "{models.create}"`, `subagent_type: "general-purpose"`.
Do **not** use `fork` — it inherits context and ignores the model override.

Its prompt is the *Subagent instructions* plus the adapter's `## Create` for
`{story_key}`. Tell it, in addition, to end the story file with one line naming the
model the dev step should use:

```
Dev Model: {models.dev}   # one-line justification
```

Default `{models.dev}`. Choose `{models.dev_escalated}` only when the story is
architecture-shaping — it picks a pattern that later stories build on, rather than
following one that exists. No third value: the review table below has rows for these two.

Also tell it: **if this story depends on, or reshapes a surface used by, a story in
another in-progress epic** (`{status_file}` says which epics those are), end the story
file with a `Proposed lane gate:` line in `{gates_file}`'s row shape — or `none`. That
line is a proposal for the owner, carried to Step 5; the dev step does not act on it.

Naming the story is not optional, here and in Steps 2 and 3: a method tool left to
discover "the next story" finds the **other lane's** when two epics are in progress.
The adapter's text names it; do not paraphrase that away. Create may dirty exactly
`{story_file}` and `{status_file}` — the `ready-for-dev` line and, on the epic's first
story, `epic-{E}: backlog → in-progress`, which is required: `lane-gates.py` derives the
in-progress epics from those rows.

## Step 2 — Implement (model from the story file)

```bash
python3 {skill_dir}/story-run-stats.py mark step2
```

Before spawning, re-verify the tree is still yours — Step 0's guard ran ten minutes ago.
The lock should have kept every other session out, but the lock is a file and files can
be forced; the tree itself is the evidence:

```bash
python3 {skill_dir}/lane-gates.py lock status   # must print HELD (this session)
git branch --show-current   # must print nothing (detached) or `main`
git status --porcelain      # must list only this story's file and {status_file}
```

Anything else — a `story/*` branch checked out, foreign untracked files, a lock that is
not this session's — means two lanes share this working tree. STOP and report the
branch, files and lock holder; do not branch on top of them. Recovery (a worktree for
one of the lanes) is the owner's call.

Read the `Dev Model:` line from the story file just written. Spawn a subagent with
that model and tell it to:

1. **First**, create and switch to the branch: `git switch -c story/{story_key} origin/main`
   — from `origin/main`, not a local `main` (a lane worktree has none checked out). Do
   this *before* any implementation, so a mid-story failure leaves `main` clean.
2. The adapter's `## Implement` for `{story_file}`. It runs to completion in one
   execution and leaves the story at `review`. Let it — do not add your own checkpoints
   inside it.
3. Commit the work in one commit, message `feat: {story title} (story {id})`.
4. Push the branch to `origin`. **Never push to `main`.** The PR is Step 3's job.

The push is what gets CI to run before anyone reviews — so the reviewer reads real
lint/typecheck/test results rather than the dev agent's account of them.

## Step 3 — Review (the model Step 2 did *not* use)

```bash
python3 {skill_dir}/story-run-stats.py mark step3
```

Re-read the `Dev Model:` line from the story file and spawn a fresh subagent on the
model `[models.review]` pairs it with — a lookup, never a fixed choice:

```bash
python3 {skill_dir}/lane-gates.py reviewer {dev model}   # prints the reviewer
```

Exit 2 means the story names a dev model the table has no row for → STOP; the owner
fixes the table or the story line, not you. Repeat the pair out loud before spawning —
*"Step 2 ran on X, so Step 3 spawns Y"* — and state in the spawn prompt which model
implemented the story and that this review is deliberately a different one, so the
reviewer knows it is the second pair of eyes.

Never the same model twice — the script refuses a table that pairs a model with itself.
A model reviewing its own output re-runs the reasoning that produced the bug and agrees
with itself; the split exists to break that. Escalating dev does **not** license the
escalated model to review — the table escalates the review too: a story is on the
escalated model because it is architecture-shaping, and that is the diff least worth
handing to a weaker reviewer. See *Models* for the shipped pairing and its reasons.

**Driving a review tool with no human at the keyboard.** The prompt is the *Subagent
instructions* plus the adapter's `## Review` for `{story_file}` — which, by contract,
passes the story file and scripts every halt the tool presents. Two rules hold whatever
the adapter says, and the spawn prompt states them:

- **Every `patch` is applied**, unattended — the bucket is defined as fixes that are
  unambiguous without a human, so there is nobody to ask.
- **Every `decision-needed` is left untouched** — written into the story file as an
  unchecked `- [ ] [Review][Decision]` item with the options laid out, never resolved by
  the reviewer. The bucket exists because the fix needs the owner's intent, and the
  draft-PR rule below rests on it.

Tell it, too, to report any cross-epic dependency the diff reveals — a file this story
changed that a story of the other in-progress epic will also need — as a proposed
`{gates_file}` row, alongside the `decision-needed` findings. Same rule: proposed, never
written by the reviewer.

Tell it to check the CI run for the pushed branch (`gh run list --branch story/{story_key}`)
and treat a red run as a finding. If it fixes anything, that lands as its **own** commit
on the same branch, pushed — never amended into the dev commit. The two-commit shape
is the record of what was implemented versus what review changed.

The review sets the story's status in `{status_file}` itself, in that same commit:
`done` when nothing is left open, `in-progress` when `decision-needed` items were left as
action items in the story file. Do not tell it otherwise, and do not correct it afterwards.
When it returns, read `{status_file}` on the branch and count the unchecked
`[Review][Decision]` lines in the story file, and check the two agree —
**`done` ⇔ no open decision** — and STOP on a mismatch rather than editing the file. On the branch, `done` reads as "implemented and reviewed"; it becomes true of the
project when the PR merges, which is the point — the owner never hand-edits the status
file.

Finally, open the PR with `gh pr create --base main --head story/{story_key}`
(never `--auto`). Add `--draft` **iff** the story is not `done`: a draft PR means "the
owner has calls to make before this can merge." Either way the PR opens, so the Step 0
guard sees it and no new story starts in this lane.

A draft PR leaves the branch at `in-progress` with the decisions written as unchecked
`[Review][Decision]` items in the story file. That is the method's resume state, not this
skill's: the adapter's `## Notes` says what the owner does once they have answered them
there, so the story reaches `done` on the branch — then they undraft the PR. This skill
never resumes a draft; Step 0 sees the open PR and stops, as for any other.

Body, four sections, in this order:

- one paragraph: what the story adds and why it lands now
- **What's in it** — the two commits by SHA, each with its rationale
- **Verification** — CI result for the branch, plus the ACs it satisfies
- **Review** — patches applied, items deferred, and any `decision-needed` findings
  written out as explicit questions for the owner

Then **STOP**. Do not merge, do not enable auto-merge, do not approve your own PR.

## Step 4 — Record the run stats

The PR is open, so the run is over: stop the clock and write the numbers down.

```bash
python3 {skill_dir}/story-run-stats.py mark end
python3 {skill_dir}/story-run-stats.py report \
  --story-file {stories_dir}/{story_key}.md --write
```

That prints the table and appends it to the end of the story file under the line
*"This story was implemented with the 'Implement next story' skill with the following
stats:"*. Re-running replaces the previous block rather than stacking a second one, so a
re-run of Step 4 is safe.

Commit it on the story branch and push, so the PR carries it:

```bash
git add {stories_dir}/{story_key}.md
git commit -m "docs: record implement-next-story run stats (story {id})"
git push
```

This is a deliberate third commit. It does not blur the two-commit shape Step 3 protects —
that split is about *implementation vs. review*, and this commit touches no code. Leave the
PR body's two-commit list alone for the same reason: it describes the work, and this is
bookkeeping about the run.

Do this yourself. Do **not** spawn an agent for it: a fourth agent would add its own
tokens to the very numbers it is reporting.

## Step 5 — Hand back to the owner

Report, briefly:

- the lane, story id and title, and its status on the branch (`done`, or `in-progress` if
  decisions are outstanding)
- the PR number and URL, and CI status
- which model implemented it, and which reviewed it — name both, so a collapsed
  split is visible in the hand-back rather than only in the commit trailers — and
  which adapter drove the run
- the file list
- patches auto-applied, and any `decision-needed` findings awaiting the owner's call
- any **proposed `{gates_file}` rows** from Step 1 or Step 3, written out as the row
  they would become — the owner approves or discards; an approved row is added on the story
  branch before merge, so it reaches `main` with the story
- **whether another lane also has an open PR** (`gh pr list --state open` filtered to
  `story/`), because then whichever merges second needs a sync before it is safe — say so
- the stats table — paste it into the hand-back as printed, so the active time and token
  cost of the run are visible without opening the story file (quote Active, not wall clock)

Release the tree — `python3 {skill_dir}/lane-gates.py lock release` — and then
**STOP**. Nothing "waits" — the run simply ends, and the open PR is where the
work sits until the owner merges it. The gate is the **merge**: nothing reaches `main`
without their explicit go-ahead, and approval of one story's merge does not carry
to the next.

---

## Step S — Sync an open PR with `main`

Entered only from Step 0, when this lane's open PR is behind `origin/main`. It exists
because the repo has no branch protection: two PRs can each be green against the `main`
they were cut from and still merge to a red `main`. The two known ways are a gate that
measures the whole tree (a bundle-size budget that both branches spend) and an identifier
both branches minted sequentially (a numbered register of spec resolutions, a migration
sequence). So the PR that merges second must be re-tested against the merged state, and
this is where that happens. The owner never resolves these conflicts by hand.

Spawn a subagent, `model: "{models.sync}"`, `subagent_type: "general-purpose"`, and tell it to:

1. `git fetch origin && git switch story/{story_key}` (create the local tracking branch if
   the worktree lacks it), then `git merge --no-ff origin/main` — a **merge commit**, never a
   rebase and never a squash: the story's commits are its record.
2. Resolve conflicts **only** by rule; anything else is a STOP. Two rules ship with the
   skill:
   - `{status_file}` — keep both sides' status lines; `last_updated` becomes today's date.
   - Any conflict in a file no rule names, or one a rule does not settle →
     `git merge --abort`, report the file and the two sides, STOP. The owner decides. A code
     conflict here is evidence of an overlap `{gates_file}` does not know about: say
     which two stories collided and propose the row that would have sequenced them.

   The rest are the project's `[[sync.rules]]` from `implement-next-story.toml`: each names
   a `file`, states the `rule` in one sentence, and may add a `check` command that must
   pass once the rule is applied. Pass them to the subagent verbatim. A rule is a
   resolution the owner has pre-approved because it needs no judgement — keep both hunks
   in a stated order, give ours the next free number and renumber its citations in this
   branch's diff (`git diff origin/main...HEAD --name-only`). Anything that would need
   judgement to apply is not a rule; it is the abort case above.
3. Commit as `chore: sync story/{story_key} with main after #{merged PR}` and push.
4. Watch CI for the branch — `gh pr checks {pr number} --watch --fail-fast`
   (ten-minute timeout; if it is still running when that expires, say so rather than
   guessing). A red run here is a **finding about the merge**, not about the story:
   report which gate failed and why, and STOP. Do not "fix" a failing gate — a size
   budget, a bench threshold — from inside a sync: changing a gate is the owner's call.
5. Append one line to the PR body under **Verification**: the sync commit SHA, what it
   merged, and the CI result.

Then release the tree (`lane-gates.py lock release`), STOP and hand back: PR, sync
commit, conflicts resolved (by rule) and CI result. No stats report — a sync is not a
story run.

---

## Two lanes, one `main` — rules for the merge desk

- **Merge one PR at a time.** After merging, the other lane's open PR is stale; its next
  run (or a manual "implement the next story --epic N") syncs it and re-runs CI. Merge it
  only once that sync is green. Never merge two PRs that were both cut before the other
  landed.
- Each lane's guard is independent: lane 3 waiting on a review never blocks lane 4 from
  starting its next story, and vice versa.
- Both epics' story files stay flat in `{stories_dir}` until *that* epic is done. The
  adapter's `## Notes` says why for its method (BMad's skills glob non-recursively);
  archive per epic, not per "phase".
- The review load doubles; that is the intended bottleneck. So does CI usage — measure
  your project's runner-minutes per story (ours ran about 40) against the plan's monthly
  cap.

---

## Models — roles, not favourites

`[models]` in the TOML says which Claude Code alias (`model:` on the `Agent` tool)
plays which role: `create` (Step 1, Opening a lane — reads the plan, writes the spec),
`dev` and `dev_escalated` (Step 2), `sync` (Step S), and `[models.review]`, the reviewer
for each dev model. Aliases, not versions: `opus` is whichever Opus is current. What
does go stale is the set of tiers and how they stand to each other — that is why the
table is the project's, required, and not a default this skill ships. `lane-gates.py`
checks it at every read: every role set, every dev model paired, no model paired with
itself.

The lookup keys on the story's `Dev Model:` line, so the line must say who actually
wrote the code. Inside a run it does. **A story finished by hand does not** — a run
that died on a usage limit, a draft PR's resume — because the owner's pass runs on
whatever their session is, and the line still names the model the run planned. Before
the review, set `Dev Model:` to the strongest model that touched the code; otherwise
the lookup pairs the review against the wrong model and the split collapses without
anyone saying so. (Observed once: a Sonnet story resumed on Opus and reviewed on Opus,
because the line said Sonnet.)

**The shipped pairing** (`implement-next-story.example.toml`, as of 2026-09) is
`sonnet → opus`, `opus → fable`. Fable is reachable only through the escalated row, so
it never touches the default path — and it costs roughly double Opus per token before
its longer turns are counted. What the example table cannot carry is craft: **when
Fable reviews, keep your own framing short** — the goal, the branch, the two hard rules
of Step 3 — and leave the method to it; it loses quality under step-by-step
prescription in a way Opus and Sonnet do not. The adapter's `## Review` is still
appended whole: its halt answers are the tool's menu, not method, and dropping them
here would drop them where a wrong menu choice costs most. When the tiers change,
this paragraph is the part to re-check by hand; the table is the part the project edits.

## Run stats

`story-run-stats.py mark <label>` stamps a boundary; `report` turns consecutive
boundaries into phases. Labels, in order: `step0`, `step1`, `step2`, `step3`, `end`.
Marks are kept in `$TMPDIR/implement-next-story-$CLAUDE_CODE_SESSION_ID.json` — keyed to
the session, so two lanes never share a marks file.

Each phase gets: active time, wall clock, the model(s) its agents ran on, and tokens
split input / output / cache-write / cache-read. Token counts are the `usage` blocks the
runtime already recorded per assistant message — measured, not estimated.

Four things worth knowing about what the numbers mean:

- **Active is the number to quote; wall clock is the raw mark-to-mark span.** A run
  paused by a usage-limit reset, a sleeping laptop, or the owner stepping away still has
  its marks hours apart. Active drops those pauses: every transcript entry (session and
  subagents alike) is timestamped, so a stretch with no entries anywhere is idle, and any
  such gap over 15 minutes is excluded (`--idle-gap MINUTES` to override). Real work never
  goes that quiet — the longest gap a tool call or CI poll produces is ~10 minutes — while
  the pauses worth excluding are 30 minutes to days. The footnote under the table lists
  each excluded gap, so an Active figure is always auditable against its wall clock.
- **A phase's cost is its whole subtree.** The report attributes every subagent
  transcript that *started* inside a phase window, so a review tool's own sub-agents
  (BMad's code review spawns three hunters) count against Step 3, not against nothing. This works only because the
  phases run strictly one after another — never overlap two spawns.
- **The orchestrator is counted too**, folded into whichever phase window its turns
  fall in, and broken out again in an *of which* row. That row is a subset, not an
  addition — do not sum it into the total when narrating the table.
- **Cache reads dominate and mean less than they look.** They are billed at a
  fraction of input rate. Read Input and Output for real effort; treat the grand total
  as a ceiling.

The script reads the subagent transcripts under
`~/.claude/projects/*/$CLAUDE_CODE_SESSION_ID/subagents/` and prints only the aggregate.
Never `cat`, `tail`, or `Read` those files yourself — they are full JSONL transcripts and
would blow out the context this skill exists to keep small.

**A run split across sessions loses its marks** (they are keyed to the session id), and
`report` then covers only the phases marked in the current session. That is a degraded
report, not a failure — say so in the hand-back rather than presenting partial numbers as
the whole run.

## Subagent instructions — apply to all spawns

- **Report terse.** The artifacts are on disk; the report is not the deliverable.
  Status, files touched, blockers. No implementation narratives — they would
  accumulate in this context across the epic and defeat the fresh-context design.
- **Name the story.** Pass the story key / file path to every method tool. With two
  epics in progress, a tool left to auto-discover finds the other lane's story.
- **Commit only to `story/{story_key}`.** Never commit or push to `main`. Only Step 3
  opens the PR, and no agent ever merges one — merging is the owner's, always.
- **HALT conditions belong to the method's tools.** If a subagent halts, surface the
  reason and stop the run. Do not work around it. The adapter scripts the answers to the
  halts that have one; anything it does not answer is a blocker.

## Adapter surface — what this skill assumes of the method, the forge and the runtime

Written against Claude Code as of 2026-09. Everything the skill depends on outside its
own files is listed here; these are the seams a port would replace, and the rest of the
skill is agnostic.

- **The method — the adapter.** Everything method-specific lives in the project's
  `adapter.md`: the three spawn prompts, what must be installed, the resume path for a
  draft PR. `adapters/CONTRACT.md` is the agreement — the board format and its transition
  table, the story-file tolerances, the triage vocabulary (`patch` / `defer` /
  `decision-needed`) and the `[Review][Decision]` marker, the every-halt-answered rule.
  `adapters/bmad/` is the reference, written against BMad Method v6.8.0.
- **`{status_file}` shape.** This skill's format (`CONTRACT.md` §3): a
  `development_status:` mapping whose keys are story keys matching
  `^(\d+)-(\d+)-[a-z0-9-]+$` (epic, story, slug — listed in order) and `epic-N` rows;
  story statuses `backlog → ready-for-dev → in-progress → review → done`; epic statuses
  `backlog → in-progress → done`; a top-level `last_updated:` date. `lane-gates.py`
  reads it with a strict stdlib parser — no PyYAML — and rejects anything outside that
  shape.
- **`{gates_file}` shape.** This skill's own file: `analysed:` and `gates:` lists of
  flat mappings, documented in `lane-gates.example.yaml`.
- **Branches, PRs, and `gh`.** The default branch is `main`; story branches are
  `story/{story_key}`, lane-opening branches `lane/{E}-gates`. PRs are GitHub PRs driven
  through the `gh` CLI, and a *draft* PR is the "decisions outstanding" signal. Six `gh`
  calls, all in this file — the seam for another forge.
- **Claude Code runtime.** The `Agent` tool with a `model:` override and
  `subagent_type: "general-purpose"`; `ScheduleWakeup` when run under `/loop`; worktrees;
  `$CLAUDE_CODE_SESSION_ID`; and the transcript layout `story-run-stats.py` scrapes
  (`~/.claude/projects/*/{session}/subagents/*.jsonl`), which is undocumented and may
  change with any release. The skill assumes it runs without permission prompts. Not a
  seam: the runtime is the product.
