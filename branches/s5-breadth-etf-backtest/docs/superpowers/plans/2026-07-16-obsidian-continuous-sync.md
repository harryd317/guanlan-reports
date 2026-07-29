# Obsidian Continuous Sync Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox syntax for tracking.

**Goal:** Build and activate a commit-driven synchronizer that mirrors screener Markdown, records every commit, generates source-cited Hermes digests, retries failures safely, and pushes the vault without relying on an AI remembering to write it.

**Architecture:** A standard-library Python core reads committed Git history and produces deterministic mirror and ledger changes. A separate orchestrator manages two cursors, Hermes digests, vault publication, and failure recovery. A post-commit wake-up plus a 15-minute launchd job provides immediate and fallback execution.

**Tech Stack:** Python 3 standard library, Git CLI, zsh, launchd, local Hermes API at 127.0.0.1:8642, Obsidian Git vault.

## Global Constraints

- Complete the historical backfill first and seed both cursors at d621afb336bd4cec26634bcd755e64650278065e.
- Source: ~/云慧养/zmu_gitee/screener.
- Vault: ~/obsidian-knowledge-base.
- Runtime: ~/.local/bin/screener-vault-sync.
- State: ~/.local/state/.
- Log: ~/Library/Logs/screener-vault-sync.log.
- Use Python standard library only.
- Mirror committed CLAUDE.md, MEMORY.md, and docs Markdown; index HTML without copying it.
- Exclude code bodies, databases, logs, screenshots, credentials, and uncommitted work.
- Raw mirror publication must succeed when Hermes is unavailable.
- Advance each cursor only after the matching vault commit reaches origin/main.
- Re-running the same range must produce no diff or duplicate.
- Preserve the user's untracked AGENTS.md.
- Do not restart or modify the screener web service.

---

### Task 1: Deterministic Git collection and Markdown mirroring

**Files:**
- Create: ~/云慧养/zmu_gitee/screener/obsidian_sync_core.py
- Create: ~/云慧养/zmu_gitee/screener/test_obsidian_sync.py

**Interfaces:**
- Produces CommitRecord, ChangedPaths, list_commits, changed_paths, mirror_markdown, and render_ledger.
- Consumes a source repo, vault, after SHA, and head SHA.

- [ ] **Step 1: Write failing tests**

Create temporary Git repositories and test:

~~~python
import os
import subprocess
import tempfile
import unittest
from pathlib import Path

from obsidian_sync_core import (
    changed_paths, list_commits, mirror_markdown, render_ledger,
)


def git(repo: Path, *args: str) -> str:
    completed = subprocess.run(
        ['git', '-C', str(repo), *args], check=True,
        text=True, capture_output=True,
    )
    return completed.stdout.strip()


def make_repo() -> tuple[Path, str, str, str]:
    repo = Path(tempfile.mkdtemp())
    git(repo, 'init', '-b', 'main')
    git(repo, 'config', 'user.name', 'Sync Test')
    git(repo, 'config', 'user.email', 'sync@example.invalid')
    (repo / 'CLAUDE.md').write_text('# 基线\n', encoding='utf-8')
    git(repo, 'add', 'CLAUDE.md')
    git(repo, 'commit', '-m', '基线')
    base = git(repo, 'rev-parse', 'HEAD')

    (repo / 'docs').mkdir()
    (repo / 'docs' / '交付.md').write_text('# 交付\n', encoding='utf-8')
    (repo / 'docs' / '原型.html').write_text('<h1>原型</h1>\n', encoding='utf-8')
    git(repo, 'add', 'docs')
    git(repo, 'commit', '-m', '交付: 新增说明')
    first = git(repo, 'rev-parse', 'HEAD')

    (repo / 'code.py').write_text('VALUE = 1\n', encoding='utf-8')
    git(repo, 'add', 'code.py')
    git(repo, 'commit', '-m', '修复: 修正逻辑')
    head = git(repo, 'rev-parse', 'HEAD')
    return repo, base, first, head


