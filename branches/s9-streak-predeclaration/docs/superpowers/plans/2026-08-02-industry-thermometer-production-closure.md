# Industry Thermometer Production Closure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox syntax (`- [ ]`) for tracking.

**Goal:** Feed the production industry thermometer from a new independently writable sidecar, publish one atomic post-close snapshot only after every upstream stage succeeds, remove the duplicated archive H1, deploy, capture production evidence, and sync the sanitized evidence mirror.

**Architecture:** Seed a new live sidecar by byte-copying the frozen T1 snapshot, then extend only that copy through the already approved T1 endpoints and fields. The post-close hook runs approved raw-source ingestion, the existing S3 derivation transaction, and the thermometer snapshot transaction in order. The HTTP API opens the sidecar snapshot table read-only and compares its date with the read-only production market calendar; it never derives the thermometer during a request.

**Tech Stack:** Python 3, SQLite, `unittest`, existing Tushare-compatible T1 client, `markdown-it-py`, stdlib HTTP service, vanilla HTML/CSS/JavaScript, macOS LaunchAgent, in-app browser.

## Global Constraints

- The frozen T1 snapshot at `.worktrees/s3-stage1-data-clusters/S3_research.db` is read-only source material. Never initialize, migrate, update, replace, move, or compact it.
- Historical raw-source coverage is exactly `2026-01-01` through the latest production trading date. Use only the existing approved endpoints and fields: `daily(vol)` and `sw_daily(open,high,low,close)`; refresh no unrelated source and introduce no fallback provider.
- If the approved client reports quota failure, a required field is missing, an endpoint returns an incomplete trading date, or the frozen SHA-256 changes, stop before installation/deployment and report the evidence.
- New data writes are limited to `industry_thermometer_sidecar.db`, its temporary seed file before atomic installation, and the thermometer snapshot table inside that sidecar. Do not write `market.db`, `screener.db`, the frozen T1 snapshot, or production `S3_research.db`.
- Post-close order is strict: approved T1/S3 raw increment → S3 derived sidecar increment → atomic thermometer snapshot. Any failed stage leaves the previously published snapshot untouched.
- Keep every existing `/api/*` function outside the thermometer endpoint byte-for-byte unchanged. The account-setting write remains the only frontend write path.
- Run production-equivalent desktop `1440×900` and mobile `390×844` checks before production deployment.

---

## Task 1: Lock the snapshot-store contract with failing tests

**Files:**
- Create: `industry_thermometer_snapshot.py`
- Create: `test_industry_thermometer_snapshot.py`
- Modify: `s3_research/db.py`

- [ ] Add a test that `init_research_db()` creates `industry_thermometer_snapshots` with `trade_date` as its primary key and the required metadata columns.
- [ ] Run `.venv/bin/python -m unittest test_industry_thermometer_snapshot.py -v` and confirm the schema test fails because the table does not exist.
- [ ] Add only the snapshot table DDL to `init_research_db()`:

```sql
CREATE TABLE IF NOT EXISTS industry_thermometer_snapshots(
    trade_date TEXT PRIMARY KEY,
    formula_version TEXT NOT NULL,
    payload_json TEXT NOT NULL,
    generated_at TEXT NOT NULL,
    source_sidecar_sha256 TEXT NOT NULL
);
```

- [ ] Add tests for `publish_snapshot(sidecar_path, market_path, expected_trade_date)`: it rejects a mismatched derived date, writes one valid JSON payload atomically, and replacing the same date cannot expose partial JSON.
- [ ] Add tests for `read_latest_snapshot(sidecar_path, market_path)`: it opens the sidecar read-only, returns `data_date`, `latest_trade_date`, `is_stale`, `generated_at`, and preserves the four heat buckets.
- [ ] Implement the minimal writer/reader. Reuse `industry_archives.compute_thermometer()` only in the writer; the reader must execute one read-only snapshot query and one read-only latest-market-date query.
- [ ] Run `.venv/bin/python -m unittest test_industry_thermometer_snapshot.py -v` and confirm all tests pass.
- [ ] Commit: `feat: add atomic thermometer snapshot store`.

## Task 2: Add approved live-range ingestion without a fallback

**Files:**
- Modify: `s3_research/sources.py`
- Create: `test_s3_live_sources.py`

- [ ] Write a fake approved client test for `ingest_live_range(research_db, pro, start, end)` that records every endpoint call and proves the requested range is exactly forwarded as `YYYYMMDD`.
- [ ] Assert the only endpoints called are `daily` with fields `trade_date,ts_code,vol` and `sw_daily` with fields `ts_code,trade_date,open,high,low,close`.
- [ ] Assert missing required columns, an empty latest trading date, a quota exception, or repeated pagination pages raises and commits no completion marker for the failed range.
- [ ] Run `.venv/bin/python -m unittest test_s3_live_sources.py -v` and confirm failures identify the missing API.
- [ ] Implement `ingest_live_range()` using existing `call_with_retry()`, `normalize_source()`, `upsert_source_rows()`, half-month windows for `daily`, and per-approved-L2 calls for `sw_daily`.
- [ ] Record a range-specific checkpoint only after both sources cover the requested latest trading date; ensure retries are idempotent through existing primary keys.
- [ ] Run `.venv/bin/python -m unittest test_s3_live_sources.py -v` and confirm all tests pass.
- [ ] Commit: `feat: ingest approved thermometer live range`.

