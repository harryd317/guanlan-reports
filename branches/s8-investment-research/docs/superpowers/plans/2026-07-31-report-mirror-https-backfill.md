# Report Mirror HTTPS Backfill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the public report mirror publish through HTTPS from every linked worktree, preserve proxy variables, include `AGENTS.md`, and backfill every publishable branch with remote hash verification.

**Architecture:** Keep the shared post-commit hook and isolated Git environment introduced by commit `f8b97840e4cabbe8172dcf22b9cba152c05b75b3`. Change the publisher's default remote to GitHub HTTPS, preserve proxy variables while scrubbing repository-scoped Git variables, and extend the sanitized delivery allowlist with the top-level `AGENTS.md` file. Publish `main` and every local `codex/*` ref, then verify a fresh HTTPS clone.

**Tech Stack:** Python standard library, `unittest`, Git, POSIX shell, macOS Keychain HTTPS credentials.

## Global Constraints

- Commit `f8b97840e4cabbe8172dcf22b9cba152c05b75b3` must remain an ancestor of `main` and `codex/t7-today-two-lines`.
- The default mirror remote must be `https://github.com/harryd317/guanlan-reports.git`; no SSH alias may remain in the publisher.
- `_git_environment()` must remove repository-scoped `GIT_*` variables and preserve `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, `NO_PROXY` and their lowercase variants exactly when present.
- The hook must continue to execute the shared main-worktree publisher for commits made in any linked worktree.
- The public allowlist is limited to `docs/`, `reports/`, and top-level `AGENTS.md`; do not publish `MEMORY.md`, code, configuration, databases, logs, backups, credentials, or sealed artifacts.
- All text delivery files must retain private-path redaction and credential scanning.
- Backfill must include `main` and every local `refs/heads/codex/*`, including T7 and S6.
- A task is not complete until a fresh HTTPS clone matches the pushed mirror commit and the final report returns that full mirror hash.
- Do not deploy, restart, stop, or modify the production web service.

---

### Task 1: Lock HTTPS, proxy inheritance, and delivery routing

**Files:**
- Modify: `test_report_mirror_sync.py`

**Interfaces:**
- Consumes: `scripts.report_mirror_sync.DEFAULT_REMOTE`
- Consumes: `scripts.report_mirror_sync._git_environment() -> dict`
- Consumes: `scripts.report_mirror_sync.should_sync_paths(paths: Sequence[str]) -> bool`
- Produces: regression tests for HTTPS, proxy inheritance, repository-environment scrubbing, and `AGENTS.md` hook triggering

- [ ] **Step 1: Import the transport contract**

Add `DEFAULT_REMOTE` and `_git_environment` to the existing import list from `scripts.report_mirror_sync`.

- [ ] **Step 2: Write the failing HTTPS test**

```python
def test_default_remote_uses_github_https_without_ssh_alias(self):
    self.assertEqual(
        DEFAULT_REMOTE,
        "https://github.com/harryd317/guanlan-reports.git",
    )
    self.assertNotIn("git@", DEFAULT_REMOTE)
    self.assertNotIn("github-personal:", DEFAULT_REMOTE)
```

- [ ] **Step 3: Write the proxy-preservation regression test**

```python
def test_git_environment_preserves_proxy_variables_and_scrubs_repository_state(self):
    inherited = {
        "HTTP_PROXY": "http://127.0.0.1:7890",
        "HTTPS_PROXY": "http://127.0.0.1:7890",
        "ALL_PROXY": "socks5://127.0.0.1:7891",
        "NO_PROXY": "127.0.0.1,localhost",
        "http_proxy": "http://127.0.0.1:7892",
        "https_proxy": "http://127.0.0.1:7892",
        "all_proxy": "socks5://127.0.0.1:7893",
        "no_proxy": "localhost",
        "GIT_DIR": "/wrong/repository",
        "GIT_WORK_TREE": "/wrong/worktree",
        "GIT_INDEX_FILE": "/wrong/index",
    }
    with mock.patch.dict(os.environ, inherited, clear=True):
        environment = _git_environment()

    for name in (
        "HTTP_PROXY",
        "HTTPS_PROXY",
        "ALL_PROXY",
        "NO_PROXY",
        "http_proxy",
        "https_proxy",
        "all_proxy",
        "no_proxy",
    ):
        self.assertEqual(environment[name], inherited[name])
    for name in ("GIT_DIR", "GIT_WORK_TREE", "GIT_INDEX_FILE"):
        self.assertNotIn(name, environment)
    self.assertEqual(environment["GIT_TERMINAL_PROMPT"], "0")
```

- [ ] **Step 4: Extend the hook-filter test**

Add:

```python
self.assertTrue(should_sync_paths(["AGENTS.md"]))
self.assertFalse(should_sync_paths(["MEMORY.md"]))
```

- [ ] **Step 5: Run the focused tests and record RED**

Run:

```bash
python3 -m unittest \
  test_report_mirror_sync.ReportMirrorSyncTests.test_default_remote_uses_github_https_without_ssh_alias \
  test_report_mirror_sync.ReportMirrorSyncTests.test_git_environment_preserves_proxy_variables_and_scrubs_repository_state \
  test_report_mirror_sync.ReportMirrorSyncTests.test_hook_filter_only_matches_delivery_paths -v
```

Expected: the HTTPS test and `AGENTS.md` hook assertion fail against the SSH/docs-only implementation; the proxy test passes and locks the already-correct inheritance behavior.

### Task 2: Publish sanitized `AGENTS.md` beside docs and reports

**Files:**
- Modify: `test_report_mirror_sync.py`
- Modify: `scripts/report_mirror_sync.py`

**Interfaces:**
- Produces: `ALLOWED_TOP_LEVEL_FILES = ("AGENTS.md",)`
- Preserves: `ALLOWED_ROOTS = ("docs", "reports")`
- Produces: optional root and branch `AGENTS.md` delivery with the same redaction and credential gates as other text artifacts

- [ ] **Step 1: Write the failing build test**

Extend `test_build_uses_strict_allowlist_and_permanent_exclusions` to create:

```python
self.write_text(
    "AGENTS.md",
    "rule path: ~/project\n",
)
self.write_text("MEMORY.md", "private memory")
```

Expect `AGENTS.md` in the copied list with `~/project`, and expect `MEMORY.md` to remain absent.

- [ ] **Step 2: Write the failing validation tests**

Add a root test that accepts a regular `AGENTS.md` and rejects an `AGENTS.md` directory or symlink. Extend the nested branch test so `branches/s5/AGENTS.md` is accepted, scanned with policy path `AGENTS.md`, and a sibling `MEMORY.md` is rejected.

- [ ] **Step 3: Write the failing committed-tree export test**

Commit `AGENTS.md` with text `committed rule`, modify the worktree copy to `uncommitted rule`, export `HEAD`, and assert the exported file still contains `committed rule`.

- [ ] **Step 4: Run the new tests and record RED**

Run:

```bash
python3 -m unittest \
  test_report_mirror_sync.ReportMirrorSyncTests.test_build_uses_strict_allowlist_and_permanent_exclusions \
  test_report_mirror_sync.ReportMirrorSyncTests.test_validate_mirror_tree_accepts_only_regular_agents_file \
  test_report_mirror_sync.ReportMirrorSyncTests.test_nested_branch_tree_allows_agents_and_rejects_memory \
  test_report_mirror_sync.ReportMirrorSyncTests.test_export_delivery_tree_reads_commit_not_uncommitted_worktree -v
```

Expected: failures show that `AGENTS.md` is not exported, copied, iterated, or validated.

- [ ] **Step 5: Add the top-level file allowlist**

In `scripts/report_mirror_sync.py`, define:

```python
ALLOWED_TOP_LEVEL_FILES = ("AGENTS.md",)
MIRROR_TOP_LEVELS = (*ALLOWED_ROOTS, *ALLOWED_TOP_LEVEL_FILES, "branches")
```

Keep `ALLOWED_SUFFIXES` scoped to `docs` and `reports`.

- [ ] **Step 6: Extend file selection and policy**

Update `exclusion_reason()` so exact paths in `ALLOWED_TOP_LEVEL_FILES` are allowed before directory-root checks. Update `_iter_delivery_files()` to yield each existing regular allowlisted top-level file after traversing `docs/` and `reports/`.

- [ ] **Step 7: Extend build, scan, and validation**

Keep `docs/` and `reports/` mandatory directories. Iterate optional top-level files in `_iter_mirror_files()` at mirror root and within every branch destination. In `validate_mirror_tree()`, require delivery roots to be directories, `AGENTS.md` to be a regular non-symlink file when present, and reject every other top-level or branch child.

- [ ] **Step 8: Extend committed export and replacement**

Make `export_delivery_tree()` probe and archive `docs`, `reports`, and `AGENTS.md`, while only creating the two directory roots afterward. Make `_replace_delivery_destination()` copy or remove root `AGENTS.md` along with replacing root `docs/` and `reports/`; branch replacement remains an atomic directory replacement.

- [ ] **Step 9: Change the default remote and messages**

Set:

```python
DEFAULT_REMOTE = "https://github.com/harryd317/guanlan-reports.git"
```

Update user-facing descriptions from “docs/reports” to “docs/reports/AGENTS.md” where they describe the delivery trigger or commit.

- [ ] **Step 10: Run the focused tests and record GREEN**

Run the four tests from Step 4 plus the three Task 1 tests. Expected: all pass.

- [ ] **Step 11: Commit the tested publisher change**

```bash
git add scripts/report_mirror_sync.py test_report_mirror_sync.py
git commit -m "fix: publish report mirror over HTTPS"
```

The post-commit hook should now use HTTPS. If credentials or network are unavailable, preserve the source commit, record the warning, and continue to the explicit push gate in Task 4.

### Task 3: Verify shared-hook behavior and full regression

**Files:**
- Verify: `.githooks/post-commit`
- Verify: `scripts/report_mirror_sync.py`
- Verify: `test_report_mirror_sync.py`

**Interfaces:**
- Consumes: shared script path resolved from `git rev-parse --git-common-dir`
- Guarantees: any linked worktree calls the main-worktree publisher with its own `--source-root`

- [ ] **Step 1: Run all mirror tests**

Run:

```bash
python3 -m unittest test_report_mirror_sync -v
```

Expected: all tests pass; the previous 22-test suite grows by the new HTTPS and `AGENTS.md` cases.

- [ ] **Step 2: Re-run the old-worktree hook integration test alone**

Run:

```bash
python3 -m unittest \
  test_report_mirror_sync.ReportMirrorSyncTests.test_shared_hook_runs_main_script_for_old_linked_worktree_branch -v
```

Expected: pass.

- [ ] **Step 3: Run static checks**

Run:

```bash
python3 -m py_compile scripts/report_mirror_sync.py test_report_mirror_sync.py
git diff --check
git merge-base --is-ancestor f8b97840e4cabbe8172dcf22b9cba152c05b75b3 main
git merge-base --is-ancestor f8b97840e4cabbe8172dcf22b9cba152c05b75b3 codex/t7-today-two-lines
```

Expected: compilation and whitespace checks succeed; both ancestry commands exit 0.

- [ ] **Step 4: Run the project regression suites**

Run:

```bash
.venv/bin/python test_rules.py
.venv/bin/python test_rules_mid.py
.venv/bin/python test_web.py
.venv/bin/python test_regime.py
```

Expected: all suites pass with no production restart.

### Task 4: Integrate publisher changes into T7 and backfill

**Files:**
- Merge: `main`
- Merge: `codex/t7-today-two-lines`
- Publish: `main`, every local `refs/heads/codex/*`

**Interfaces:**
- Produces: one source commit on `main` containing the HTTPS publisher
- Produces: T7 branch containing the same publisher by cherry-pick before T7 implementation continues
- Produces: final public mirror commit on `main`

- [ ] **Step 1: Confirm clean worktrees**

Run:

```bash
git -C ~/云慧养/zmu_gitee/screener status --short
git -C ~/云慧养/zmu_gitee/screener/.worktrees/t7-today-two-lines status --short
```

Expected: both empty.

- [ ] **Step 2: Cherry-pick only the publisher fix into T7**

Do not merge `main` into T7: `main` contains the production rollback commit that intentionally reverted T7. After the publisher branch is integrated into `main`, cherry-pick only the HTTPS publisher commit from the T7 worktree:

```bash
git cherry-pick 27175b60716c611ac2665a0852abff1526f6517b
```

Expected: clean cherry-pick with the publisher and tests only. T7 interface commits remain present, and production remains on the unchanged running deployment because this is only Git history.

- [ ] **Step 3: Dry-build main and every codex branch**

Run:

```bash
python3 scripts/report_mirror_sync.py --source-root . --all-codex-branches
python3 scripts/report_mirror_sync.py --source-root .
```

Expected: JSON reports zero blocked findings. T7 output contains its production-equivalent design; S6 output contains its formal measurement and acceptance records; `AGENTS.md` appears for publishable refs.

- [ ] **Step 4: Push main mirror root over HTTPS**

Run:

```bash
python3 scripts/report_mirror_sync.py \
  --source-root . \
  --live-secret-root . \
  --push
```

Expected: JSON contains `"pushed": true` or `"changed": false`, plus a full `mirror_commit`.

- [ ] **Step 5: Push every codex branch over HTTPS**

Run:

```bash
python3 scripts/report_mirror_sync.py \
  --source-root . \
  --live-secret-root . \
  --all-codex-branches \
  --push
```

Expected: one atomic branch-backfill commit, zero blocked findings, and a full `mirror_commit`.

### Task 5: Fresh-clone verification and evidence report

**Files:**
- Create: `docs/验收-报告镜像HTTPS与缺失产物补同步-20260731.md`
- Verify: fresh temporary HTTPS clone of `guanlan-reports`

**Interfaces:**
- Consumes: mirror commit returned by Task 4
- Produces: verified mirror hash and file-equality evidence

- [ ] **Step 1: Clone the mirror into a new temporary directory**

Use `mktemp -d`, then:

```bash
git clone --quiet \
  https://github.com/harryd317/guanlan-reports.git \
  <temporary-directory>/guanlan-reports
```

- [ ] **Step 2: Verify the remote hash**

Run `git rev-parse HEAD` in the fresh clone and require exact equality with the final `mirror_commit` from Task 4.

- [ ] **Step 3: Verify mandatory backfill paths**

Require all of these:

```text
AGENTS.md
branches/t7-today-two-lines/AGENTS.md
branches/t7-today-two-lines/docs/superpowers/specs/2026-07-31-t7-production-equivalent-remediation-design.md
branches/s6-screen-measurement/AGENTS.md
```

Discover the exact S6 formal measurement and acceptance report paths from the source commit, then require their matching mirror paths. Compare SHA-256 after replacing the private user prefix in source text, because the mirror redacts that prefix.

- [ ] **Step 4: Verify mirror safety**

Run the complete mirror structure validator and credential scanner against the fresh clone. Require zero blocked findings, no `.env`, database, Python, shell, log, backup, `MEMORY.md`, or unredacted private user prefix.

- [ ] **Step 5: Write and commit the acceptance report**

Record:

- old fix ancestry: `f8b97840e4cabbe8172dcf22b9cba152c05b75b3`;
- HTTPS publisher source commit;
- main and T7 merge commits;
- test commands and counts;
- proxy-variable regression result;
- mandatory path hashes;
- final mirror commit;
- fresh-clone HEAD equality;
- explicit statement that production was not restarted or deployed.

Commit:

```bash
git add docs/验收-报告镜像HTTPS与缺失产物补同步-20260731.md
git commit -m "docs: verify HTTPS report mirror backfill"
```

- [ ] **Step 6: Sync the acceptance report and verify once more**

Push `main` and affected codex branches again through the HTTPS publisher, fresh-clone again, and record the new final mirror hash. This last remotely verified hash is the only mirror hash reported as task completion evidence.