class CoreTests(unittest.TestCase):
    def test_commits_are_oldest_first(self):
        repo, base, first, head = make_repo()
        rows = list_commits(repo, base, head)
        self.assertEqual([r.sha for r in rows], [first, head])
        self.assertEqual([r.kind for r in rows], ['交付', '修复'])

    def test_paths_split_markdown_html_and_other(self):
        repo, base, _, head = make_repo()
        paths = changed_paths(repo, base, head)
        self.assertEqual(paths.markdown, ('docs/交付.md',))
        self.assertEqual(paths.html, ('docs/原型.html',))
        self.assertIn('code.py', paths.other)

    def test_mirror_and_ledger_are_idempotent(self):
        repo, base, first, head = make_repo()
        vault = Path(tempfile.mkdtemp())
        paths = changed_paths(repo, base, first)
        self.assertEqual(mirror_markdown(repo, vault, first, paths.markdown),
                         ['docs/交付.md'])
        self.assertEqual(mirror_markdown(repo, vault, first, paths.markdown), [])
        rows = list_commits(repo, base, head)
        once = render_ledger('', rows, {})
        self.assertEqual(render_ledger(once, rows, {}), once)
~~~

- [ ] **Step 2: Verify tests fail**

~~~bash
cd '~/云慧养/zmu_gitee/screener'
.venv/bin/python -m unittest test_obsidian_sync.CoreTests -v
~~~

Expected: import failure for obsidian_sync_core.

- [ ] **Step 3: Implement the core**

Use immutable dataclasses and these exact signatures:

~~~python
def run_git(repo: Path, *args: str, check: bool = True) -> str
def classify_subject(subject: str) -> str
def list_commits(repo: Path, after_sha: str, head_sha: str) -> list[CommitRecord]
def changed_paths(repo: Path, after_sha: str, head_sha: str) -> ChangedPaths
def mirror_markdown(source: Path, vault: Path, head_sha: str,
                    paths: tuple[str, ...]) -> list[str]
def render_ledger(existing: str, commits: list[CommitRecord],
                  topics: dict[str, str]) -> str
~~~

Rules:

- git log uses reverse order and field separators.
- mirror_markdown reads git show HEAD:PATH, never mutable worktree bytes.
- Store full 40-character SHA in the ledger.
- Escape vertical bars in subjects.
- A second run returns no changes and identical ledger text.
- Ledger columns are commit, time, title, type, evidence, topic, and status.

- [ ] **Step 4: Run tests and commit**

~~~bash
.venv/bin/python -m unittest test_obsidian_sync.CoreTests -v
git add obsidian_sync_core.py test_obsidian_sync.py
git commit -m '同步: 添加Git证据镜像核心'
~~~

Expected: core tests pass.

### Task 2: Safety guards and deletion tombstones

**Files:**
- Modify: ~/云慧养/zmu_gitee/screener/obsidian_sync_core.py
- Modify: ~/云慧养/zmu_gitee/screener/test_obsidian_sync.py

**Interfaces:**
- Produces scan_sensitive, dirty_knowledge_paths, deleted_markdown, and write_tombstones.

- [ ] **Step 1: Write failing tests**

~~~python
def test_secret_patterns_stop_sync(self):
    self.assertEqual(scan_sensitive(
        'API_SERVER_KEY=secret-value-1234567890'),
        ['API_SERVER_KEY assignment'])
    self.assertEqual(scan_sensitive('normal note'), [])

def test_dirty_paths_only_report_knowledge_files(self):
    repo, _, _, _ = make_repo()
    (repo / 'docs' / 'draft.md').write_text('draft')
    (repo / 'scratch.py').write_text('draft')
    self.assertEqual(dirty_knowledge_paths(repo), ('docs/draft.md',))

def test_deletion_writes_one_tombstone(self):
    repo, _, first, _ = make_repo()
    git(repo, 'rm', 'docs/交付.md')
    git(repo, 'commit', '-m', '删除旧文档')
    head = git(repo, 'rev-parse', 'HEAD')
    vault = Path(tempfile.mkdtemp())
    self.assertEqual(write_tombstones(
        vault, head, deleted_markdown(repo, first, head)), ['docs/交付.md'])
    self.assertEqual(write_tombstones(
        vault, head, ('docs/交付.md',)), [])
