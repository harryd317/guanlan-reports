# Industry Archive v2.2 Review Remediation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Apply the Kimi four-ruler review to the Animal Health II draft and prevent the same defects in future archive drafts with a versioned Prompt v2.2 skeleton and validator.

**Architecture:** Extend the existing skeleton builder with normalized thermometer and cycle-indicator fields, then make v2.2 generation use a stricter prompt and version-aware validation while retaining v2.1 compatibility. Preserve the original draft, add a corrected review candidate and immutable review evidence, and keep every production database read-only.

**Tech Stack:** Python 3, SQLite read-only URIs/CLI, `unittest`, Markdown evidence, Git mirror scripts.

## Global Constraints

- Implement on `ui/today-two-lines-final`; after all mirror and production-equivalent gates pass, merge to `main` and deploy under the user's 2026-08-08 authorization.
- Do not read or print `KIMI_API_KEY`; no secret may enter logs, tests, Git, or mirror artifacts.
- Do not publish the Animal Health II archive; the corrected result remains `改后待再审`.
- Do not write `screener.db`, `market.db`, `industry_thermometer_sidecar.db`, `S8_research.db`, or any T1 snapshot.
- New generation uses `industry-archive-v2.2-web`; historical v2.1 drafts retain their existing validation behavior.
- Completion requires a committed final report and a freshly cloned, verified mirror commit hash.

---

### Task 1: Freeze v2.2 skeleton behavior with tests

**Files:**
- Modify: `test_s8_research.py`
- Modify: `s8_research/archives.py`
- Modify: `s8_research/hook.py`
- Modify: `service.py`

**Interfaces:**
- Consumes: the stored thermometer snapshot object already present in `DailyInputs.thermometer`.
- Produces: `build_industry_skeleton(..., thermometer_snapshot=None, cycle_indicators=None) -> dict` with `thermometer_snapshot` and `cycle_indicators` keys.

- [ ] **Step 1: Write a failing skeleton test**

Add a test using hand-written thermometer and partial cycle fixtures. Assert the real builder preserves the snapshot date/heat/lights, sorts members as before, keeps an available pig-price input, and emits explicit `missing` values for breeding profit and vaccine batch release.

- [ ] **Step 2: Run the focused test and verify RED**

Run `.venv/bin/python -m unittest test_s8_research.S8ArchiveTests.test_skeleton_injects_thermometer_and_explicit_cycle_gaps -v` and require a signature or missing-key failure.

- [ ] **Step 3: Implement the minimal normalized fields**

Add optional inputs to `build_industry_skeleton`; always emit the three fixed cycle keys. Pass the matching thermometer industry row plus snapshot data date from the daily hook and current-service builder. Keep missing inputs as structured `missing`; never search for them.

- [ ] **Step 4: Run focused skeleton tests and verify GREEN**

Run the new test and the existing missing-profit-margin test.

### Task 2: Freeze Prompt v2.2 and version-aware admission

**Files:**
- Modify: `test_industry_archive_center.py`
- Modify: `reviewer.py`
- Modify: `s8_research/archives.py`

**Interfaces:**
- Consumes: a v2.2 skeleton and the prompt version saved by `begin_archive_generation`.
- Produces: `validate_archive_v2(..., prompt_version=None)` with legacy v2.1 behavior by default and v2.2-only errors when requested.

- [ ] **Step 1: Write failing behavioral tests**

Add tests that generate through the real prompt boundary and assert the returned version is `industry-archive-v2.2-web`. Add validator fixtures proving v2.2 rejects a missing thermometer snapshot, missing cycle indicators, absent evidence tier, `板块最新指数`, and `近20日涨幅 +1.87%（单日）`, while the same legacy draft remains governed by v2.1 rules.

- [ ] **Step 2: Run the focused tests and verify RED**

Run the new `ArchiveV22Tests` plus the existing archive draft channel test; require failures against v2.1.

- [ ] **Step 3: Implement Prompt v2.2 and version-aware validation**

Bump the prompt version. Require the skeleton thermometer at the top of section 1, explicit cycle gaps, BK/SW code separation, stale market-snapshot warnings, and tiered sources. Read the in-progress row's prompt version before admission and apply v2.2 rules only to v2.2 generations.

- [ ] **Step 4: Run focused prompt/lifecycle tests and verify GREEN**

Run all `test_industry_archive_center.py` tests and the skeleton tests.

### Task 3: Preserve the review and create the corrected candidate

**Files:**
- Create: `reports/evidence/industry-archive-v2.2-20260808/review/动物保健Ⅱ-四尺审校意见-20260807.md`
- Create: `reports/evidence/industry-archive-v2.2-20260808/review/动物保健Ⅱ-改后待再审初稿.md`
- Create: `reports/evidence/industry-archive-v2.2-20260808/validation.json`

