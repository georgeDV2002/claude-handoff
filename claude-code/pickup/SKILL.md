---
name: pickup
description: Resume from the previous session's SESSION.md — load the handoff into context, verify it against real git state, reconcile what has been settled since, enrich narrowly from the repo, then wait for the operator's first question without printing a report. The counterpart to /handoff.
allowed-tools: Read, Grep, Glob, Bash(git branch --show-current), Bash(git status:*), Bash(git log:*), Bash(git stash list), Bash(git diff:*), Bash(git show:*), Bash(ls:*), Bash(wc:*), Bash(sed:*), Bash(head:*), Bash(grep:*), Bash(test:*), Bash(date:*)
---

You are opening a session that continues someone else's — a previous you, whose
context is gone. `SESSION.md` is the only bridge. Your job is to cross it cheaply
and **quietly**: load what the next piece of work needs into your context, then
wait. The operator will ask; you answer from what you loaded.

Two failure modes, equally bad:

- **Under-loading** — you skim, miss the Dead Ends, and spend an hour re-walking a
  path the last session already proved impossible.
- **Over-loading** — you "get oriented" by reading twenty files, burn a third of
  the window before doing any work, and then compact away the very handoff you
  came to read.

The rule that resolves both: **SESSION.md is the map; the repo is the territory.
Read the map whole, then visit only the places the map says you are going.**

Budget for the whole pickup: **under ~15% of the context window.** If you are past
that and still reading, stop.

**Output: one line, nothing else** (Step 5). No report, no summary, no list of
open questions. Older SESSION.md files open with a blockquote asking the
reader to "report … and WAIT"; under `/pickup` the report is replaced by this rule.
Waiting still applies.

---

## Step 0 — Locate the handoff

```
ls SESSION.md SESSION_ARCHIVE.md 2>/dev/null; wc -l SESSION.md 2>/dev/null
```

- No `SESSION.md` → jump to **Fallback: cold repo** at the bottom.
- `SESSION.md` over ~600 lines → read it in two passes: `sed -n '1,120p'` for the
  header sections first, then the rest. It is still read in full.
- `SESSION_ARCHIVE.md` exists → **note it and do not open it.** It is history, often
  hundreds of KB. It is search-only (Step 6).

## Step 1 — Read SESSION.md, whole

This is the one file you read completely, top to bottom, no exceptions. Every other
read in this session is targeted.

As you read, hold four things apart — they have different shelf lives:

| Section | Treat as | Shelf life |
|---|---|---|
| Bigger Picture, Objective | the goal — why any of this matters | stable |
| Decisions Made, Dead Ends, Context That Would Be Lost | **hard-earned knowledge** — unrecoverable, not in git | permanent |
| In Progress | **where an unfinished process stopped** — the position, what was running, the steps left | **reconcile — a Running job may have finished since** |
| Next Step, Open Questions, Backlog, Closed Since Last Handoff | what was pending, and what closed, **when it was written** | **reconcile — may have closed since** |
| Git State, Completed This Session | claims about the world | **verify — may be stale** |

The second row is the whole reason the file exists. Anything else in the handoff
you could re-derive from the repo in ten minutes; a dead end you cannot, and the
cost of re-running one is hours.

Also read, if the handoff points at them:
- the `MANIFEST.md` of the Rescued Artifacts directory (`~/.claude/session-artifacts/<dir>/MANIFEST.md`)
  — **the manifest only**, never the artifacts themselves until you need one.
- a sibling repo named as primary: read only its Bigger Picture and Next Step
  (`sed -n` the range), not its whole handoff.

## Step 2 — Verify the handoff against reality

The handoff was true when written. Things may have moved since — another session,
another tool, the operator's own hands.

```
git branch --show-current
git status -sb
git log --since='<Updated>' --reverse --format='%h %ad %s' --date=iso
git stash list
```