~~~

- [ ] **Step 2: Verify tests fail**

~~~bash
.venv/bin/python -m unittest test_obsidian_sync.CoreTests -v
~~~

- [ ] **Step 3: Implement guards**

Use named regexes for OpenAI-style keys, Bearer tokens, API_SERVER_KEY assignments, and Server酱 SendKey assignments. scan_sensitive returns pattern names. mirror_markdown raises before writing any matching file.

dirty_knowledge_paths parses porcelain v1 with NUL separators and reports only CLAUDE.md, MEMORY.md, AGENTS.md, and docs Markdown/HTML.

deleted_markdown reads deleted Markdown in a commit range. write_tombstones creates one immutable note under 10-项目/screener/源材料/_deleted/PATH.SHA12.md and never erases prior evidence.

- [ ] **Step 4: Run tests and commit**

~~~bash
.venv/bin/python -m unittest test_obsidian_sync.CoreTests -v
git add obsidian_sync_core.py test_obsidian_sync.py
git commit -m '同步: 添加敏感信息与未提交守卫'
~~~

### Task 3: Source-cited Hermes digests and atomic cursors

**Files:**
- Create: ~/云慧养/zmu_gitee/screener/obsidian_sync_digest.py
- Modify: ~/云慧养/zmu_gitee/screener/test_obsidian_sync.py

**Interfaces:**
- Produces build_material, request_digest, validate_digest, read_cursor, and write_cursor.

- [ ] **Step 1: Write failing tests**

~~~python
def test_digest_requires_sections_and_sources(self):
    good = '''# 自动同步摘要

## 本批交付
- 完成确定性镜像。[commit:abcdef1]

## 研究结论与数字
- 本批包含 2 个提交。[source:docs/交付.md]

## 被证伪或终止的方案
- 不再依赖人工提醒。[commit:abcdef1]

## 用户拍板与冻结边界
- 保留未提交文件边界。[source:docs/交付.md]

## 新待办和已过期待办
- 新待办是接入定时任务。[commit:abcdef1]

## 事故与方法论
- 失败时不得推进游标。[source:docs/交付.md]
'''
    validate_digest(good, {'abcdef1'}, {'docs/交付.md'})
    with self.assertRaises(ValueError):
        validate_digest(good.replace('[commit:abcdef1]', '', 1),
                        {'abcdef1'}, {'docs/交付.md'})

def test_cursor_write_is_atomic(self):
    path = Path(tempfile.mkdtemp()) / 'cursor.head'
    self.assertIsNone(read_cursor(path))
    write_cursor(path, 'a' * 40)
    self.assertEqual(read_cursor(path), 'a' * 40)
    self.assertFalse(path.with_name(path.name + '.tmp').exists())
~~~

The validator must require all six headings shown above. Every bullet carries a valid commit or source token.

- [ ] **Step 2: Verify tests fail**

~~~bash
.venv/bin/python -m unittest test_obsidian_sync.CoreTests -v
~~~

- [ ] **Step 3: Implement digest and cursor functions**

request_digest uses urllib.request, reads API_SERVER_KEY from ~/.hermes/.env without logging it, obtains the model from /v1/models, and calls /v1/chat/completions with a 600-second timeout.

validate_digest rejects missing headings, unsourced bullets, unknown commit tokens, unknown source paths, and secret patterns.

write_cursor writes a sibling temporary file, flushes, fsyncs, and uses os.replace.

- [ ] **Step 4: Run tests and commit**

~~~bash
.venv/bin/python -m unittest test_obsidian_sync.CoreTests -v
git add obsidian_sync_digest.py test_obsidian_sync.py
git commit -m '同步: 添加Hermes摘要与双游标'
~~~

### Task 4: Orchestration, publication, and retry behavior

**Files:**
- Create: ~/云慧养/zmu_gitee/screener/obsidian_sync.py
- Modify: ~/云慧养/zmu_gitee/screener/test_obsidian_sync.py

**Interfaces:**
- Produces SyncConfig, publish_vault, sync_mirror, sync_digest, run, and CLI modes dry-run, mirror-only, digest-only.

