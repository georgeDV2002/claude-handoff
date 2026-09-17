# handoff

A Claude Code skill that closes a coding session properly: it writes a
self-sufficient `SESSION.md`, rescues the files you left in `/tmp`, archives the
previous handoff, and commits — in every repo the session touched.

The problem it solves: when a session ends, the expensive part isn't the code —
that's in git. It's everything that only ever existed in the conversation. Which
approach you already tried and abandoned. Why the obvious fix doesn't work. The
error message that explained everything. The script you wrote in `/tmp` that took
an hour and will be deleted on the next reboot.

`/handoff` writes that down while it still exists.

## Install

```bash
git clone https://github.com/<you>/claude-handoff
mkdir -p ~/.claude/skills/handoff
cp claude-handoff/claude-code/handoff/SKILL.md ~/.claude/skills/handoff/
```

Then run `/handoff` at the end of a session.

## What it produces

**`SESSION.md`** — the current session only, in the repo root. Ten fixed sections:
Bigger Picture, Objective, Git State, Completed This Session, Decisions Made,
**Dead Ends — Do Not Retry**, Rescued Artifacts, Context That Would Be Lost,
Next Step, Open Questions. See [`examples/SESSION.md`](examples/SESSION.md).

It opens with a blockquote addressed to the next session, so resuming works
without the reader ever having seen this skill:

> **New session:** read only this file, then report to the operator (1) where we
> left off, (2) pending decisions and open questions — and WAIT for confirmation
> before acting.

**`SESSION_ARCHIVE.md`** — every previous handoff, newest first, numbered
`FLASH 1..N`. Explicitly marked *not* to be loaded at session start; it's there to
be searched when a specific question needs history.

**`~/.claude/session-artifacts/<repo>-<timestamp>/`** — whatever was worth keeping
from `/tmp`, plus a `MANIFEST.md` saying what each file is and why it was kept.
Secrets are deliberately never copied here.

## Design notes

A few choices that aren't obvious:

- **`SESSION.md` is capped at one session.** A file that accumulates is a file the
  next session can't afford to read. Rotation into the archive is what keeps it
  small enough to be read in full, every time.
- **"Dead Ends — Do Not Retry" is the highest-value section.** It's the one thing
  a fresh context will otherwise cheerfully redo.
- **The rescue step runs before `SESSION.md` is written**, so the handoff can cite
  the surviving path rather than a `/tmp` path that won't exist tomorrow.
- **It never pushes.** Committing is enough; pushing is the operator's call.
- **It never invents git output.** If a command didn't print it, it doesn't go in
  the file.

## License

MIT — see [LICENSE](LICENSE).
