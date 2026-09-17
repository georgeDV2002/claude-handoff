# handoff

A Claude Code skill that closes out a working session: it writes a `SESSION.md`
the next session reads to resume, rotates the previous one into an archive, copies
anything worth keeping out of `/tmp`, and commits — in each repository the session
touched.

## The problem

Git records what changed. It does not record why, which approach was tried first
and abandoned, or which obvious-looking fix turns out to break something three
files away. That part of a session exists only in the conversation, so it goes when
the context window fills and the conversation is replaced.

The next session then starts without it. It has the code and none of the reasoning,
so it re-derives what it can and re-proposes what was already ruled out, and the
first stretch of work goes into reconstructing context that existed an hour earlier.

## Two jobs

### 1. The next session continues the work

`/handoff` writes one file, `SESSION.md`, in the repository root. It is not a
summary of the conversation. It is the state needed to keep going: what was
completed, what was decided and on what grounds, what was tried and failed, the
next concrete step, and the questions still open.

It is deliberately short, because a handoff the next session cannot afford to read
in full is not doing its job, and session history is kept in a separate file for
that reason.

The file opens by addressing the next session directly, so resuming works whether
or not that session has this skill installed:

> **New session:** read only this file, then report to the operator (1) where we
> left off, (2) pending decisions and open questions — and WAIT for confirmation
> before acting.

### 2. The repository keeps a record of its own decisions

Each run moves the previous handoff into `SESSION_ARCHIVE.md`, numbered and newest
first, and existing entries are never edited. Over time this accumulates the part of
a project's history that version control does not hold.

`git log` will tell you that a batch size changed. The archive entry records that a
larger batch was tested, that it pushed p99 latency past the freshness target the
dashboard depends on, and that the current value is the ceiling rather than a
default — which is the answer to a question that tends to come up again months
later. [`examples/SESSION.md`](examples/SESSION.md) shows the full shape.

Decisions, rejected approaches, and the reasoning behind both, recorded in the
repository in the order they happened.

## Install

```bash
git clone https://github.com/georgeDV2002/claude-handoff
mkdir -p ~/.claude/skills/handoff
cp claude-handoff/claude-code/handoff/SKILL.md ~/.claude/skills/handoff/
```

Run `/handoff` as a session winds down, rather than once the context window has
already forced the issue.

## What it writes

**`SESSION.md`** — the current session only, in the repository root. Ten sections:
Bigger Picture, Objective, Git State, Completed This Session, Decisions Made, Dead
Ends — Do Not Retry, Rescued Artifacts, Context That Would Be Lost, Next Step, Open
Questions.

**`SESSION_ARCHIVE.md`** — every previous handoff, newest first. It carries an
explicit instruction not to load it at session start; it is there to be searched
when a particular question needs history, which lets it grow without costing the
next session any context.

**`~/.claude/session-artifacts/<repo>-<timestamp>/`** — the files the session left
in `/tmp` that were worth keeping, with a `MANIFEST.md` recording what each one is
and how it was produced. Scripts, extracted datasets, query output: work that took
real time and would be deleted at the next reboot. Credentials, tokens and `.env`
files are excluded by rule.

A session that spanned three repositories produces three handoffs, each standing on
its own.

## Design notes

- **`SESSION.md` holds one session, not a growing log.** Rotation is what keeps it
  short enough to be read in full every time.
- **Dead Ends — Do Not Retry is the section that earns the most.** A fresh session
  will otherwise re-attempt the approach that was already eliminated.
- **The `/tmp` rescue runs before the handoff is written**, so the handoff can cite
  a path that still exists rather than one that will not survive a reboot.
- **Git state is taken from git.** If a command did not print it, it does not go in
  the file.
- **It commits and stops.** Pushing is left to the operator.

## License

MIT — see [LICENSE](LICENSE).