- [ ] **Step 1: Write failing local-remote integration tests**

Build a temporary bare Git remote and cloned vault with this fixture and tests:

~~~python
from unittest.mock import patch

from obsidian_sync import SyncConfig, run
from obsidian_sync_digest import read_cursor, write_cursor


class SyncIntegrationTests(unittest.TestCase):
    def setUp(self):
        self.source, self.base, self.first, self.head = make_repo()
        self.root = Path(tempfile.mkdtemp())
        self.remote = self.root / 'vault.git'
        subprocess.run(['git', 'init', '--bare', str(self.remote)], check=True,
                       text=True, capture_output=True)
        seed = self.root / 'seed'
        subprocess.run(['git', 'init', '-b', 'main', str(seed)], check=True,
                       text=True, capture_output=True)
        git(seed, 'config', 'user.name', 'Sync Test')
        git(seed, 'config', 'user.email', 'sync@example.invalid')
        (seed / 'README.md').write_text('# Vault\n', encoding='utf-8')
        git(seed, 'add', 'README.md')
        git(seed, 'commit', '-m', 'init vault')
        git(seed, 'remote', 'add', 'origin', str(self.remote))
        git(seed, 'push', '-u', 'origin', 'main')
        self.vault = self.root / 'vault'
        subprocess.run(
            ['git', 'clone', '--branch', 'main', str(self.remote), str(self.vault)],
            check=True, text=True, capture_output=True,
        )
        git(self.vault, 'config', 'user.name', 'Sync Test')
        git(self.vault, 'config', 'user.email', 'sync@example.invalid')
        state = self.root / 'state'
        self.config = SyncConfig(
            source=self.source,
            vault=self.vault,
            state_dir=state,
            log_path=self.root / 'sync.log',
            mirror_cursor=state / 'mirror.head',
            digest_cursor=state / 'digest.head',
            lock_dir=state / 'lock',
        )
        write_cursor(self.config.mirror_cursor, self.base)
        write_cursor(self.config.digest_cursor, self.base)

    def test_remote_receives_mirror_before_cursor_advances(self):
        self.assertEqual(run(self.config, 'mirror-only', self.first, False), 0)
        git(self.vault, 'fetch', 'origin', 'main')
        mirrored = git(
            self.vault, 'show',
            'origin/main:10-项目/screener/源材料/docs/交付.md',
        )
        self.assertEqual(mirrored, '# 交付')
        self.assertEqual(read_cursor(self.config.mirror_cursor), self.first)

    def test_push_failure_does_not_advance_mirror_cursor(self):
        with patch('obsidian_sync.publish_vault',
                   side_effect=RuntimeError('simulated push failure')):
            self.assertEqual(run(
                self.config, 'mirror-only', self.first, False), 1)
        self.assertEqual(read_cursor(self.config.mirror_cursor), self.base)

    def test_hermes_failure_keeps_digest_pending(self):
        with patch('obsidian_sync.request_digest',
                   side_effect=RuntimeError('Hermes unavailable')):
            self.assertEqual(run(self.config, 'all', self.first, False), 0)
        self.assertEqual(read_cursor(self.config.mirror_cursor), self.first)
        self.assertEqual(read_cursor(self.config.digest_cursor), self.base)
        git(self.vault, 'fetch', 'origin', 'main')
        self.assertEqual(git(
            self.vault, 'rev-list', '--left-right', '--count',
            'origin/main...main'), '0\t0')

    def test_second_run_creates_no_commit(self):
        self.assertEqual(run(self.config, 'mirror-only', self.first, False), 0)
        before = git(self.vault, 'rev-parse', 'HEAD')
        self.assertEqual(run(self.config, 'mirror-only', self.first, False), 0)
        self.assertEqual(git(self.vault, 'rev-parse', 'HEAD'), before)

    def test_existing_lock_exits_without_writes(self):
        self.config.lock_dir.mkdir(parents=True)
        self.assertEqual(run(self.config, 'mirror-only', self.first, False), 0)
        self.assertEqual(read_cursor(self.config.mirror_cursor), self.base)

    def test_dry_run_writes_no_vault_or_state(self):
        before_status = git(self.vault, 'status', '--porcelain')
        before_head = git(self.vault, 'rev-parse', 'HEAD')
        before_state = {
            path.name: path.read_bytes()
            for path in self.config.state_dir.iterdir()
        }
        self.assertEqual(run(self.config, 'mirror-only', self.first, True), 0)
        self.assertEqual(git(self.vault, 'status', '--porcelain'), before_status)
        self.assertEqual(git(self.vault, 'rev-parse', 'HEAD'), before_head)
        self.assertEqual({
            path.name: path.read_bytes()
            for path in self.config.state_dir.iterdir()
        }, before_state)
