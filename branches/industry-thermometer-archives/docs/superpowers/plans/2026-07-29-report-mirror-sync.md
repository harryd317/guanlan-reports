# Report Mirror Sync Implementation Plan

> **For Codex:** Execute in this session with test-first checkpoints. Do not push until all local verification and the final secret gate pass.

**Goal:** Publish a sanitized `docs/` and `reports/` mirror and make the same policy run automatically after future delivery commits.

**Architecture:** A standalone Python script builds an allowlisted copy in a temporary clone, applies permanent exclusions and path redaction, scans the completed copy for live or unapproved credentials, then commits and pushes only the mirror `main`. A versioned `post-commit` hook invokes the script when a commit touches delivery paths.

**Tech Stack:** Python standard library, Git, `unittest`, POSIX shell hook.

---

### Task 1: Specify policy with tests

**Files:**
- Create: `test_report_mirror_sync.py`
- Test: `test_report_mirror_sync.py`

1. Add tests for strict directory/file allowlists and permanent exclusions.
2. Add tests for path redaction and replacement counts.
3. Add tests for live-secret blocking, approved synthetic fixtures, and `sk-xxx`.
4. Add tests for delivery-path hook filtering.
5. Run the tests and confirm they fail because the implementation does not exist.

### Task 2: Implement the mirror builder and gate

**Files:**
- Create: `scripts/__init__.py`
- Create: `scripts/report_mirror_sync.py`
- Modify: `test_report_mirror_sync.py`

1. Implement deterministic traversal, exclusions, type allowlists, and redaction.
2. Implement final tree validation and credential scanning without logging values.
3. Implement temporary-clone commit/push flow and JSON statistics.
4. Run unit tests until green.

### Task 3: Attach the delivery hook

**Files:**
- Create: `.githooks/post-commit`

1. Invoke the sync only for commits that touch `docs/` or `reports/`.
2. Log each run’s file count, exclusions, replacements, and push state.
3. Configure `core.hooksPath` after the implementation commit.

### Task 4: Verify and publish

**Files:**
- Verify: temporary mirror clone

1. Run the full unit test suite for the new synchronization component.
2. Run a dry-run mirror build and inspect its complete file list.
3. Confirm the two screenshots, `.DS_Store`, backups, code, configuration, databases, models, and logs are absent.
4. Confirm no source user path or live/unapproved API key remains.
5. Commit the local delivery-flow changes.
6. Run the synchronization with push enabled.
7. Fresh-clone the remote and repeat allowlist, exclusion, redaction, and credential checks.
