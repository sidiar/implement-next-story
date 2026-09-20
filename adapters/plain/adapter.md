# plain — no framework, just a git repo and Claude Code

A story is a Markdown file; the plan is `{epics_file}`; the checks are whatever the
project runs. Nothing to install. `adapters/CONTRACT.md` says what each section must do.

## Requires

- `{epics_file}` with one `## Epic N` heading per epic and, under it, one entry per story:
  a `### Story N.M: Title` heading or a `- **N-M-slug** — description` bullet, followed
  by whatever the plan says about it (fixtures/docs/planning-artifacts/epics.md is the shape).
- `[adapter.plain] check = [...]` in `implement-next-story.toml`: the commands that must pass
  before a story is done — `["npm run lint", "npm test"]`, `["make check"]`, or `[]`.
- `story-template.md` beside this file, which `## Create` reads via `{skill_dir}`.

## Create

Write `{story_file}` for **`{story_key}`** and no other story. Find it in `{epics_file}`:
the `## Epic {E}` section, then the entry whose key is `{story_key}` or whose number is
`{E}.M` for this story, and everything under it up to the next story or epic. Copy
`{skill_dir}/adapters/plain/story-template.md` to `{story_file}` and fill every section
as the template's own text directs, from that entry and from the code as it is on `main`
— read the files you cite, do not guess. Then set `{story_key}: ready-for-dev` in
`{status_file}`, and if `epic-{E}` is `backlog`, set it `in-progress`. Touch nothing
else. Halt if the story is not in `{epics_file}` or has no description at all, or if
`epic-{E}` is `done`.

## Implement

Implement `{story_file}` completely, in this execution. First set `Status: in-progress`
in the file and `{story_key}: in-progress` in `{status_file}`. Then every unchecked task,
in order, following the dev notes — including any unchecked `[Review][Decision]` line
under *Review Findings*: the owner's answer is written beneath it; implement that answer
and tick the line. Run every command in `[adapter.plain] check` (read
`implement-next-story.toml`); all must pass. Fill *File List* with every file touched and
*Completion Notes* with what a reviewer needs to know; leave the `Dev Model:` and
`Proposed lane gate:` lines exactly as they are. Finish with `Status: review` in the file
and `{story_key}: review` in `{status_file}`. Halt on an AC you cannot make testable, a
dependency the story did not ask for, an owner's answer you cannot implement as written,
or a check that still fails after three honest attempts.

## Review

Review `git diff origin/main...HEAD` against `{story_file}` — its acceptance criteria are
the spec; a code change the ACs do not explain is a finding, and an AC the diff does not
meet is a finding. `{story_file}`, `{status_file}` and the `Dev Model:` / `Proposed lane
gate:` lines are the story's own bookkeeping, not findings. Triage every finding into
exactly one bucket:

- `patch` — a real issue whose correct fix is unambiguous without human input.
- `defer` — real, but pre-existing or outside this story's scope.
- `decision-needed` — real, but the fix depends on the owner's intent: an ambiguous
  choice, a product call, a trade-off the story does not settle. Never resolve one.

Drop noise. If there are findings, write them under `### Review Findings` at the end of
`## Tasks / Subtasks` — create the heading if absent, append to it if present — in this
order and shape: `- [ ] [Review][Decision] <title> — <the options, one line>`, then
`- [ ] [Review][Patch] <title> [<file>:<line>]`, then `- [x] [Review][Defer] <title>
[<file>:<line>] — deferred: <why>`. Apply every patch and tick its line; re-run every
command in `[adapter.plain] check`, and if one fails after a patch, revert that patch,
leave its line unchecked, and halt. Set the status in both `{story_file}` and
`{status_file}`: `done` if no `[Review][Decision]` line is left unchecked, else
`in-progress`. Report by bucket: patches applied, items deferred, decisions as questions
for the owner. This adapter presents no prompts; there is nothing to answer.

## Notes

- **Resume path for a draft PR.** The branch is at `in-progress` with unchecked
  `[Review][Decision]` lines. The owner writes their answer beneath each line, leaves it
  unchecked, and in a session on the branch runs the `## Implement` prompt, then the
  `## Review` prompt, placeholders filled by hand — Implement implements each answer and
  ticks its line; Review flips the story to `done` — then commits, pushes, and undrafts
  the PR. The skill never resumes a draft itself. If that Implement pass runs on a
  stronger model than the story's `Dev Model:` line names, change the line to that model
  first, so the Review is looked up against the model that actually wrote the code
  (SKILL.md, *Models*).
- Nothing here globs `{stories_dir}`; keep story files flat anyway, the orchestrator reads
  `{stories_dir}/{story_key}.md` by name.
- The only files a story PR touches outside the code are `{story_file}` and `{status_file}`.
- Written 2026-09-17 as the proof that the seam is real; `fixtures/` is a project it can run on.