## Task 3: Enforce the three-stage post-close chain

**Files:**
- Modify: `s3_research/hook.py`
- Modify: `test_s3_research.py`

- [ ] Add ordered-call tests proving `_run_isolated(trade_date)` invokes live raw ingestion first, `runner.run_incremental()` second, and `publish_snapshot()` third.
- [ ] Add one failure test per stage and assert later stages are not called; specifically assert a failed snapshot leaves the prior snapshot row unchanged.
- [ ] Run `.venv/bin/python -m unittest test_s3_research.HookTests -v` and confirm the new tests fail.
- [ ] Implement the ordered chain. Construct the approved client only inside the background worker and pass the configured sidecar and market paths explicitly.
- [ ] Treat `published != True` as a hard stop for snapshot publication while retaining the hook’s failure-isolation behavior toward the main EOD process.
- [ ] Run `.venv/bin/python -m unittest test_s3_research.HookTests -v` and confirm all tests pass.
- [ ] Commit: `feat: chain thermometer publication after s3 success`.

## Task 4: Switch the API to read-only snapshots and fix the archive H1

**Files:**
- Modify: `industry_archives.py`
- Modify: `service.py`
- Modify: `test_industry_archives.py`

- [ ] Add an archive fixture whose body starts with `# 标题`, then includes a later `## 小节`; assert the rendered body omits only the first H1 and retains the later heading and content.
- [ ] Add an API test proving `/api/industry-thermometer` serves a stored snapshot and never calls `compute_thermometer()` during the request.
- [ ] Add a stale API test with a market date later than the snapshot date.
- [ ] Run `.venv/bin/python -m unittest test_industry_archives.py -v` and confirm the new tests fail.
- [ ] Strip only the first Markdown H1 line and its immediately following blank line before rendering archive body HTML.
- [ ] Replace only the thermometer endpoint’s compute call with the read-only snapshot reader; leave all other `/api/*` handlers unchanged.
- [ ] Run `.venv/bin/python -m unittest test_industry_archives.py -v` and confirm all tests pass.
- [ ] Generate and compare a normalized source listing/hash proving non-thermometer `/api/*` function bodies are unchanged from the pre-task commit.
- [ ] Commit: `fix: serve thermometer snapshots and dedupe archive title`.

## Task 5: Render exact data-date copy and four heat styles

**Files:**
- Modify: `static/home.html`
- Modify: `test_industry_archives.py`

- [ ] Add contract assertions for all four pills (`cold`, `warm`, `hot`, `scorching`) and the exact stale string template `数据为${month}月${day}日`.
- [ ] Add an assertion that a current snapshot renders `数据截至${month}月${day}日`, while no snapshot renders `数据未就绪`.
- [ ] Run `.venv/bin/python -m unittest test_industry_archives.FrontendArchiveContractTests -v` and confirm the copy assertion fails.
- [ ] Implement the three-state date label from the API’s `is_stale` field without changing navigation or introducing a new write.
- [ ] Run `.venv/bin/python -m unittest test_industry_archives.FrontendArchiveContractTests -v` and confirm all tests pass.
- [ ] Commit: `fix: label thermometer snapshot freshness`.

## Task 6: Add a guarded sidecar seed/backfill operator command

**Files:**
- Create: `scripts/industry_thermometer_sidecar.py`
- Create: `test_industry_thermometer_sidecar_script.py`

- [ ] Add CLI tests that require explicit frozen source, destination, market DB, start, and end paths/dates.
- [ ] Assert start must equal `2026-01-01`, end must equal the read-only latest market trading date, source and destination resolve to different paths, and an existing destination is rejected.
- [ ] Assert the script hashes the frozen source before copying, copies to a sibling temporary file, ingests only the approved live range, runs `runner.run_backfill()`, publishes the initial snapshot, verifies source hash again, and atomically renames the temporary file only after every check passes.
- [ ] Assert any source, derivation, snapshot, or hash failure leaves the destination absent and prints structured failure evidence.
- [ ] Run `.venv/bin/python -m unittest test_industry_thermometer_sidecar_script.py -v` and confirm failures identify the missing command.
- [ ] Implement the command with a streaming SHA-256 function, read-only latest-date check, free-space preflight, `shutil.copy2`, and `os.replace` only at final installation.
- [ ] Run `.venv/bin/python -m unittest test_industry_thermometer_sidecar_script.py -v` and confirm all tests pass.
- [ ] Commit: `feat: add guarded thermometer sidecar bootstrap`.

## Task 7: Run focused and full verification in the isolated worktree

