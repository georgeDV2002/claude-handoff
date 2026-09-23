---
name: handoff
description: Capture full session state to SESSION.md (latest session only), rescue useful /tmp artifacts to persistent storage, rotate the previous handoff into SESSION_ARCHIVE.md, and commit — in every repo touched. Use when closing, handing off, pausing, or preserving a coding or research session for a fresh context.
allowed-tools: Read, Write, Edit, Grep, Bash(grep:*), Bash(git show:*), Bash(git status:*), Bash(git branch --show-current), Bash(git log:*), Bash(git stash list), Bash(git add SESSION.md SESSION_ARCHIVE.md), Bash(git commit:*), Bash(date:*), Bash(ls:*), Bash(du:*), Bash(find:*), Bash(mkdir -p:*), Bash(cp:*)
---

You are closing this session. Before context is lost, capture everything the next session needs to continue without asking questions — especially the hard-earned knowledge that lives only in this conversation: mistakes made, assumptions confirmed or broken, approaches that worked and that failed. SESSION.md is what a fresh chat reads to resume with a SMALL context window; it must be self-sufficient on its own, without the archive.

## Step 0 — Scope the repos

List every repo this session did meaningful work in (the cwd plus siblings, e.g. `~/my-other-app`). Run Steps 1 and 3–5 in EACH of them; Step 2 runs ONCE for the whole session. Pick one as the **primary** (where the session's main narrative lives); a sibling's SESSION.md may be shorter and link to the primary for the shared story, but it must stand alone for its own repo's state. A repo that was merely read, not changed, does not need a handoff — mention it in the primary's Context section instead.

## Step 1 — Gather state (per repo)

Run all of these:
- `git branch --show-current`
- `git status -sb` (ahead/behind origin)
- `git status --short`
- `git stash list`
- `git log --oneline -5`
- `date -u +"%Y-%m-%dT%H:%M:%SZ"` (once is enough)

Use that output as evidence. Do not invent ahead/behind counts, dirty files, stashes, commit subjects, or branch names — if a command did not print it, it does not go in the handoff.

## Step 2 — Rescue ephemeral files (once per session)

**`/tmp` is wiped when WSL restarts.** Anything this session produced there — the
scratchpad directory, ad-hoc scripts, extracted datasets, query output, generated
reports, notes — is gone the moment the VM stops, and SESSION.md would then point
at paths that no longer exist. Copy what still has value somewhere persistent
BEFORE writing SESSION.md, so the file can reference the real, surviving location.

### 2a — Inventory

Look in both places:
- the session scratchpad directory (the path given in your environment, under
  `/tmp/claude-*/…/scratchpad`)
- any other `/tmp` path this session wrote to or read from — scan the conversation
  for them; also `ls -lat /tmp` and `find /tmp -maxdepth 2 -newermt '-1 day' -type f`
  to catch files you forgot you made.

Then `du -sh` the candidates so you know what you're about to copy.

### 2b — Triage

Keep a file if recreating it would cost real work or is impossible now:
- scripts, SQL, notebooks, one-off tooling written this session
- extracted / cleaned / joined datasets, query results, API dumps
- generated reports, diffs, logs that document a finding
- notes, plans, or drafts not yet in a repo

Skip:
- anything already committed in a repo, or trivially regenerable in seconds
- caches, venvs, `node_modules`, build output, downloaded packages
- very large blobs (> ~200 MB) — record the path and how to regenerate instead of copying
- **secrets**: credentials, tokens, `.env` files, private keys, exported customer
  data. Do not copy these into persistent storage. Note in SESSION.md that they
  existed and how to obtain them again.

If a file's value is unclear, keep it — the cost is a few KB.

### 2c — Copy

```
mkdir -p ~/.claude/session-artifacts/<repo-or-topic>-<UTC timestamp, e.g. 2026-08-02T14-30Z>
cp -a <each kept file> ~/.claude/session-artifacts/<that dir>/
```

Use one directory per session (per primary repo/topic). Preserve original
filenames; if two rescued files collide, prefix with their source subdirectory.
Never copy into `/tmp`, and never delete the originals.

### 2d — Write MANIFEST.md

In that directory, write `MANIFEST.md`:

```
# Rescued artifacts — <repo/topic>, <ISO datetime>
Original location: <the /tmp path(s) these came from>

- `<filename>` — <what it is, why it was kept, how it was produced>
- ...

Not rescued: <files deliberately skipped and why — secrets, too large, regenerable>
```

### 2e — Nothing to rescue

If `/tmp` holds nothing worth keeping, skip the copy and say so explicitly in
SESSION.md's Rescued Artifacts section ("none — nothing in /tmp worth keeping").
Silence there is ambiguous; the next session can't tell it from an oversight.

## Step 3 — Rotate the previous handoff into the archive (per repo)

SESSION.md holds ONLY the latest session. Before writing the new one:

1. If `SESSION.md` exists, move its entire current handoff into `SESSION_ARCHIVE.md` as a new section: `## FLASH <N> — <short title> (<its Updated date>)`, where N = highest FLASH number already in the archive + 1 (start at 1 if none). Demote the moved handoff's `##` headings to `###` so the archive stays navigable.
2. Insert the new FLASH directly under the archive's intro paragraph — **newest first**. Never edit or delete existing archive entries.
3. If `SESSION_ARCHIVE.md` doesn't exist yet, create it with this header:

```
# Session Archive — <repo name>

Previous session handoffs, newest first (highest FLASH = most recent). Do NOT
load this file at session start — SESSION.md carries the current state. Search
here only when a specific question needs history.
```

If the previous handoff carries no `Updated:` value, write `date unknown` in the new FLASH heading rather than guessing a date.

## Step 3½ — Check what is actually finished (per repo)

Do this BEFORE writing SESSION.md. A handoff that lists finished work as pending
sends the next session back over it, so **nothing goes under Next Step or Open
Questions without being checked.**

1. Collect every candidate: the previous handoff's Next Step and Open Questions
   (you just rotated it — read them from the archive's new FLASH), plus anything
   this session left undone, deferred or asked about.
2. Check each one against evidence, cheapest first:
   - this conversation — was it done, answered or decided here?
   - `git log` since the previous handoff's `Updated:` — does a commit do it?
   - the repo — one `ls`, `grep` or `git log -- <path>` that shows it done.
3. Put it in exactly one place:
   - **Done** → Completed This Session (if done now) or drop it (if done
     earlier). Never under Next Step or Open Questions.
   - **Still pending, and you are sure** → tag `[open]`.
   - **Waiting on a condition that has not happened** ("only if a third app
     needs it") → tag `[if <condition>]`. It is not pending work.
   - **Cannot tell** — evidence is missing or contradicts itself → tag
     `[unsure]` and say why and what one check would settle it. Never round an
     unsure item up to open or down to done.

If nothing is left, Next Step says `Nothing outstanding.` and Open Questions says
`none` — explicitly, so the next session cannot mistake silence for an oversight.

## Step 4 — Write SESSION.md (per repo)

Write `SESSION.md` in the repo root with the following structure. Be specific and terse. No filler sentences.

```
# Session Handoff
Updated: <ISO datetime from date command>

> **New session:** read this file whole and keep it in context — do not recite
> it back. Only `[open]` items are pending; `[if …]` items are dormant and
> `[unsure]` items must be shown as unsure, not as pending. Finished work is
> never listed below as pending. Check git for commits newer than `Updated:`,
> then WAIT for the operator before acting. History lives in
> SESSION_ARCHIVE.md (FLASH sections, newest first) — search it, never read it.

## Bigger Picture
<2–3 lines: the overarching goal this work serves, and where the project
stands on the road to it. Stable across sessions — update, don't rewrite.>

## Objective
<One sentence: what this session was trying to accomplish>

## Git State
- Branch: <branch> (<in sync with origin | N unpushed | N behind>)
- Dirty files:
  <list each file with its status flag, e.g. M src/foo.py — or "none">
- Stashes: <count or "none">
- Recent commits:
  <paste git log --oneline -5 output verbatim>

## Completed This Session
- <concrete action taken — be specific about files, functions, decisions>
- ...

## Decisions Made
- <decision>: <one-line rationale>
- ...

## Dead Ends — Do Not Retry
- <approach or idea>: <why it failed or was ruled out>
- ...

## Rescued Artifacts
<Absolute path of the ~/.claude/session-artifacts/<dir> from Step 2, then one
line per file: what it is and what the next session would use it for. Write
"none — nothing in /tmp worth keeping" if that's the case. Also note anything
deliberately NOT rescued (secrets, oversized files) and how to regenerate it.
Never reference a /tmp path as if it will still exist — it won't.>

## Context That Would Be Lost
<Anything important that lives only in this conversation — error messages
seen, assumptions confirmed, things tried interactively that aren't in git>

## Next Step
<[open] Single, concrete, immediately actionable instruction for the next
session, checked in Step 3½ as not yet done. Or: "Nothing outstanding.">

## Open Questions
- [open] <unresolved item>
- [if <condition>] <item that only matters once the condition arises>
- [unsure] <item> — <why it is unclear; the one check that would settle it>
<Or: "none". Only items that passed Step 3½; nothing finished appears here.>
```

## Step 5 — Commit (per repo)

```
git add SESSION.md SESSION_ARCHIVE.md
git commit -m "Session handoff: <one line, what this session shipped>"
```

Commit only — do not push unless the operator asked. If the working tree has other dirty files, leave them alone: stage exactly these two files.

If unrelated changes were already staged before the handoff began, do not disturb them — commit only the two handoff paths:

```
git commit --only SESSION.md SESSION_ARCHIVE.md -m "Session handoff: <one line>"
```

If the commit cannot be created at all, keep the finished handoff files on disk and report the exact blocker. Do not alter Git configuration, unstage the operator's work, or touch unrelated state to force it through.

## Step 6 — Confirm

Print exactly one line per repo:
"Handoff saved → <repo>/SESSION.md | Branch: <branch> | Committed <short sha> (<N unpushed>) | Next: <next step, max 10 words, or "nothing outstanding">"

If the commit failed, replace the `Committed …` segment with a truthful `Commit blocked: <reason>` segment rather than reporting a commit that does not exist.

Then one final line for the session:
"Rescued <N> file(s) → ~/.claude/session-artifacts/<dir>" — or "Rescued: none".

## Note on resume

The resume protocol is embedded in SESSION.md itself (the blockquote at the top), so any future session that reads the file follows it without ever having seen this command: load the file without reciting it, treat only `[open]` items as pending and `[unsure]` ones as unsure, then wait — and touch SESSION_ARCHIVE.md only when a specific question requires history. `pickup` does the same with a git check and a reconcile of its own.
