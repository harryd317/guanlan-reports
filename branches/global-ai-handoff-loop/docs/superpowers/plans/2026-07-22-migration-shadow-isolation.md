# Migration Shadow Isolation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make classified migration shadow and authenticated staging classification deterministic and isolated from live legacy archives, while retaining bounded staging digests, metadata-only quarantine, and every ordinary runtime behavior.

**Architecture:** Add a rule-first migration fallback classifier that produces `general-temporary / migration_temporary / private` plus the existing safe pending payload. Propagate migration behavior only from classified dry-run backfill or the identity-checked staging capability. Render shadow archives and digest previews against a temporary empty Vault view; staging preserves pending/quarantine and skips semantic reconciliation, while the existing bounded project digest/handoff/overview batch still completes.

**Tech Stack:** Python 3.9 standard library, `unittest`, immutable dataclasses, local JSON state and Markdown archives.

## Global Constraints

- Use only synthetic fixtures; never read or output real session text or real user paths.
- Migration shadow makes zero Hermes calls. Staging makes zero unmatched-classification and reconciliation calls, but its first pass retains the approved bounded project digest/Handoff/Overview Hermes batch; the unchanged second pass makes zero calls.
- Unmatched migration segments use project `general-temporary`, source `migration_temporary`, sensitivity `private`, and remain safely pending.
- Ordinary run, collect, reconcile, and catch-up behavior remains unchanged and retains Hermes fallback.
- Classified `collect --dry-run` and backfill shadow read the real registry and state read-only, but never read, merge, or validate a live legacy managed archive and write no live bytes.
- Staging quarantines unsafe messages as strict metadata-only evidence, archives only safe messages, and continues without exposing quarantined content to Vault or Hermes.
- Only `_STAGING_MIGRATION_CAPABILITY` may enable write-capable staging behavior; a forged object and every ordinary entry fail closed.
- Do not weaken credential scanning, touch trading code, or stage unrelated shared-worktree changes.

---

### Task 1: Close migration shadow and staging isolation

**Files:**
- Modify: `ai_session_sync_classify.py`
- Modify: `ai_session_sync_core.py`
- Modify: `ai_session_sync.py`
- Create: `test_ai_session_sync_migration_shadow.py`

**Interfaces:**
- Produces: `classify_segment_for_migration(segment, registry) -> ClassificationDecision`.
- Extends: `collect_sessions(..., migration_unmatched=False, archive_vault=None)` with validation that an alternate archive view is migration-classified dry-run only, has no symlink boundary, and does not equal, contain, or sit inside the live Vault.
- Extends: `_collect_stage(..., _staging_capability=None)` and passes the existing identity capability from `_run_pipeline_locked`.
- Preserves: all existing public CLI signatures and normal classification/reconciliation/digest behavior.

- [ ] **Step 1: Write failing synthetic tests**

Add tests that prove: 101 unmatched segments call no classification or reconciliation client and persist the exact temporary/private pending classification; classified collect dry-run also routes 101 unmatched with zero client calls and preserves the exact activation-smoke CLI contract; deterministic matches retain deterministic routing; a malformed live legacy archive cannot affect classified shadow and the entire isolated root byte/mode snapshot remains equal; report/stdout contains no marker or path; live-child/live-parent/symlink archive views fail closed; safe plus quarantined staging completes with strict metadata-only quarantine and no unsafe bytes in pending/receipts/state/managed/Hermes prompts; staging keeps pending across its internal reconcile point while still producing digested/current-handoff/global-overview artifacts and honestly reports `reconciled=False`; ordinary classifier/reconcile paths still invoke supplied clients; forged staging capability and valid token with a non-staging config fail before writes; first/second staging manifests and pending payloads are identical and the unchanged second pass makes zero Hermes calls.

- [ ] **Step 2: Run the focused tests and verify RED**

Run:

```bash
python3 -m unittest test_ai_session_sync_migration_shadow.py
```

Expected: failures show missing migration classifier/options, live legacy archive reads, staging quarantine abort, staging Hermes calls, or pending consumption.

- [ ] **Step 3: Add the deterministic migration fallback**

Implement `classify_segment_for_migration` by reusing the exact deterministic routing loop from `classify_segment`. For an unmatched safe segment, return a queued `PendingClassification` whose classification is category `综合临时`, project `general-temporary`, no topic tags, segment content type, sensitivity `private`, confidence `0.0`, source `migration_temporary`, and scope `personal`; use the same category/project as the proposal and empty safe name/alias metadata.

- [ ] **Step 4: Isolate classified shadow archives**

Allow core collection to select the migration classifier. Permit `archive_vault` only when `dry_run`, `classified`, migration fallback, and the internal identity capability are all present; reject overlap in both directions plus every symlink-to-live boundary before materialization. In classified collect dry-run and backfill shadow, create a resolved temporary empty Vault view, render the classified archive and its private ledger into that disposable view, and have backfill run digest preview against the rendered view. Registry, state, pending, sources, and the live Vault remain configured read-only inputs, and the temporary view is removed on exit.

- [ ] **Step 5: Gate staging and defer online work**

Validate `_staging_capability` by identity and staging-config boundary again in `_collect_stage`, pass it only from `_run_pipeline_locked`, and enable migration fallback only for the valid capability. Under valid staging, continue after strict metadata-only quarantine and skip semantic reconciliation while retaining the pending queue; report `reconciled=False` and do not record false reconciliation success. Preserve `_digest_staging_batch`, archived-to-digested completion, current handoffs, global overview, and the existing stable gate; the first pass may make one bounded digest call per ready project and the unchanged second pass must make zero.

- [ ] **Step 6: Run focused GREEN and regression tests**

Run the focused file, then the classification/core/migration suites. Confirm all new tests pass and ordinary existing tests remain green; update only existing assertions whose documented staging contract has intentionally changed from eager Hermes digest to deferred pending.

- [ ] **Step 7: Run full verification and commit only owned files**

Run AI/Obsidian, migration, web, rules, `py_compile`, `bash -n`, and `git diff --check`. Stage only the five implementation/test files plus this plan, excluding unrelated scanner changes, and create one focused Chinese commit.