~~~

- [ ] **Step 2: Verify tests fail**

~~~bash
.venv/bin/python -m unittest test_obsidian_sync.SyncIntegrationTests -v
~~~

- [ ] **Step 3: Implement orchestration**

Use these signatures:

~~~python
@dataclass(frozen=True)
class SyncConfig:
    source: Path
    vault: Path
    state_dir: Path
    log_path: Path
    mirror_cursor: Path
    digest_cursor: Path
    lock_dir: Path

def publish_vault(config: SyncConfig, message: str) -> None
def sync_mirror(config: SyncConfig, head: str, dry_run: bool = False) -> str
def sync_digest(config: SyncConfig, head: str, dry_run: bool = False) -> str
def run(config: SyncConfig, mode: str, head: str | None,
        dry_run: bool) -> int
~~~

publish_vault must run pull --rebase --autostash origin main, stage, commit when needed, push origin main, fetch, and require rev-list left/right count 0 0.

sync_mirror publishes mirror, tombstones, HTML index, and ledger before cursor write. sync_digest validates and publishes digest before cursor write. Hermes failure logs a sanitized line and leaves digest pending.

Dry-run prints source range, commit count, Markdown count, HTML count, deletion count, dirty knowledge count, both cursors, and whether vault would change.

- [ ] **Step 4: Run tests and commit**

~~~bash
.venv/bin/python -m unittest test_obsidian_sync -v
git add obsidian_sync.py test_obsidian_sync.py
git commit -m '同步: 添加双阶段发布与失败恢复'
~~~

### Task 5: Repair vault autosync and install triggers

**Files:**
- Create: ~/云慧养/zmu_gitee/screener/install_obsidian_sync.sh
- Modify: ~/云慧养/zmu_gitee/screener/test_obsidian_sync.py
- Modify: ~/.local/bin/vault-autosync.sh
- Modify: ~/obsidian-knowledge-base/90-系统/scripts/vault-autosync.sh
- Create: ~/obsidian-knowledge-base/90-系统/scripts/screener-vault-sync
- Create: ~/Library/LaunchAgents/com.harryd317.screener-vault-sync.plist
- Create: ~/云慧养/zmu_gitee/screener/.git/hooks/post-commit

**Interfaces:**
- Produces an idempotent installer, immediate wake-up, and 15-minute fallback.

- [ ] **Step 1: Add installer dry-run smoke test**

Add this test; the installer must accept `HOME` and `SYNC_SOURCE` overrides so the test never writes to real runtime paths:

~~~python
class InstallerTests(unittest.TestCase):
    def test_dry_run_lists_destinations_and_touches_nothing(self):
        fake_home = Path(tempfile.mkdtemp())
        completed = subprocess.run(
            ['bash', 'install_obsidian_sync.sh', '--dry-run'],
            cwd=Path(__file__).parent,
            env={**os.environ, 'HOME': str(fake_home),
                 'SYNC_SOURCE': str(Path(__file__).parent)},
            check=True, text=True, capture_output=True,
        )
        for expected in (
            '.local/bin/screener-vault-sync',
            'com.harryd317.screener-vault-sync.plist',
            '.git/hooks/post-commit',
            'Library/Logs/screener-vault-sync.log',
            'screener service: untouched',
        ):
            self.assertIn(expected, completed.stdout)
        self.assertEqual(list(fake_home.rglob('*')), [])
~~~

- [ ] **Step 2: Repair vault-autosync.sh**

