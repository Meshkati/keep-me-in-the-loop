# Keep me in the loop

A Claude skill for long-lived projects you care about understanding.

## Why I built this

I'm all in on AI-assisted coding. On most work it makes me dramatically faster, and I
wouldn't go back. But on the projects I care about most, the long-lived ones I'll still
be maintaining a year from now, I noticed something uncomfortable: the agent was holding
all the context and running every action, and I was slowly losing my grip on my own
codebase. I felt like I was getting dumber.

My first fix was to have Claude hand me every command to run myself. That kept me sharp,
but it turned every investigation into a slow back-and-forth.

This skill is where I landed instead. The agent investigates at full speed but leaves a
short trace of the commands that actually mattered, so I can follow its line of thought.
Anything that touches the real system, like running code, starting a training job or
pushing, comes back to me to run. And every large change arrives with a half-page sketch
of its architecture. I stay fast, and I still know what's going on.

Turns out this has a name,
[cognitive debt](https://margaretstorey.com/blog/2026/02/09/cognitive-debt/), and
[Anthropic's own study](https://www.anthropic.com/research/AI-assistance-coding-skills)
found that developers who used AI to build understanding, not just to produce code,
came away with much stronger mastery.

## What it does

It is not a teaching mode: no quizzes, no exercises, no `TODO(human)`. The goal is that
you can reconstruct what Claude did, why, and what it found, in a couple of minutes.

**1. Investigates freely, leaves a trace.** Claude reads, greps and diffs at full speed,
then prints a short block with only the commands that changed what it believes:

```
trace: why is the worker dropping jobs?
  rg "ack\(" internal/queue/        -> acks happen in consumer.go only
  git log -p -S"visibility" --since=2w -> visibility timeout cut to 30s last Tuesday
  cat deploy/worker.yaml              -> no CPU limit; p99 handler time is 45s
found: jobs exceed the new visibility window and get redelivered, not dropped
```

Each block is also appended to `.claude/loop-trace.md` in your repo, so you have a
record after the session ends.

**2. Hands execution to you.** Claude only runs read-only commands. Anything with side
effects or a long wait (running the code, tests, training jobs, migrations, installs,
commits, pushes, deploys) comes back to you as a copyable command, with what to expect:

```
run this:
  go test ./internal/queue/... -run TestVisibility -v
expect: the new test fails with "redelivered after 30s"; paste the output if it doesn't
```

Say "just run these yourself" to lift this for the current task.

**3. Summarizes the architecture of large changes.** For anything over roughly fifty
lines or more than two files, you get a half-page note: the components and how data
moves between them, the non-obvious decisions, and where to look first if it breaks.

## Install

### Claude Code

Clone it into your personal skills directory to have it in every project:

```
git clone https://github.com/Meshkati/keep-me-in-the-loop.git ~/.claude/skills/keep-me-in-the-loop
```

To use it in one project only, clone it into that project's `.claude/skills/` instead.
To update later, run `git pull` inside the cloned folder.

### claude.ai

Zip the `keep-me-in-the-loop` folder (with `SKILL.md` directly inside it) and upload it
as a custom skill in claude.ai's skills settings.

## Usage

Invoke it with `/keep-me-in-the-loop`. It also triggers when you say things like
"keep me in the loop", "I want to understand what's happening", or "let's solve this
together".

To have it on by default in a repo, add a line to that repo's `CLAUDE.md`:

```
Use the keep-me-in-the-loop skill for all work in this repo.
```

Adjusting it mid-session:

- "skip the trace" or "faster": drops to a one-line finding per investigation
- "walk me through that": expands one thread, then goes back to the default

### The trace file

`.claude/loop-trace.md` is written into whatever repo you're working in. Commit it if
you want the record shared with your team, or add it to `.gitignore` if not:

```
echo ".claude/loop-trace.md" >> .gitignore
```

## Roadmap

- Enforce rule 2 with a `PreToolUse` hook (deny `go run`, `python train`, `make`,
  `kubectl apply`, `git push`, ...) instead of relying on the prompt alone.

## License

[MIT](LICENSE)
