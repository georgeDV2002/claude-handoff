# handoff + pickup

Two Claude Code skills that work as a pair. `/handoff` closes out a working
session: it writes a `SESSION.md` the next session reads to resume, rotates the
previous one into an archive, copies anything worth keeping out of `/tmp`, and
commits — in each repository the session touched. `/pickup` opens the next
session: it loads that file, checks it against git, and waits for you.

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

Before anything is written as pending, it is checked. Every item from the previous
handoff and from this session is compared against the conversation, the commits
and the repository. Finished work moves to Completed. What remains is tagged
`[open]`, `[if <condition>]` for things that only matter once something happens,
or `[unsure]` with the one check that would settle it. An unsure item stays
unsure rather than being rounded up to pending or down to done.

Each pending item also carries an ID and the date it was first written down —
`[open] T12 (since 2026-02-26)` — and keeps both from one handoff to the next.
Every item the previous handoff listed ends up either still pending or under
Closed Since Last Handoff, with what closed it: a commit, a decision, or a reason
it was dropped. Nothing leaves the list without a line saying why, so the next
session can tell an item that has waited three weeks from one raised yesterday,
and one that was finished from one that was forgotten.

A session that stops halfway through something writes an In Progress entry: what
is done, exactly where it stopped, what is still running and how to check it, and
the steps left in order. The next step is then resuming it, so the position is not
traded for an older item on the list. Items that are still open but parked go
under Backlog, which the next session reports as a count rather than reading out
in full. An item moves between the two only when someone moves it, never because
of its age.

It is deliberately short, because a handoff the next session cannot afford to read
in full is not doing its job, and session history is kept in a separate file for
that reason.

The file opens by addressing the next session directly, so resuming works whether
or not that session has this skill installed:

> **New session:** read this file whole and keep it in context — do not recite
> it back. Only `[open]` items are pending; `[if …]` items are dormant and
> `[unsure]` items must be shown as unsure, not as pending. Items keep their ID
> (`T12`) across handoffs; one under Closed Since Last Handoff is finished.
> If In Progress is filled, the session stopped mid-process: resume from its
> "Stopped at" line, after checking anything listed as Running. Backlog items
> are open but parked — give them as a count unless asked.
> Check `git log --since=<Updated>` for commits newer than this file, then WAIT
> for the operator before acting.

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

### 3. The next session starts loaded, not lectured

`/pickup` is the other half. It reads `SESSION.md` in full, checks it against
git (every commit made after the handoff, in order and dated, a different branch,
uncommitted work), and re-checks every pending item by its ID, since something may
have been finished after the handoff was written. It reads only the few files the
next step names, and never opens the archive except to search it for a specific
question.

Then it prints one line — `Loaded <repo> @ <branch> — handoff from <date>` — and
waits. Everything it read is in context, so you ask what you need ("what's
pending?", "why did we drop the 10k batch?") instead of reading a report. Asked
what is pending, it lists open items first, then unsure ones marked as unsure,
each with its ID and, if it has waited a while, its age, and leaves finished work
out. Anything it finds in the archive is read as history, dated by the entry it
came from, never as current status.

## In practice

I have run this at the end of most working sessions since the end of June, across
about twenty repositories. The longest of those archives now holds 28 entries, one
for each previous session on that project, each recording what was decided and what
had already been ruled out.

It has made a real difference to how much carries over from one session to the next,
which is why I am publishing it rather than leaving it in my own setup.

## Install

```bash
git clone https://github.com/georgeDV2002/claude-handoff
mkdir -p ~/.claude/skills/handoff ~/.claude/skills/pickup
cp claude-handoff/claude-code/handoff/SKILL.md ~/.claude/skills/handoff/
cp claude-handoff/claude-code/pickup/SKILL.md ~/.claude/skills/pickup/
```

Run `/handoff` as a session winds down, rather than once the context window has
already forced the issue. Run `/pickup` as the first thing in the next one.

## What it writes

**`SESSION.md`** — the current session only, in the repository root. Thirteen
sections: Bigger Picture, Objective, Git State, Completed This Session, Decisions
Made, Dead Ends — Do Not Retry, Rescued Artifacts, Context That Would Be Lost,
Closed Since Last Handoff, In Progress, Next Step, Open Questions, Backlog. Pending items carry the
`[open]` / `[if …]` / `[unsure]` tags, an ID and a since-date, or the sections say
"Nothing outstanding" and "none" outright.

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
- **Finished work is checked, not remembered.** A handoff that lists done work as
  pending sends the next session back over it.
- **Unsure is a state of its own**, with the check that would settle it.
- **Items keep their ID across sessions, and closing one is written down.** A
  list rewritten from scratch each time cannot show how long something has
  waited, or tell finished from forgotten.
- **It commits and stops.** Pushing is left to the operator. Only the two handoff
  files are committed, even when other work is already staged; a blocked commit
  is reported as blocked.
- **`/pickup` prints one line.** The handoff is for loading into context, not for
  reading back to the person who lived through the session.

## License

MIT — see [LICENSE](LICENSE).