**Files:**
- Create: `reports/industry-thermometer-tests-20260802.txt`

- [ ] Run `git diff --check` and the focused test files from Tasks 1–6.
- [ ] Run the required suite:

```bash
.venv/bin/python test_rules.py
.venv/bin/python test_rules_mid.py
.venv/bin/python test_web.py
.venv/bin/python test_regime.py
```

- [ ] Run all discoverable unit tests with `.venv/bin/python -m unittest discover -v`.
- [ ] Save command, exit status, test count, and SHA-256 of the raw log in the report without committing secrets or raw data.
- [ ] Inspect `git status --short`; confirm no database, WAL, token, plist, browser profile, or user-owned file is staged.
- [ ] Commit: `test: record thermometer closure verification`.

## Task 8: Bootstrap and verify the production-equivalent sidecar

**Files:**
- Create: `reports/industry-thermometer-sidecar-bootstrap-20260802.json`
- Create: `reports/industry-thermometer-equivalent-20260802.json`
- Create: `docs/screenshots/industry-thermometer-production/20260802/温度计-等效-桌面.jpg`
- Create: `docs/screenshots/industry-thermometer-production/20260802/温度计-等效-手机.jpg`

- [ ] Hash the frozen T1 source and production `market.db`, `screener.db`, and `S3_research.db` before any data operation.
- [ ] Run the guarded bootstrap against an isolated destination and an independent equivalent service port, with copied read-only production databases.
- [ ] If the approved interface lacks quota, omits a required field/date, or returns incomplete coverage, stop here without a destination install or deployment.
- [ ] Query the sidecar and assert raw/derived coverage reaches the latest trading date, snapshot JSON contains at least one `cold`, `warm`, `hot`, and `scorching` industry, and the snapshot date equals latest market date.
- [ ] Rehash the frozen T1 source and assert exact equality with the preflight hash and the prior accepted value `111f2017d476e87dfdc68d3541d7678630d466d410a91f4ab37d8bcb9ce049d5`.
- [ ] Start the equivalent service with the production LaunchAgent environment plus sidecar overrides, then capture desktop and mobile one-screen screenshots showing four colored pill levels and the date.
- [ ] Verify archive reader title appears once, stock-pool back navigation works, legacy URLs return default 404, and account-equity save survives refresh in the equivalent environment.
- [ ] Rehash all copied production databases and assert they match their preflight values.
- [ ] Commit only sanitized JSON evidence and screenshots: `test: accept thermometer production equivalent`.

## Task 9: Deploy, restart, and capture production evidence

**Files:**
- Modify outside Git under explicit authorization: `~/Library/LaunchAgents/com.screener.gui.plist`
- Create: `reports/industry-thermometer-production-20260802.json`
- Create: `docs/screenshots/industry-thermometer-production/20260802/温度计-生产-桌面.jpg`
- Create: `docs/screenshots/industry-thermometer-production/20260802/温度计-生产-手机.jpg`

- [ ] Capture production code revision, service health, plist hash, frozen T1 hash, and protected database hashes before deployment.
- [ ] Run the guarded bootstrap once to create `industry_thermometer_sidecar.db`; do not overwrite an existing destination.
- [ ] Configure production to use that file for `S3_RESEARCH_DB`, `T1_RESEARCH_DB_PATH`, and the thermometer snapshot path, while retaining the existing read-only market path.
- [ ] Run `./restart.sh` because Python production code changed. Confirm exactly one healthy service process and successful API responses.
- [ ] Capture desktop and mobile production screenshots showing all four colored heat pills and a visible data date.
- [ ] Confirm API snapshot date, `latest_trade_date`, and `is_stale`; confirm the archive title appears once and all unrelated smoke checks still pass.
- [ ] Rehash the frozen source and protected databases. The frozen SHA-256 must remain the accepted value; the protected production DB hashes must match pre-deploy hashes.
- [ ] Record the sidecar SHA-256, screenshot SHA-256 values, service PID/health, code commit, and all before/after hashes in sanitized JSON.
- [ ] Commit: `docs: record thermometer production closure`.

## Task 10: Sync and reread the public evidence mirror

**Files:**
- Mirror allowlist only: sanitized `docs/`, `reports/`, and the new acceptance screenshots.

- [ ] Run the repository mirror sanitizer/dry-run and confirm no `.db`, WAL, token, credential, absolute private path, LaunchAgent body, or user-owned `docs/AI评分校准-202608.md` is included.
- [ ] Push through `scripts/report_mirror_sync.py --source-root . --push` to `guanlan-reports`.
- [ ] Record `git ls-remote` HEAD, clone that exact revision into a fresh temporary directory, and reread the production report plus screenshot files from the clone.
- [ ] Verify remote file SHA-256 values equal the local values and the expected production/frozen hashes are present in the reread report.
- [ ] Report the application commit, production sidecar SHA-256, frozen T1 SHA-256, production screenshot paths, and remote mirror commit hash to the user.
