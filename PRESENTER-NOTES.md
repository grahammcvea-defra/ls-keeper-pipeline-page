# Presenter notes — anticipated Q&A

For presenting the animated explainer to a mixed Defra audience (end users +
engineers). Answers are grounded in `origin/develop @ 4c6761e` — verify before
presenting if the pipeline has moved on. **Flagged answers are ones where honesty
beats confidence.**

## Framing (everyone)

**What am I looking at?**
The journey of supplier data files from their S3 bucket to the SQLite read model
consumers query. Every card/folder maps to a real stage or directory — only
timings and counts are illustrative.

**Where does the data come from?**
Two encrypted families in supplier-owned S3: 12 `LITP_*` extracts (`litprd/` —
SAM, CTS holdings, AMLS2, AMES) and 6 CTS CADS tables (`cads/cts/`, header/data/
trailer format, bulk baseline + daily deltas). 18 datasets, all delta mode.

**How often does it run?**
Scheduled (daily, cron is config not code) plus on-demand via
`POST /api/etl/imports` or the `etl` CLI. One run at a time — distributed lock.

**Why so many steps?**
Every stage materialises an inspectable artefact (`raw/` → `normalised/` →
`optimised/` → `snapshots/` → `staging/` → `views/`). That makes failures
diagnosable, reruns cheap (existing artefacts are skipped), and keeps each stage
single-purpose.

## End users / stakeholders

**How fresh is the data?**
As fresh as the last successful run. Snapshot and export filenames carry the
newest source timestamp — `krds-db_20260821070003.sqlite` is self-describing.

**What happens when it fails?**
The run records where it stopped and why — per-stage/dataset progress, timings
and errors via `GET /api/etl/imports/{id}`. Stages skip existing artefacts, so a
rerun resumes rather than restarts.

**What if a supplier sends a bad file?**
Decrypt/validation failures are recorded as structured errors — nothing passes
silently. File ordering comes only from the filename timestamp: a missing or
duplicate timestamp fails the import rather than being guessed.

**Is personal data protected?**
Yes — the read model holds keeper names/emails/phones, so its presigned download
URL has a capped expiry (much shorter than intermediate artefacts). A leaked URL
has a short window, not permanent access.

**How do people get the data?**
`GET /api/etl/staging/sqlite/latest` → presigned URL for the newest read model.
Engineers can also pull the DuckDB staging DB or per-dataset parquet snapshots.

**Where does the cleanse report fit?**
**It doesn't — deliberately.** Cleanse still reads MongoDB fed by the *legacy*
import pipeline, not this ETL output. It's also dormant: scheduled job disabled,
new rule engine merged but toggled off (`UseRuleBasedEngine = false`). This page
shows the new world; the cleanse lives in the old one.

## Engineers

**Why DuckDB in the middle?**
Snapshots are Parquet; DuckDB reads Parquet natively and runs the relational
transform across all 18 tables in one file. The staging DB is itself a
downloadable artefact — the SQLite export is just "run versioned SQL, emit a
portable file." It's a staging tier, not a serving tier.

**Why SQLite as output, not an API?**
Single file: trivially distributable, versioned by timestamp, no serving infra.
Consumers download and query locally — suits read-model-with-batch-freshness.

**How does a rerun avoid redoing work?**
Existing artefacts are skipped; snapshot names carry the source timestamp
(nothing newer = reuse); exports are fingerprinted by transform version
(changed SQL rebuilds rather than skips).

**What if the CTS bulk baseline is republished?**
The snapshot name embeds a bulk-filename hash. Changed hash → rebuild from all
bulk files + only deltas newer than newest bulk. Older deltas aren't replayed —
last-writer-wins would let them revert baseline rows.

**How parallel is it really?**
`decrypt`/`normalise`/`optimise` map files concurrently (`ParallelMapStage`):
IO budget for decrypt, CPU budget for the other two. Output order preserved.
*The animation's burst is illustrative, not a measured concurrency level.*

**How do I rebuild from scratch?**
`POST /api/etl/imports?rebuild=true` clears all six stage folders in order first
(`EtlStageCascade`). Also `etl purge` and `DELETE /api/etl/storage`. Narrower:
`?dataset=sam_cph_holdings` runs one dataset.

**Schema drift / changed transform?**
Optimise handles per-dataset projection/types. The transform fingerprint is
stored on each export — a changed SQL script rebuilds rather than reuses.
`SchemaVersion` covers semantic changes without script changes.

**Two runs at once?**
Second gets `409 Conflict` — distributed lock held (and renewed) for the run.

**Observability?**
Status document per import: current stage, per-stage counts/timings, per-dataset
progress, artefact paths, row counts, merge stats, structured errors. Plus
`GET /api/etl/storage/objects` for a bucket listing.

## Questions that could sting

**Is this what's in production?**
This depicts `develop` — `main` is behind (no optimise stage). Say "current
develop state, pinned @ 4c6761e" — not "production."

**Are those real numbers?**
No — counts and timings are illustrative. Real numbers live in the status API.
Don't let anyone quote the animated row count as fact.

**Does the cleanse use this output?**
No — legacy MongoDB ingestion. "Will it?" is a roadmap question, not a fact;
reasonable inference is migrate or retire.

**Why not event-driven/streaming?**
The supply is files dropped in a bucket — batch is the shape of the input.
Every stage is incremental anyway; only new deltas get processed.

**What happens to the old pipeline?**
It still runs — it feeds the cleanse. This page deliberately shows only the
ETL path; legacy `api/import` → MongoDB is a separate live lane.