**Interfaces:**
- Consumes: the immutable original draft, the user's four-field review, and the latest read-only 801018.SI thermometer snapshot.
- Produces: a corrected, unpublished v2.2 candidate and machine-readable validation evidence.

- [ ] **Step 1: Record protected database hashes and read the exact snapshot**

Use `shasum -a 256` and `sqlite3 -readonly`. Record only paths, hashes, the relevant snapshot row, and dates; do not copy databases.

- [ ] **Step 2: Save the four-ruler review verbatim as evidence**

Preserve facts/specification/logic/conclusion and the three pipeline improvements. Mark the decision `改后再审`.

- [ ] **Step 3: Write the corrected candidate**

Apply all five factual corrections, put the 801018.SI thermometer snapshot first, keep BK1254 separate and stale-labelled, make all three cycle gaps explicit, and add a source tier to every external evidence line. Keep five or fewer core stocks and both draft warnings.

- [ ] **Step 4: Run the real v2.2 validator**

Call `validate_archive_v2(candidate, prompt_version='industry-archive-v2.2-web')`. Record the exact result plus explicit probes for the five review defects.

### Task 4: Full verification and immutable reporting

**Files:**
- Create: `docs/验收-产业档案Prompt-v2.2与动物保健Ⅱ返修-20260808.md`
- Create: `reports/evidence/industry-archive-v2.2-20260808/tests/`

**Interfaces:**
- Consumes: source changes, corrected candidate, before hashes, and existing regression commands.
- Produces: committed final report and sanitized test/hash evidence.

- [ ] **Step 1: Run focused and full Python regressions**

Run the archive/S8 suite, the established core unittest set, `test_web.py`, rules 8/20/6, zero-scroll suite where applicable, mirror tests, and `compileall`. Save concise outputs.

- [ ] **Step 2: Recompute protected hashes**

Require byte-identical hashes for the thermometer sidecar and S8 production database. Record no database content beyond the approved snapshot evidence.

- [ ] **Step 3: Run the secret blocker**

Scan tracked changes and mirror input with the existing secret gate. Require zero Kimi key-pattern hits and no private configuration file.

- [ ] **Step 4: Commit source, tests, evidence, and final report**

Commit on `ui/today-two-lines-final`. Report the exact source commit; do not merge or deploy.

### Task 5: Mirror sync and fresh-clone verification

**Files:**
- Modify: mirror output under the existing `today-two-lines-final` alias only through the repository's mirror scripts.

**Interfaces:**
- Consumes: the committed `docs/` and `reports/` artifacts plus permitted mirror-safe source metadata.
- Produces: a pushed mirror commit and a clean-clone verification of all new artifacts.

- [ ] **Step 1: Build and dry-run the mirror**

Require the mirror allowlist and secret gate to pass before push.

- [ ] **Step 2: Push over the configured HTTPS mirror channel**

Use the existing alias `codex/today-two-lines-final`; do not print credentials.

- [ ] **Step 3: Fresh-clone the mirror and verify**

Clone to a new temporary directory, resolve the remote branch, verify the corrected draft, review evidence, final report, and source-commit metadata, then report the mirror commit hash.

### Task 6: Production-equivalent acceptance, merge, and deploy

**Files:**
- Modify: the final acceptance report with equivalent and production hashes, health probes, merge/deploy/rollback commits, and screenshots or route probes required by the permanent gate.

**Interfaces:**
- Consumes: the verified source commit, mirror commit, production LaunchAgent clone, read-only database copies, and existing deployment scripts.
- Produces: a healthy production instance on the main branch or a verified rollback to the prior deployment.

- [ ] **Step 1: Run the production-equivalent environment**

Clone the production plist without modifying its bytes. Use the same working-directory semantics, `start.sh`, static serving, and a separate port. Use database copies in read-only mode; record production plist, source databases, and copies before/after hashes.

- [ ] **Step 2: Verify equivalent routes and archive behavior**

Require `ok=true`, no old-source API/render errors, current pages, the archive queue read model, and v2.2 generation metadata. Prove the Animal Health II corrected candidate remains unpublished and no production data source changed.

- [ ] **Step 3: Merge and deploy with a rollback anchor**

Record the previous main/deployment commit, merge the verified branch, run the repository deployment/restart path, and require the launchd fingerprint to match the new source.

- [ ] **Step 4: Verify production and protected hashes**

Require health `ok=true`, representative page/API probes, v2.2 prompt metadata, and byte-identical protected database hashes. If any check fails, restore the recorded deployment commit and restart, then report the rollback instead of success.

- [ ] **Step 5: Update and re-sync the final report**

Commit the production outcome on main, rebuild the mirror, push over HTTPS, and fresh-clone the final mirror commit. The task is not complete until this second mirror hash is verified.
