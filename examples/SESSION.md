# Session Handoff
Updated: 2026-03-14T22:41:09Z

> **New session:** read only this file, then report to the operator (1) where
> we left off, (2) pending decisions and open questions — and WAIT for
> confirmation before acting. Session history lives in SESSION_ARCHIVE.md
> (FLASH sections, newest first) — search it only when a question needs it.

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

## Next Step
Run the backfill for 2026-03-01 to 2026-03-13 against staging, then compare with
`compare.sql` one day at a time.

## Open Questions
- Cut over during business hours or wait for the weekend window?
- Keep the CSV export running in parallel for a week as a fallback, or cut clean?