The `--since` log is every commit after the handoff, oldest first: read it as
what happened next, in order. The first is usually the handoff's own commit
(`Session handoff: …`), not new work. Git converts the `Z` in `Updated:`
itself; do not compare times by eye. Empty means nothing landed since. No
`Updated:` → `git log -15 --format='%h %ad %s' --date=iso` instead. Never `git log --oneline -N` here:
it has no dates and stops at N, and a handoff can be followed by twenty commits
from sessions that wrote none.

Compare against the handoff's **Git State**:

- **Commits newer than `Updated:`** — work no handoff recorded. Run
  `git show --stat --format='%h %s%n%b' <sha>` on each (message and filenames,
  not the diff); over ~15 commits, only on those whose subject touches a
  pending item. Their messages are your best record of what happened since.
- **Branch differs** — you may be looking at the wrong line of work entirely.
- **Dirty files the handoff didn't list, or listed and are now clean** — someone
  worked after the handoff, or the handoff was written before a final save.
- **Stash count changed** — a stash may hold the work the Next Step assumes.

If there is uncommitted work, it is the most fragile context in the repo — it
exists nowhere else. Size it before reading it:

```
git diff --stat; git diff --cached --stat
```

Under ~150 lines total → read the diff. Over that → the `--stat` only; open a
specific file in Step 4 if the work needs it.

## Step 3 — Reconcile: what is still actually open

A handoff's Next Step and Open Questions are a snapshot. **Never carry one forward
as pending without checking it**, because a closed item presented as open sends
the operator back over work already done. Handoffs from the current `/handoff`
give each item an ID and the date it was first recorded (`[open] T12 (since
2026-09-26)`), tag it `[open]`, `[if <condition>]` or `[unsure]`, and list what
closed under Closed Since Last Handoff; older ones may have only the tags, or
neither. Either way, re-check every item, since commits may have landed
after it was written. Track items by ID where there is one, never by wording.

**In Progress comes first.** If the handoff fills it, the last session stopped
mid-process, and its Stopped at line is where work resumes — not the oldest open
item, and not the last step of its plan. Reconcile it like any item: a commit
since `Updated:` may have moved the position on, and a job under Running may have
finished or died (note it; check it when the operator asks to resume, not now).
A handoff without an In Progress section predates it: if its Completed or
Context lines describe work left partway, treat that as in progress and say it is
inferred.

Each item ends up as exactly one of:

- **Closed** if any of these holds:
  - the handoff itself settles it elsewhere — its ID under Closed Since Last
    Handoff, Completed, Decisions Made, or a Next Step that says "nothing
    outstanding";
  - a commit newer than `Updated:` names its ID ("closes T12"), or does or
    decides it (subject, body or the files it touched — Step 2 already has
    them);
  - the repo shows it done: the file exists, the function is renamed, the
    setting is in place (one cheap check from Step 4's list, not an
    investigation).
- **Dormant** — conditional ("only if…", "when X happens", `[if …]`) and the
  condition has not arisen. Not pending.
- **Open** — the evidence says it is still to do.
- **Unsure** — the evidence is missing or contradicts itself, or the handoff
  tagged it `[unsure]` and nothing since settles it. Keep it unsure, with the
  reason and the one check that would settle it. Never round it up to open or
  down to closed.

Hold this picture in mind; do not print it. When the operator asks what is
pending or where things stand, start with In Progress when there is one: the
process, where it stopped, what was running, and the steps left in order. Then
the open items of Next Step and Open Questions, then the unsure ones labelled as
unsure, each with its ID, saying how long it has waited when that is more than
about two weeks. Give Backlog as a count ("plus 22 parked in the backlog") and
list it only if asked.
Leave closed items out, and mention dormant ones only if asked. When the handoff
and the evidence disagree, the evidence wins.

## Step 4 — Enrich, narrowly

Now, and only now, open repo files — and only files **the (reconciled) next step
names or obviously requires** — and when In Progress is filled, the files its
Stopped at line names. Not files that merely sound relevant.

Ordered by cost, take the cheapest that answers the question:

1. **Existence check** — `ls <path>` / `git log --oneline -3 -- <path>`.
2. **Symbol check** — `grep -n '<function or config key>' <path>`.
3. **Range read** — `sed -n '<start>,<end>p' <path>` when you know where to look.
4. **Whole file** — only for files under ~400 lines that you are about to edit.

