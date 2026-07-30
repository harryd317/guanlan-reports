# Report Mirror Branch Coverage Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Mirror every `codex/` branch's committed `docs/` and `reports/` under `branches/<name>/` while preserving the existing publication gates.

**Architecture:** The main repository owns one versioned hook and one branch-aware Python publisher. The publisher exports committed Git trees, builds each sanitized delivery subtree with the existing policy, updates one or all mirror destinations in a single clone, validates the complete tree, and pushes one commit.

**Tech Stack:** Python standard library, Git, `unittest`, POSIX shell.

## Global Constraints

- Preserve the current allowlists, exclusions, path redaction, and credential scanning.
- Read committed trees only; ignore uncommitted worktree content.
- Keep root `docs/` and `reports/` for `main`.
- Map `codex/<name>` to `branches/<name>/`.
- Skip every non-`main`, non-`codex/` branch.
- Do not change S2, S3, S5, production databases, research databases, reports, ledgers, or sealed artifacts.
- Keep every existing branch pointer unchanged.
- Use one atomic mirror commit for the initial all-branch backfill.

---

### Task 1: Lock branch routing and nested policy

**Files:**
- Modify: `test_report_mirror_sync.py`
- Modify: `scripts/report_mirror_sync.py`

**Interfaces:**
- Produces: `branch_destination(branch_name: str) -> pathlib.PurePosixPath | None`
- Produces: nested mirror validation and secret scanning with branch-prefix normalization

- [ ] Add a failing routing test for `main`, `codex/s5-breadth-etf-backtest`, nested `codex/a/b`, and unsupported branches.
- [ ] Add a failing nested-tree test that proves the existing exclusions, redaction, and secret approvals apply under `branches/<name>/`.
- [ ] Run the focused tests and confirm both fail because branch routing and nested scanning do not exist.
- [ ] Implement the smallest routing and nested-tree changes.
- [ ] Run the focused tests and confirm they pass.

### Task 2: Export committed branch trees

**Files:**
- Modify: `test_report_mirror_sync.py`
- Modify: `scripts/report_mirror_sync.py`

**Interfaces:**
- Produces: `export_delivery_tree(source_root: Path, ref: str, destination: Path) -> str`
- Returns: the resolved commit SHA

- [ ] Add a failing test that commits one document, changes it without committing, exports `HEAD`, and expects the committed text.
- [ ] Add a failing test for a branch with only `docs/` and no `reports/`.
- [ ] Run the tests and confirm they fail because export support does not exist.
- [ ] Implement safe `git archive` extraction for the allowlisted roots.
- [ ] Run the tests and confirm they pass.

### Task 3: Update one branch without erasing siblings

**Files:**
- Modify: `test_report_mirror_sync.py`
- Modify: `scripts/report_mirror_sync.py`

**Interfaces:**
- Produces: `sync_refs(source_root: Path, refs: Sequence[str], remote: str, push: bool) -> dict`
- Preserves: mirror root and every destination not listed in `refs`

- [ ] Add a failing integration test with root delivery files and two branch destinations.
- [ ] Sync one branch and assert that root files and the sibling branch remain byte-identical.
- [ ] Run the test and confirm it fails against the current whole-tree clearing behavior.
- [ ] Implement destination-scoped replacement and one-clone, one-commit publishing.
- [ ] Run the integration test and confirm it passes.

### Task 4: Backfill all codex branches

**Files:**
- Modify: `test_report_mirror_sync.py`
- Modify: `scripts/report_mirror_sync.py`

**Interfaces:**
- Produces: `list_codex_branches(source_root: Path) -> list[str]`
- CLI: `--all-codex-branches`

- [ ] Add a failing test with two `codex/` branches and one unrelated branch.
- [ ] Assert that the batch result includes only the two `codex/` destinations and creates one mirror commit.
- [ ] Run the test and confirm it fails because batch enumeration is absent.
- [ ] Implement deterministic branch enumeration and the CLI switch.
- [ ] Run the test and confirm it passes.

### Task 5: Share one post-commit hook across worktrees

**Files:**
- Modify: `.githooks/post-commit`
- Modify: `test_report_mirror_sync.py`

**Interfaces:**
- Consumes: Git common directory and current worktree root
- Calls: main repository `scripts/report_mirror_sync.py --source-root <current-worktree> --if-head-touches-delivery --push`

- [ ] Add an executable hook integration test using a linked worktree whose branch lacks the script.
- [ ] Confirm the test fails because the current hook looks for the script inside that worktree.
- [ ] Change the hook to locate the common repository root and execute the shared script.
- [ ] Run the hook integration test and confirm it passes.

### Task 6: Verify, configure, and backfill

**Files:**
- Verify: source branches and temporary mirror clone
- Configure: repository-local `core.hooksPath`

- [ ] Run `python3 -m unittest test_report_mirror_sync -v`.
- [ ] Run the existing S5 and S3 focused suites that guard the changed repository.
- [ ] Record the four重点 branch tips and sealed artifact hashes.
- [ ] Configure `core.hooksPath` to the main repository's absolute `.githooks` path.
- [ ] Run the all-`codex/` dry build and inspect every destination.
- [ ] Run the all-`codex/` push once.
- [ ] Clone the mirror afresh and verify structure, redaction, exclusions, credentials,重点 branch file equality, and source hashes.