Use explicit pull from origin main and check its exit code. Return nonzero on push failure. Fetch origin main and require 0 0 divergence. Keep the runtime and vault backup byte-identical.

- [ ] **Step 3: Implement installer**

The installer copies the three Python modules into ~/.local/lib/screener-vault-sync, installs an executable wrapper, writes a plist with RunAtLoad and StartInterval 900, preserves any existing post-commit hook by chaining it, writes a wake-up hook using launchctl kickstart, validates plist, and bootstraps the job. In dry-run mode it writes nothing unless `PLIST_OUTPUT` is explicitly supplied; that override renders only the plist for linting.

- [ ] **Step 4: Test dry-run**

~~~bash
TMP="$(mktemp -d)"
mkdir -p "$TMP/home"
HOME="$TMP/home" PLIST_OUTPUT="$TMP/job.plist" \
  bash install_obsidian_sync.sh --dry-run
plutil -lint "$TMP/job.plist"
test ! -e "$TMP/home/.local"
~~~

Expected: all destinations listed and plist OK.

- [ ] **Step 5: Commit tracked files**

Commit the installer in screener. Commit repaired script backup and new runtime backup in the vault. Do not commit .git/hooks or ~/.local files.

### Task 6: Activate, verify, document, and review

**Files:**
- Modify: ~/obsidian-knowledge-base/90-系统/新机器引导.md
- Modify: ~/obsidian-knowledge-base/90-系统/记忆恢复咒语.md
- Create through Hermes: ~/obsidian-knowledge-base/20-AI复核/2026-07-16-screener自动同步.md

**Interfaces:**
- Consumes completed backfill and automation.
- Produces active job, current vault, remote equality, and recovery documentation.

- [ ] **Step 1: Run all tests before installation**

~~~bash
cd '~/云慧养/zmu_gitee/screener'
.venv/bin/python -m unittest test_obsidian_sync -v
.venv/bin/python test_rules.py
.venv/bin/python test_rules_mid.py
.venv/bin/python test_web.py
.venv/bin/python test_regime.py
~~~

Expected: sync tests pass and project suites remain green. Do not restart web.

- [ ] **Step 2: Production dry-run**

~~~bash
cd '~/云慧养/zmu_gitee/screener'
.venv/bin/python obsidian_sync.py --dry-run
~~~

Expected: start cursor d621afb; pending includes ea3354d, 77ec779, implementation plans, and newer commits. Dirty AGENTS.md appears only as warning.

- [ ] **Step 3: Install and run**

~~~bash
bash install_obsidian_sync.sh
launchctl kickstart -k "gui/$(id -u)/com.harryd317.screener-vault-sync"
~~~

Poll logs in intervals below 60 seconds.

- [ ] **Step 4: Verify cursors, idempotence, and remote**

~~~bash
cat '~/.local/state/screener-vault-mirror.head'
cat '~/.local/state/screener-vault-digest.head'
~/.local/bin/screener-vault-sync
git -C '~/obsidian-knowledge-base' status --short
git -C '~/obsidian-knowledge-base' fetch origin main
git -C '~/obsidian-knowledge-base' rev-list --left-right --count origin/main...main
~~~

Expected: cursors equal source HEAD, second run reports no changes, vault clean, remote 0 0.

- [ ] **Step 5: Update recovery docs**

Document runtime, cursors, launchd label, log, manual run, uninstall, and failure rule.

- [ ] **Step 6: Hermes review**

~~~bash
~/.local/bin/hermes-review 'screener自动同步' /tmp/screener-auto-sync-review.md
~~~

Expected first line 裁决:通过. Fix every required item if returned 打回.

- [ ] **Step 7: Final verification**

~~~bash
launchctl print "gui/$(id -u)/com.harryd317.screener-vault-sync"
rg -n 'SYNC FAILED|Traceback' \
  '~/Library/Logs/screener-vault-sync.log' \
  '~/Library/Logs/vault-autosync.log'
git -C '~/obsidian-knowledge-base' status -sb
~~~

Expected: launchd loaded, no unresolved failure after final success, vault main aligned with origin/main.
