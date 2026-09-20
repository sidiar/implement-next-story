# bmad — BMad Method v6 adapter

Written against BMad Method v6.8.0, 2026-09. The three sections `Create`, `Implement` and
`Review` are spawn-prompt text; `Requires` and `Notes` are for the owner. See
`adapters/CONTRACT.md` for what each must do.

## Requires

- BMad Method v6 installed in the project (`_bmad/`), written against 6.8.0.
- Its `bmad-create-story`, `bmad-dev-story` and `bmad-code-review` skills in `.claude/skills/`.
- `_bmad/config.toml`: `implementation_artifacts` is `[paths] stories_dir` and holds
  `status_file`; `planning_artifacts` holds `epics_file`.

## Create

Invoke `bmad-create-story` for `{story_key}` **by name** — give it the key; never let it
auto-discover. Its discovery takes the first `backlog` story in the whole of
`{status_file}`, which with two epics in progress is the other lane's. It writes
`{story_file}`, sets `{story_key}: ready-for-dev`, and on the epic's first story flips
`epic-{E}: backlog → in-progress` itself; both writes are expected. Its HALTs — no such
story in `{epics_file}`, missing planning documents, an epic already `done` — end the run:
report the reason and stop.

## Implement

Invoke `bmad-dev-story` on `{story_file}` — the path, never a bare call (bare, it takes
the first `ready-for-dev` story in `{status_file}`). It runs to completion on its own:
marks the story `in-progress`, implements every task, runs the project's checks, and
leaves the story at `review`. Let it — add no checkpoints inside it. Its HALTs are its
own — a dependency the story did not ask for, missing configuration, three consecutive
failed attempts, an ambiguous task — and end the run: report the reason and stop.

## Review

Invoke `bmad-code-review` on `{story_file}` — the path, always. That is what sets its
`review_mode` to `full`; without it the review silently reclassifies every
`decision-needed` finding as `patch` or `defer`, and the draft-PR rule never fires. It
stops for a numbered answer several times. Answer as follows, and never otherwise:

- **Step 1 checkpoint** (diff stats, review mode) — proceed. If it offers to chunk a
  large diff, decline: review the whole.
- **§4, resolve `decision-needed`** — do not. Leave each as an unchecked
  `- [ ] [Review][Decision]` item in the story file with the options laid out, and go on
  to §5. Its "decisions before patches" rule assumes the decider is present; here the
  decider is the owner, after the PR.
- **§5, the `patch` menu** — **1, "Apply every patch"**, no per-finding confirmation.
- **§6, status** — let it write: `done` when nothing is left open, `in-progress` when
  decisions were left as action items. Do not correct it.
- **§7, next steps** — **3, "Done"**. Never 1, "Start the next story": that runs
  dev-story on the first `ready-for-dev` story in the file — the other lane's, on this
  branch.

Its `dismiss` bucket is dropped at triage; `patch`, `defer` and `decision-needed` are the
skill's vocabulary as written, and its `[Review][Decision]` lines are the open-decision
marker as written.

## Notes

- **Resume path for a draft PR.** The branch is at `in-progress` with unchecked
  `[Review][Decision]` items in the story file. Once the owner has answered them there,
  they run `bmad-dev-story` and then `bmad-code-review` on the story file, on the branch —
  dev-story picks up the unchecked review items, the second review flips the story to
  `done` — and undraft the PR. The skill never resumes a draft itself. If that dev-story
  pass runs on a stronger model than the story's `Dev Model:` line names, change the line
  to that model first, so the second review is looked up against the model that actually
  wrote the code (SKILL.md, *Models*).
- BMad's skills glob story files **non-recursively**: a story moved into a subfolder is
  invisible to create-story, dev-story and retrospective. Keep every in-progress epic's
  story files flat in `{stories_dir}`; archive per epic once it is done, never per phase.
- `bmad-code-review` also appends each `defer` finding to BMad's deferred-work file
  (`deferred-work.md` beside `{status_file}`), so a story PR can touch it. A
  `[[sync.rules]]` entry for that file is in `implement-next-story.example.toml`.
