# Session Handoff
Updated: 2026-03-14T22:41:09Z
Previous handoff: 2026-03-11T19:02:44Z

> **New session:** read this file whole and keep it in context — do not recite
> it back. Only `[open]` items are pending; `[if …]` items are dormant and
> `[unsure]` items must be shown as unsure, not as pending. Items keep their ID
> (`T12`) across handoffs; one under Closed Since Last Handoff is finished.
> If In Progress is filled, the session stopped mid-process: resume from its
> "Stopped at" line, after checking anything listed as Running. Backlog items
> are open but parked — give them as a count unless asked.
> Check `git log --since=<Updated>` for commits newer than this file, then WAIT
> for the operator before acting. History lives in SESSION_ARCHIVE.md (FLASH
> sections, newest first) — search it, never read it; nothing there is current.

## Bigger Picture
Replacing the nightly CSV export with a streaming ingest so the dashboard stops
being a day behind. Ingest path is done; the cutover and the backfill are not.

## Objective
Get the new ingest writing to the staging table and prove it matches the CSV
export row-for-row for one full day.

## Git State
- Branch: streaming-ingest (3 unpushed)
- Dirty files:
  M ingest/consumer.py
  ?? scratch/compare.sql — scratch, do not commit
- Stashes: none
- Recent commits:
  9f2c1ab Consumer commits offsets only after a successful write
  4e77d30 Dead-letter topic for rows that fail validation
  1b09e55 Staging table + indexes

## Completed This Session
- `ingest/consumer.py` now commits Kafka offsets after the DB write, not before
- Added the dead-letter path for validation failures (`ingest/dlq.py`)
- Verified 2026-03-12 matches the CSV export exactly: 1,284,301 rows both sides

## Decisions Made
- Commit offsets after the write: at-least-once beats silently dropping rows
- Staging table stays separate until the backfill is verified: one-command rollback

## Dead Ends — Do Not Retry
- Kafka transactions for exactly-once: the sink isn't transactional, so it bought
  nothing and cost ~40% throughput
- Batching at 10k rows: the p99 latency spike broke the dashboard's 60s freshness
  target. 1k is the ceiling that holds.

## Rescued Artifacts
`~/.claude/session-artifacts/ingest-2026-03-14T22-30Z/`

- `compare.sql` — the row-for-row comparison between staging and the CSV export.
  Took a while to get the timezone handling right; reuse it for the backfill check.
- `throughput.csv` — the batch-size benchmark behind the 1k decision.

Not rescued: the local `.env` with the staging DB password — regenerate from the
vault entry `staging/ingest`.

## Context That Would Be Lost
The consumer silently stalls if the topic has fewer partitions than the consumer
count — it doesn't error, it just sits there. We have 6 partitions, so don't
scale past 6 without repartitioning first.

The CSV export rounds timestamps to the second; the stream doesn't. The
comparison only matches once you truncate both sides.

## Closed Since Last Handoff
- T2 (since 2026-03-09) Rows that fail validation crash the consumer — done:
  dead-letter topic, 4e77d30
- T4 (since 2026-03-11) Prove one full day matches the CSV export — done this
  session: 2026-03-12, 1,284,301 rows both sides
- T5 (since 2026-03-11) Try Kafka transactions for exactly-once — dropped: the
  sink isn't transactional (see Dead Ends)

## In Progress
none — the cutover check finished; the backfill has not started.

## Next Step
[open] T8 (since 2026-03-14) Run the backfill for 2026-03-01 to 2026-03-13
against staging, then compare with `compare.sql` one day at a time.

## Open Questions
- [open] T3 (since 2026-03-09) Cut over during business hours or wait for the
  weekend window?
- [open] T6 (since 2026-03-11) Keep the CSV export running in parallel for a
  week as a fallback, or cut clean?
- [if we scale past 6 consumers] T7 (since 2026-03-14) Repartition the topic
  first (see Context).
- [unsure] T9 (since 2026-03-14) Whether the DLQ consumer is deployed to
  staging — its deploy was started but never confirmed; `kubectl get deploy
  dlq-consumer -n staging` settles it.

## Backlog
- [open] T10 (since 2026-03-14) Move the DLQ alert from email to the on-call
  pager — not needed for the cutover.