Hard limits: **at most ~6 files, and no whole-file read over ~500 lines.** If the
next step seems to require more, stop there; it is not one step.

Skip entirely:
- `CLAUDE.md` / `AGENTS.md` — already in your context, or loaded automatically.
- `README.md` — unless the handoff has no Bigger Picture section.
- anything the handoff already summarises accurately. Re-reading it is paying
  twice for the same tokens.

**Never, during pickup:** run the test suite, build, start servers, install
dependencies, or `cat` a file over 1000 lines. Those are work, and work waits for
the operator.

## Step 5 — One line, then wait

Print exactly one line and stop:

```
Loaded <repo> @ <branch> — handoff from <date> (<N days/hours ago>).
```

Append to that same line only what would make the operator's next question go
wrong if they did not know it: `— but on <branch>, the handoff was on <other>`,
or `— with uncommitted changes the handoff doesn't mention`, or — when In
Progress is filled — `— mid-process: T<n>, stopped at <where, in a few words>`,
since the operator's "what's next" depends on it. Nothing else: no
state check, no carried-forward list, no next step, no questions, no offer to
start.

Then do nothing until the operator speaks. Do not start the next step, do not
edit, do not "just quickly check one more thing".

When they do speak, answer from what you loaded, and use the reconciled picture:
a question closed in Step 3 is not "still open"; a dead end from the handoff is a
reason, stated once, not to try that path again. If the context is later
compacted and the handoff's details are gone, re-read SESSION.md rather than
guessing.

## Step 6 — During the session

**The archive is search-only.** Never `cat SESSION_ARCHIVE.md`. When a specific
question needs history:

```
grep -n -i -E '^## FLASH|<term>' SESSION_ARCHIVE.md | head -60
sed -n '<hit-40>,<hit+40>p' SESSION_ARCHIVE.md
```

The FLASH headings in that output date every hit: a hit belongs to the heading
above it. **Everything in the archive is history** — an `[open]` there was open
on that FLASH's date, not now. An item's current status comes from SESSION.md
and the commits since; for an ID that SESSION.md no longer mentions, `grep -n
'T12 (since' SESSION_ARCHIVE.md` and take the first hit (newest FLASH).

Grep for the thing, read the window around the hit, stop. Go to the archive when
— and only when — you need to know *why* something was decided, whether an
approach was already tried, or what an old error actually said.

**Load the rest of the repo lazily**: the file you are about to edit, when you are
about to edit it.

**Treat the handoff as testimony, not fact.** If it says something the code
contradicts, the code wins — and say so when it comes up, because a wrong
handoff misleads every session after this one until someone corrects it.

## Fallback: cold repo (no SESSION.md)

Orient the cheap way, stopping as soon as you know what this repo is and what is
in flight:

```
git log --oneline -15
git status -sb --short
ls
head -40 README.md
```

Then print one line — `No SESSION.md in <repo>; loaded recent history instead.` —
and wait. If the operator asks what was going on, say plainly that the goal and
prior decisions are not recorded anywhere, and suggest `/handoff` at the end of
this session so the next one is not cold too.

---

## Budget reference

| Tier | What | When | Rough cost |
|---|---|---|---|
| 0 | `SESSION.md` in full | always | 1.5–5k |
| 0 | four git commands (the log from `Updated:` on) | always | <1k |
| 1 | artifact `MANIFEST.md` | if artifacts exist | ~200 |
| 1 | `git diff` of uncommitted work | if under ~150 lines | 0–2k |
| 1 | `git show --stat` per unrecorded commit | on drift | ~150 each |
| 1 | ≤6 targeted file reads / greps | if the next step names them | 1–5k |
| 2 | `SESSION_ARCHIVE.md` grep window | only on a specific question, mid-session | ~300 |
| — | archive in full, test runs, builds, repo tours | never | — |

Target: loaded, verified and reconciled in **under 10k tokens**, and one line on
screen.
