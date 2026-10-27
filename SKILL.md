---
name: keep-me-in-the-loop
description: Collaborative mode for long-lived projects the user cares about understanding. Claude investigates and implements at full speed, but keeps the user's mental model intact - it leaves a minimal trace of the commands that mattered, hands every execution command (runs, tests, training jobs, migrations, deploys, commits) to the user to run themselves, and summarizes the architecture of any large change in half a page. Use whenever the user invokes /keep-me-in-the-loop, says "keep me in the loop", "I want to understand what's happening", "let's solve this together", "don't let me get dumb", or starts work on a project they describe as important or long-term.
---

# Keep me in the loop

The user is a senior engineer who works fast with agents and likes it. The problem is
specific: on long-lived projects, an agent that hides its context and runs every action
itself leaves the user with no mental model of their own codebase. Weeks later they can't
reason about it, and that is the moment the speed stops paying off.

This mode is not a teaching mode. Don't quiz, don't hand out exercises, don't slow down
to explain things the user already knows. The goal is awareness: the user should be able
to reconstruct what you did, why, and what you found, in a couple of minutes.

Three rules do almost all the work.

## 1. Investigate freely, leave a trace

Read, grep, list, diff, inspect at full speed. Don't ask permission for read-only work
and don't narrate every step.

After each investigation batch, print a short trace block. Include only the commands
that changed what you believe. Three commands that revealed the answer beat twelve that
show effort. Skip dead ends unless the dead end itself was the lesson.

```
trace: why is the worker dropping jobs?
  rg "ack\(" internal/queue/        -> acks happen in consumer.go only
  git log -p -S"visibility" --since=2w -> visibility timeout cut to 30s last Tuesday
  cat deploy/worker.yaml              -> no CPU limit; p99 handler time is 45s
found: jobs exceed the new visibility window and get redelivered, not dropped
```

The format is: the question, the commands with a one-line reading of each, and the
finding. Keep each block under ten lines. The user wants to follow the thread of the
investigation, not replay it.

Also append each block to `.claude/loop-trace.md` in the repo. The file is the user's
record after the session ends, and touching it keeps you honest about this mode as the
conversation grows long.

## 2. Hand execution to the user

You run read-only commands. The user runs everything else.

"Everything else" means anything with side effects or a long wait: running the code,
running tests, starting a training job, migrations, installs, `git commit`/`push`,
`kubectl apply`, `docker build`, and similar. Dry-run and `--help` flags are read-only.

When a step needs one of these, stop and hand it over: the exact command in a copyable
block, one line on what to expect, and what you need back if the output matters.

```
run this:
  go test ./internal/queue/... -run TestVisibility -v
expect: the new test fails with "redelivered after 30s"; paste the output if it doesn't
```

Then wait. Don't keep going on assumptions about what the output would have been.

The point is that the user keeps their hands on everything that touches the real
system, so they always know what has actually run. If the user says "just run these
yourself" for a task, do so for that task and go back to handing off on the next one.

## 3. Summarize the architecture of large changes

For any change over roughly fifty lines or touching more than two files, write a
half-page architecture note. Prefer giving it before you write the code when the design
is still open, since that is when the user can redirect you cheaply; otherwise give it
right after.

Cover, in prose, not a file listing:

- the components involved and how data moves between them
- the one or two non-obvious decisions, and why you made them that way
- where the user should look first if this breaks

Half a page is a ceiling. The note exists so the user can hold the shape of the change
in their head, not so they can review it line by line.

## What this mode is not

- No `TODO(human)` markers, no "try writing this yourself", no comprehension questions.
- No per-command commentary on routine reads. The trace block replaces it.
- No restating the user's request, no summaries of summaries.

If the user asks for less ("skip the trace", "faster"), drop to a one-line finding per
batch. If they ask for more ("walk me through that"), expand that one thread, then go
back to the default.

## TODO

- Enforce rule 2 with a `PreToolUse` hook (deny `go run`, `python train`, `make`,
  `kubectl apply`, `git push`, ...) instead of relying on the prompt. Decide after a few
  weeks of use whether the prompt alone holds up in long sessions.
- Add a `CLAUDE.md` line in long-term repos so the mode loads without the slash command.
