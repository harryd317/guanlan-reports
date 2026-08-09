# AI Session Sync Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Archive all user-visible Codex and Claude conversations in Obsidian, backfill the last 30 days, and let Hermes maintain source-linked daily handoff notes.

**Architecture:** A deterministic Python collector discovers top-level Codex and Claude JSONL sessions, extracts only user-visible text, redacts credentials, and atomically renders one Markdown file per platform, session, and Shanghai date. A separate Hermes layer summarizes changed daily archives, while launchd schedules collection and digest work; the existing Vault autosync remains the only Git publisher.

**Tech Stack:** Python 3.9+ standard library, `unittest`, local Hermes OpenAI-compatible API, zsh installer, launchd, Obsidian Markdown.

## Global Constraints

- Implement `docs/superpowers/specs/2026-07-19-ai-session-sync-design.md` without expanding scope.
- Backfill from `2026-06-20 00:00:00 Asia/Shanghai` through the current time.
- Read all top-level Codex and Claude sessions across all projects.
- Exclude Claude `subagents/`, Codex collaboration events, system/developer instructions, hidden reasoning, tool calls, tool results, diagnostics, usage data, and binary payloads.
- Store only redacted user-visible text in the Vault; never copy source JSONL.
- Use `ZoneInfo("Asia/Shanghai")` for discovery, rendering, schedules, and date partitioning.
- Use Python standard library only; runtime must work with `/usr/bin/python3` 3.9.6.
- Perform managed-note writes with sibling temporary files, `fsync`, and `os.replace`.
- The new program must never run Git commands. Existing Vault autosync remains the sole publisher.
- Preserve the user's untracked `AGENTS.md`, `install_obsidian_sync.sh`, and `obsidian_sync*.py` files.
- Do not modify or restart the screener web service, trading rules, databases, or trading jobs.
- Install only after isolated tests, an isolated end-to-end run, and a production dry run pass.

---

### Task 1: Message model, timestamps, fingerprints, and redaction

**Files:**
- Create: `ai_session_sync_core.py`
- Create: `test_ai_session_sync.py`

**Interfaces:**
- Produces `Message`, `RedactionResult`, `parse_timestamp`, `stable_message_id`, `redact_sensitive`, and `atomic_write_text`.
- Consumed by every later task.

- [ ] **Step 1: Write the failing core tests**

Create `test_ai_session_sync.py` with `CoreTests`. It must assert:

```python
class CoreTests(unittest.TestCase):
    def test_timestamp_converts_to_shanghai(self):
        stamp = parse_timestamp("2026-07-18T16:30:00Z")
        self.assertEqual(stamp.isoformat(), "2026-07-19T00:30:00+08:00")

    def test_stable_message_id_is_repeatable_and_role_sensitive(self):
        args = ("codex", "s1", "user", "2026-07-19T08:00:00+08:00", "hello")
        self.assertEqual(stable_message_id(*args), stable_message_id(*args))
        self.assertNotEqual(
            stable_message_id(*args),
            stable_message_id("codex", "s1", "assistant", args[3], args[4]),
        )
        self.assertEqual(len(stable_message_id(*args)), 64)

    def test_redacts_credentials_and_keeps_normal_text(self):
        raw = "\n".join((
            "OPENAI_API_KEY=sk-" + "a" * 48,
            "Authorization: Bearer abcdefghijklmnopqrstuvwxyz123456",
            "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=secret",
            "normal decision text stays",
        ))
        result = redact_sensitive(raw)
        self.assertNotIn("sk-", result.text)
        self.assertNotIn("abcdefghijklmnopqrstuvwxyz", result.text)
        self.assertNotIn("key=secret", result.text)
        self.assertIn("normal decision text stays", result.text)
        self.assertEqual(set(result.kinds), {"api_key", "bearer", "webhook"})

    def test_redacts_private_key_and_long_base64(self):
        raw = (
            "-----BEGIN OPENSSH PRIVATE KEY-----\nsecret\n"
            "-----END OPENSSH PRIVATE KEY-----\n" + "A" * 120
        )
        result = redact_sensitive(raw)
        self.assertNotIn("OPENSSH PRIVATE KEY", result.text)
        self.assertNotIn("A" * 96, result.text)
        self.assertEqual(set(result.kinds), {"private_key", "base64"})

    def test_atomic_write_replaces_content_and_removes_temp(self):
        root = Path(tempfile.mkdtemp())
        path = root / "note.md"
        atomic_write_text(path, "first\n")
        atomic_write_text(path, "second\n")
        self.assertEqual(path.read_text(encoding="utf-8"), "second\n")
        self.assertEqual(list(root.glob(".note.md.*.tmp")), [])
```

Include imports for `tempfile`, `unittest`, and `Path`, plus the six production symbols.

- [ ] **Step 2: Run the tests and verify RED**

Run:

```bash
.venv/bin/python -m unittest test_ai_session_sync.CoreTests -v
```

Expected: import failure for `ai_session_sync_core`.

- [ ] **Step 3: Implement the minimal core**

Create immutable dataclasses with these fields:

```python
@dataclass(frozen=True)
class Message:
    platform: str
    session_id: str
    message_id: str
    timestamp: datetime
    role: str
    text: str
    project: str
    source_path: str

    @property
    def day(self) -> str:
        return self.timestamp.astimezone(SHANGHAI).date().isoformat()


@dataclass(frozen=True)
class RedactionResult:
    text: str
    kinds: tuple[str, ...]
```

Implement the six interfaces exactly:

```python
SHANGHAI = ZoneInfo("Asia/Shanghai")

def parse_timestamp(raw: str) -> datetime
def stable_message_id(
    platform: str, session_id: str, role: str, timestamp: str, text: str,
) -> str
def redact_sensitive(text: str) -> RedactionResult
def atomic_write_text(path: Path, text: str) -> None
```

`stable_message_id` joins the five inputs with `\x1f` and returns a SHA-256 hex digest. `atomic_write_text` uses `tempfile.mkstemp` in the target directory, flushes, calls `os.fsync`, and replaces with `os.replace`. Redaction patterns run in this order: private-key blocks, notification webhooks, Bearer tokens, named secret assignments, `sk-` keys, and Base64 runs of at least 96 characters. Each match becomes `[已脱敏：<kind>]`; `kinds` preserves first-seen order without duplicates.

- [ ] **Step 4: Verify GREEN and Python 3.9 compatibility**

Run:

```bash
.venv/bin/python -m unittest test_ai_session_sync.CoreTests -v
/usr/bin/python3 -m py_compile ai_session_sync_core.py
```

Expected: 5 tests pass and compilation succeeds.

- [ ] **Step 5: Commit Task 1**

Run:

```bash
git diff --check -- ai_session_sync_core.py test_ai_session_sync.py
git add ai_session_sync_core.py test_ai_session_sync.py
git commit -m '同步: 添加AI会话模型与脱敏核心'
```

### Task 2: Codex and Claude source adapters

**Files:**
- Create: `ai_session_sync_sources.py`
- Modify: `test_ai_session_sync.py`

**Interfaces:**
- Consumes the Task 1 core.
- Produces `ParseReport`, `discover_session_files`, `parse_codex_session`, and `parse_claude_session`.

- [ ] **Step 1: Add failing adapter tests**

Add a `write_jsonl(path, rows)` helper and `SourceTests` with complete fixtures for these cases:

```python
class SourceTests(unittest.TestCase):
    def test_codex_keeps_user_and_assistant_messages_only(self):
        rows = [
            {"type": "session_meta", "timestamp": "2026-07-19T00:00:00Z",
             "payload": {"id": "cx1", "cwd": "/project"}},
            {"type": "response_item", "timestamp": "2026-07-19T00:01:00Z",
             "payload": {"type": "message", "id": "u1", "role": "user",
                         "content": [{"type": "input_text", "text": "用户问题"}]}},
            {"type": "response_item", "timestamp": "2026-07-19T00:02:00Z",
             "payload": {"type": "message", "id": "d1", "role": "developer",
                         "content": [{"type": "input_text", "text": "内部指令"}]}},
            {"type": "response_item", "timestamp": "2026-07-19T00:03:00Z",
             "payload": {"type": "function_call", "name": "exec"}},
            {"type": "response_item", "timestamp": "2026-07-19T00:04:00Z",
             "payload": {"type": "message", "id": "a1", "role": "assistant",
                         "content": [{"type": "output_text", "text": "可见回复"}]}},
        ]
        path = self.write_fixture("codex.jsonl", rows)
        report = parse_codex_session(path)
        self.assertEqual([(m.role, m.text) for m in report.messages], [
            ("user", "用户问题"), ("assistant", "可见回复")
        ])
        self.assertEqual({m.session_id for m in report.messages}, {"cx1"})
        self.assertEqual({m.project for m in report.messages}, {"/project"})

    def test_claude_keeps_text_and_excludes_tool_blocks(self):
        rows = [
            {"type": "user", "uuid": "u1", "sessionId": "cl1",
             "timestamp": "2026-07-19T08:00:00+08:00", "cwd": "/project",
             "message": {"role": "user", "content": "用户问题"}},
            {"type": "assistant", "uuid": "a1", "sessionId": "cl1",
             "timestamp": "2026-07-19T08:01:00+08:00", "cwd": "/project",
             "message": {"role": "assistant", "content": [
                 {"type": "thinking", "thinking": "隐藏推理"},
                 {"type": "tool_use", "name": "Read", "input": {"file": "x"}},
                 {"type": "text", "text": "可见回复"},
             ]}},
            {"type": "user", "uuid": "t1", "sessionId": "cl1",
             "timestamp": "2026-07-19T08:02:00+08:00", "cwd": "/project",
             "message": {"role": "user", "content": [
                 {"type": "tool_result", "content": "终端输出"}
             ]}},
        ]
        report = parse_claude_session(self.write_fixture("claude.jsonl", rows))
        self.assertEqual([(m.role, m.text) for m in report.messages], [
            ("user", "用户问题"), ("assistant", "可见回复")
        ])
```

Also add concrete tests named:

- `test_codex_missing_id_gets_repeatable_hash`
- `test_malformed_middle_and_incomplete_final_lines_are_reported`
- `test_discovery_excludes_claude_subagents_and_old_files`
- `test_adapter_redacts_before_constructing_messages`

The malformed-line test writes two valid rows around `{bad json}` and a final unterminated `{`; it expects both valid messages and `bad_lines == (2, 4)`. The discovery test creates current Codex, current Claude, old Claude, and `subagents/agent.jsonl`; it expects only the two current top-level files.

- [ ] **Step 2: Run adapter tests and verify RED**

Run:

```bash
.venv/bin/python -m unittest test_ai_session_sync.SourceTests -v
```

Expected: import failure for `ai_session_sync_sources`.

- [ ] **Step 3: Implement the adapters**

Create:

```python
@dataclass(frozen=True)
class ParseReport:
    messages: tuple[Message, ...]
    bad_lines: tuple[int, ...]
    redactions: tuple[str, ...]

def discover_session_files(
    codex_root: Path, claude_root: Path, since: datetime,
) -> tuple[tuple[str, Path], ...]

def parse_codex_session(path: Path) -> ParseReport
def parse_claude_session(path: Path) -> ParseReport
```

Read JSONL line by line with UTF-8 replacement. Codex accepts only `response_item` records whose payload is a `message`, role is `user` or `assistant`, and content block type is `input_text` or `output_text`. Claude accepts only top-level `user` and `assistant` records and string content or `text` blocks. Prefer Codex `payload.id` and Claude `uuid`; otherwise use `stable_message_id`. Redact before constructing `Message`. Sort by `(timestamp, message_id)` and deduplicate IDs. Discovery excludes every path containing a `subagents` component and filters clearly old files before parsing.

- [ ] **Step 4: Verify GREEN and commit Task 2**

Run:

```bash
.venv/bin/python -m unittest \
  test_ai_session_sync.CoreTests \
  test_ai_session_sync.SourceTests -v
/usr/bin/python3 -m py_compile ai_session_sync_sources.py
git diff --check -- ai_session_sync_sources.py test_ai_session_sync.py
git add ai_session_sync_sources.py test_ai_session_sync.py
git commit -m '同步: 解析Codex与Claude可见会话'
```

### Task 3: Managed Markdown rendering and atomic incremental state

**Files:**
- Modify: `ai_session_sync_core.py`
- Modify: `test_ai_session_sync.py`

**Interfaces:**
- Produces `SyncState`, `CollectReport`, `load_state`, `save_state`, `archive_path`, `merge_archive`, and `collect_sessions`.
- Consumes Task 2 parser reports.

- [ ] **Step 1: Add failing archive tests**

Add `ArchiveTests` with a `make_message` helper and these assertions:

```python
class ArchiveTests(unittest.TestCase):
    def make_message(self, message_id="m1", minute=0, text="hello"):
        return Message(
            platform="codex", session_id="session/unsafe", message_id=message_id,
            timestamp=parse_timestamp(f"2026-07-19T00:{minute:02d}:00Z"),
            role="user", text=text, project="/project",
            source_path="/sessions/source.jsonl",
        )

    def test_archive_path_is_partitioned_and_filename_safe(self):
        path = archive_path(Path("/vault"), self.make_message())
        self.assertEqual(
            path.parent,
            Path("/vault/90-系统/AI会话/原文/2026-07-19"),
        )
        self.assertEqual(path.name, "Codex-session-unsafe.md")

    def test_merge_is_idempotent_and_orders_messages(self):
        root = Path(tempfile.mkdtemp())
        path = root / "note.md"
        late = self.make_message("m2", 2, "late")
        early = self.make_message("m1", 1, "early")
        self.assertTrue(merge_archive(path, [late, early]))
        once = path.read_text(encoding="utf-8")
        self.assertLess(once.index("early"), once.index("late"))
        self.assertFalse(merge_archive(path, [early, late]))
        self.assertEqual(path.read_text(encoding="utf-8"), once)
        self.assertEqual(once.count("ai-session-sync:m1"), 1)
        self.assertEqual(once.count("ai-session-sync:m2"), 1)

    def test_state_round_trip_is_atomic(self):
        root = Path(tempfile.mkdtemp())
        path = root / "state.json"
        expected = SyncState.empty()
        save_state(path, expected)
        self.assertEqual(load_state(path), expected)
        self.assertEqual(list(root.glob(".state.json.*.tmp")), [])
```

Add complete integration tests named:

- `test_collect_write_failure_does_not_advance_state`: patch `atomic_write_text` to raise on a managed note; assert the persisted state bytes stay unchanged.
- `test_active_session_append_adds_exactly_one_message`: collect one Codex message, append one JSONL row, collect again; assert two unique markers.
- `test_cross_midnight_session_writes_two_day_files`: use 23:59 and 00:01 Shanghai messages; assert two dated output files.
- `test_existing_markers_prevent_duplicates_after_state_loss`: delete state after a successful collect, rerun, and assert unchanged archive bytes.
- `test_source_truncation_preserves_archived_blocks`: truncate the source to one message and assert the older managed block remains.
- `test_collect_dry_run_writes_neither_vault_nor_state`: assert populated counts but no filesystem changes.

- [ ] **Step 2: Run archive tests and verify RED**

Run:

```bash
.venv/bin/python -m unittest test_ai_session_sync.ArchiveTests -v
```

Expected: missing archive symbols from `ai_session_sync_core`.

- [ ] **Step 3: Implement state, rendering, and collection**

Add:

```python
@dataclass(frozen=True)
class SyncState:
    version: int
    sources: dict[str, dict[str, object]]
    digests: dict[str, dict[str, str]]

    @classmethod
    def empty(cls) -> "SyncState":
        return cls(version=1, sources={}, digests={})


@dataclass(frozen=True)
class CollectReport:
    scanned_files: int
    changed_files: int
    accepted_messages: int
    written_notes: tuple[str, ...]
    bad_lines: int
    redactions: int

def load_state(path: Path) -> SyncState
def save_state(path: Path, state: SyncState) -> None
def archive_path(vault: Path, message: Message) -> Path
def merge_archive(path: Path, messages: list[Message]) -> bool
def collect_sessions(
    codex_root: Path,
    claude_root: Path,
    vault: Path,
    state_path: Path,
    since: datetime,
    dry_run: bool = False,
) -> CollectReport
```

Use YAML fields `platform`, `session_id`, `date`, `project`, and `generated_by`. Precede every message with `<!-- ai-session-sync:<id> ts=<ISO> -->`. Parse existing markers, merge new blocks, preserve old blocks after source truncation, sort by `(timestamp, id)`, and replace only when bytes differ. Sanitize filename components to ASCII letters, digits, dot, underscore, and hyphen. Group by `(platform, session_id, day)`.

Build the next state in memory. Write every changed note first, then atomically save state. State records source size, `mtime_ns`, and accepted IDs. `dry_run` performs discovery and parsing without creating directories, temporary files, Vault notes, or state.

- [ ] **Step 4: Verify GREEN and commit Task 3**

Run:

```bash
.venv/bin/python -m unittest test_ai_session_sync -v
/usr/bin/python3 -m py_compile ai_session_sync_core.py ai_session_sync_sources.py
git diff --check -- ai_session_sync_core.py test_ai_session_sync.py
git add ai_session_sync_core.py test_ai_session_sync.py
git commit -m '同步: 原子归档AI会话原文'
```

### Task 4: Source-linked hierarchical Hermes digest

**Files:**
- Create: `ai_session_sync_digest.py`
- Modify: `test_ai_session_sync.py`

**Interfaces:**
- Produces `DailySource`, `DigestReport`, `collect_daily_sources`, `chunk_sources`, `request_hermes`, `validate_digest`, and `digest_pending_days`.
- Reads only managed, redacted archives.

- [ ] **Step 1: Add failing digest tests**

Define `GOOD_DIGEST` with all eight approved headings and one allowed wikilink. Add `DigestTests` that assert:

```python
class DigestTests(unittest.TestCase):
    def test_collect_sources_ignores_unmanaged_markdown(self):
        # The fixture creates one note with generated_by: ai-session-sync and
        # one ordinary Markdown file. Only the managed note is returned.
        self.assertEqual([source.relative_link for source in sources], [
            "90-系统/AI会话/原文/2026-07-19/Codex-s1"
        ])

    def test_validate_requires_all_headings_and_known_links(self):
        allowed = {"90-系统/AI会话/原文/2026-07-19/Codex-s1"}
        validate_digest(GOOD_DIGEST, allowed)
        with self.assertRaises(ValueError):
            validate_digest(
                GOOD_DIGEST.replace("## 今日完成", "## 其他"), allowed
            )
        with self.assertRaises(ValueError):
            validate_digest(GOOD_DIGEST.replace("Codex-s1", "unknown", 1), allowed)
```

Complete the first test with temporary files before running. Add complete tests named:

- `test_chunking_keeps_each_session_source_whole`
- `test_unchanged_source_hash_skips_client_and_write`
- `test_hermes_failure_keeps_note_and_digest_state_unchanged`
- `test_output_requiring_redaction_is_rejected`
- `test_multiple_chunks_make_chunk_calls_then_one_merge_call`
- `test_one_failed_day_does_not_block_later_day`
- `test_digest_dry_run_never_calls_client_or_writes`

For the multi-chunk test, force two chunks and inject a counting client that returns two source-linked partial summaries followed by `GOOD_DIGEST`; assert exactly three calls. For failure tests, snapshot state and note bytes before the call and compare afterward.

- [ ] **Step 2: Run digest tests and verify RED**

Run:

```bash
.venv/bin/python -m unittest test_ai_session_sync.DigestTests -v
```

Expected: import failure for `ai_session_sync_digest`.

- [ ] **Step 3: Implement the Hermes layer**

Create:

```python
@dataclass(frozen=True)
class DailySource:
    relative_link: str
    content: str

@dataclass(frozen=True)
class DigestReport:
    checked_days: tuple[str, ...]
    updated_days: tuple[str, ...]
    skipped_days: tuple[str, ...]
    failed_days: tuple[str, ...]

def collect_daily_sources(vault: Path, day: str) -> tuple[DailySource, ...]
def chunk_sources(
    sources: tuple[DailySource, ...], max_chars: int = 60000,
) -> tuple[tuple[DailySource, ...], ...]
def request_hermes(prompt: str, timeout: int = 600) -> str
def validate_digest(text: str, allowed_links: set[str]) -> None
def digest_pending_days(
    vault: Path,
    state_path: Path,
    since_day: str,
    through_day: str,
    client=request_hermes,
    dry_run: bool = False,
) -> DigestReport
```

Read only notes under the selected day whose front matter declares `generated_by: ai-session-sync`. Hash sorted `(link, content)` pairs and skip unchanged days. Keep each source whole during chunking; truncate an individual over-limit source only in the Hermes material and label the truncation.

Read `API_SERVER_KEY` from `~/.hermes/.env` without logging it. Call `http://127.0.0.1:8642/v1/chat/completions` with model `auto`, temperature `0.2`, `max_tokens` 5000, and a 600-second timeout. Require the eight design headings. Every factual bullet except a literal `暂无` bullet must contain an allowed wikilink. Reject output if `redact_sensitive` reports a hit. Write the daily note atomically, then advance only that day's digest hash. Continue after a failed day.

- [ ] **Step 4: Verify GREEN and commit Task 4**

Run:

```bash
.venv/bin/python -m unittest test_ai_session_sync -v
/usr/bin/python3 -m py_compile \
  ai_session_sync_core.py ai_session_sync_sources.py ai_session_sync_digest.py
git diff --check -- ai_session_sync_digest.py test_ai_session_sync.py
git add ai_session_sync_digest.py test_ai_session_sync.py
git commit -m '同步: 用Hermes生成可追溯双脑日报'
```

### Task 5: CLI orchestration, process locking, and isolated end-to-end test

**Files:**
- Create: `ai_session_sync.py`
- Modify: `test_ai_session_sync.py`

**Interfaces:**
- Produces `SyncConfig`, `acquire_lock`, `run_collect`, `run_digest`, `run_backfill`, `build_parser`, and `main`.

- [ ] **Step 1: Add failing CLI tests**

Add `CliTests` with a temporary configuration:

```python
class CliTests(unittest.TestCase):
    def make_config(self):
        root = Path(tempfile.mkdtemp())
        return SyncConfig(
            codex_root=root / "codex",
            claude_root=root / "claude",
            vault=root / "vault",
            state_path=root / "state" / "state.json",
            lock_path=root / "state" / "run.lock",
            log_path=root / "sync.log",
        )

    def test_runtime_source_contains_no_git_publication_commands(self):
        source = Path("ai_session_sync.py").read_text(encoding="utf-8")
        for forbidden in ("git add", "git commit", "git pull", "git push"):
            self.assertNotIn(forbidden, source)
```

Add complete tests named:

- `test_collect_dry_run_reports_both_platforms_and_writes_nothing`
- `test_held_process_lock_skips_cleanly_without_writes`
- `test_backfill_collects_once_before_digesting_oldest_day_first`
- `test_invalid_date_is_rejected_before_filesystem_access`
- `test_end_to_end_two_platform_two_day_fixture_is_byte_idempotent`
- `test_log_contains_counts_but_no_message_or_secret_text`

The end-to-end test runs collect and a mocked digest twice, snapshots every Vault and state file after the first run, and asserts identical paths and bytes after the second.

- [ ] **Step 2: Run CLI tests and verify RED**

Run:

```bash
.venv/bin/python -m unittest test_ai_session_sync.CliTests -v
```

Expected: import failure for `ai_session_sync`.

- [ ] **Step 3: Implement CLI, defaults, lock, and logging**

Create:

```python
@dataclass(frozen=True)
class SyncConfig:
    codex_root: Path
    claude_root: Path
    vault: Path
    state_path: Path
    lock_path: Path
    log_path: Path

def acquire_lock(path: Path)
def run_collect(config: SyncConfig, since_day: str, dry_run: bool) -> int
def run_digest(
    config: SyncConfig, since_day: str, through_day: str, dry_run: bool,
) -> int
def run_backfill(config: SyncConfig, since_day: str, dry_run: bool) -> int
def build_parser() -> argparse.ArgumentParser
def main(argv: list[str] | None = None) -> int
```

Support:

```text
ai-session-sync collect [--since YYYY-MM-DD] [--dry-run]
ai-session-sync digest [--since YYYY-MM-DD] [--through YYYY-MM-DD] [--dry-run]
ai-session-sync backfill --since YYYY-MM-DD [--dry-run]
```

Default paths are `~/.codex/sessions`, `~/.claude/projects`, `~/obsidian-knowledge-base`, `~/.local/state/ai-session-sync/state.json`, `~/.local/state/ai-session-sync/run.lock`, and `~/Library/Logs/ai-session-sync.log`. Use `fcntl.flock(LOCK_EX | LOCK_NB)` and keep the descriptor open for the context lifetime. Log one compact line per run with mode, counts, duration, and status. Never log message text, Hermes output, environment values, or matched secret text.

- [ ] **Step 4: Verify new suite twice and Python 3.9 compatibility**

Run:

```bash
.venv/bin/python -m unittest test_ai_session_sync -v
.venv/bin/python -m unittest test_ai_session_sync -v
/usr/bin/python3 -m py_compile \
  ai_session_sync.py ai_session_sync_core.py \
  ai_session_sync_sources.py ai_session_sync_digest.py
```

Expected: both runs pass with identical counts; compilation succeeds.

- [ ] **Step 5: Run mandatory project regressions**

Run:

```bash
.venv/bin/python test_rules.py && \
.venv/bin/python test_rules_mid.py && \
.venv/bin/python test_web.py && \
.venv/bin/python test_regime.py
```

Expected: all existing suites pass.

- [ ] **Step 6: Commit Task 5**

Run:

```bash
git diff --check -- ai_session_sync.py test_ai_session_sync.py
git add ai_session_sync.py test_ai_session_sync.py
git commit -m '同步: 编排AI会话增量与回填'
```

### Task 6: Idempotent installer and launchd schedules

**Files:**
- Create: `install_ai_session_sync.sh`
- Modify: `test_ai_session_sync.py`

**Interfaces:**
- Installs runtime modules under `~/.local/lib/ai-session-sync/`.
- Installs executable `~/.local/bin/ai-session-sync`.
- Installs collector and digest LaunchAgents.

- [ ] **Step 1: Add failing installer tests**

Add `InstallerTests` with this source-safety assertion:

```python
class InstallerTests(unittest.TestCase):
    def test_installer_does_not_assign_home_or_touch_screener_service(self):
        text = Path("install_ai_session_sync.sh").read_text(encoding="utf-8")
        self.assertNotRegex(text, r"(?m)^HOME=")
        self.assertNotIn("com.rui.stock-screener", text)
        self.assertNotIn("obsidian_sync.py", text)
```

Add complete tests named:

- `test_dry_run_with_isolated_home_writes_nothing`
- `test_install_with_launchctl_disabled_creates_valid_plists`
- `test_collector_interval_is_900_seconds`
- `test_digest_slots_are_0550_1150_1750_2350`
- `test_second_install_is_byte_idempotent`
- `test_wrapper_uses_system_python_and_installed_library`

Each test uses an isolated temporary `HOME`, passes the checkout through `AI_SYNC_SOURCE`, and sets `AI_SYNC_SKIP_LAUNCHCTL=1`. Parse plists with `plistlib`; do not rely only on XML string matching.

- [ ] **Step 2: Run installer tests and verify RED**

Run:

```bash
.venv/bin/python -m unittest test_ai_session_sync.InstallerTests -v
```

Expected: `FileNotFoundError` for `install_ai_session_sync.sh`.

- [ ] **Step 3: Implement the installer**

Create a Bash script with `set -euo pipefail` and these task-specific variables:

```bash
USER_ROOT="${HOME:?HOME is required}"
SOURCE_ROOT="${AI_SYNC_SOURCE:-$USER_ROOT/云慧养/zmu_gitee/screener}"
RUNTIME_LIB="$USER_ROOT/.local/lib/ai-session-sync"
RUNTIME_BIN="$USER_ROOT/.local/bin/ai-session-sync"
LAUNCH_DIR="$USER_ROOT/Library/LaunchAgents"
LOG_PATH="$USER_ROOT/Library/Logs/ai-session-sync.log"
```

Required behavior:

- `--dry-run` prints target paths and writes nothing.
- Copy all four `ai_session_sync*.py` runtime modules to `RUNTIME_LIB`.
- Write a wrapper that executes `/usr/bin/python3 "$RUNTIME_LIB/ai_session_sync.py" "$@"`.
- Collector label is `com.harryd317.ai-session-collect`; arguments are wrapper and `collect`; `RunAtLoad=true`; `StartInterval=900`.
- Digest label is `com.harryd317.ai-session-digest`; arguments are wrapper, `digest`, `--since`, and `2026-06-20`; calendar slots are 05:50, 11:50, 17:50, and 23:50.
- Both jobs send stdout and stderr to `LOG_PATH` and set PATH to `/usr/bin:/bin:/usr/sbin:/sbin`.
- Validate plists with `plutil -lint` when `plutil` exists.
- Skip launchctl only when `AI_SYNC_SKIP_LAUNCHCTL=1`.
- Otherwise boot out the two exact labels, bootstrap the two exact files, and kickstart only the collector.
- Never modify existing Vault autosync, Vault review, screener sync, or screener web-service jobs.

- [ ] **Step 4: Verify GREEN and commit Task 6**

Run:

```bash
.venv/bin/python -m unittest test_ai_session_sync -v
bash -n install_ai_session_sync.sh
git diff --check -- install_ai_session_sync.sh test_ai_session_sync.py
git add install_ai_session_sync.sh test_ai_session_sync.py
git commit -m '同步: 安装AI会话归档定时任务'
```

### Task 7: Production dry run, 30-day backfill, activation, and evidence

**Files:**
- Create: `docs/交付总结-AI会话双脑同步-20260719.md`
- Create: `~/obsidian-knowledge-base/90-系统/AI会话/README.md`
- Install runtime files under `~/.local/` and LaunchAgents under `~/Library/LaunchAgents/`.

**Interfaces:**
- Produces a verified installation, 30-day archives, Hermes daily notes, logs, and recovery evidence.

- [ ] **Step 1: Re-run all automated verification before production writes**

Run:

```bash
.venv/bin/python -m unittest test_ai_session_sync -v
/usr/bin/python3 -m py_compile \
  ai_session_sync.py ai_session_sync_core.py \
  ai_session_sync_sources.py ai_session_sync_digest.py
bash -n install_ai_session_sync.sh
.venv/bin/python test_rules.py && \
.venv/bin/python test_rules_mid.py && \
.venv/bin/python test_web.py && \
.venv/bin/python test_regime.py
```

Expected: zero failures and zero syntax errors.

- [ ] **Step 2: Run production dry-run and record counts**

Run:

```bash
.venv/bin/python ai_session_sync.py backfill --since 2026-06-20 --dry-run
git -C '~/obsidian-knowledge-base' status --short
```

Expected: both platforms, main-session counts, accepted-message counts, bad-line counts, redaction counts, date range, and pending digest days are reported. No Vault or state files are written. Record exact counts in the delivery summary.

- [ ] **Step 3: Create the Vault README**

Use `apply_patch` to create the README with this exact content:

```markdown
# AI 会话共同档案

本目录由 `ai-session-sync` 管理，归档 Codex 和 Claude 主会话中用户可见的文字。

- `原文/`：按 Asia/Shanghai 日期分组的脱敏对话，只读。
- `20-AI复核/YYYY-MM-DD-AI双脑日报.md`：Hermes 生成的每日接力摘要。
- 系统不归档隐藏推理、系统指令、工具输出、子代理日志或二进制附件。
- 不在 `原文/` 中手工编辑；修正请另建笔记并链接原文。
```

- [ ] **Step 4: Run the 30-day source backfill**

Run:

```bash
.venv/bin/python ai_session_sync.py collect --since 2026-06-20
```

Verify nonzero note counts for both platforms:

```bash
find '~/obsidian-knowledge-base/90-系统/AI会话/原文' \
  -type f -name '*.md' | wc -l
rg -l '^platform: codex$' \
  '~/obsidian-knowledge-base/90-系统/AI会话/原文' | wc -l
rg -l '^platform: claude$' \
  '~/obsidian-knowledge-base/90-系统/AI会话/原文' | wc -l
```

- [ ] **Step 5: Sample content and scan secrets before Hermes**

Choose three Codex and three Claude output files across different days. Compare visible roles and order with source JSONL without printing source content to logs. Confirm system/developer content, tool calls/results, source JSONL, Base64 payloads, and hidden reasoning are absent. Confirm redaction markers replace credentials.

Run the production scanner using the same named patterns as `redact_sensitive` across all new notes. It must report zero unredacted matches. Record only filenames, pattern names, and pass/fail results in the delivery summary.

- [ ] **Step 6: Backfill Hermes daily notes and resume failures**

Run:

```bash
.venv/bin/python ai_session_sync.py digest \
  --since 2026-06-20 --through 2026-07-19
```

If a day fails, inspect `~/Library/Logs/ai-session-sync.log`, reproduce the failure in a new test, fix it through TDD, and rerun the same range. Never advance digest state manually. Verify every day with archived messages has a daily note; days without messages must appear as skipped.

- [ ] **Step 7: Prove production idempotence**

Capture checksums of managed archives, daily notes, and state. Run the same collect and digest commands again. Recompute checksums and run:

```bash
git -C '~/obsidian-knowledge-base' diff --check
```

Expected: zero written archives, zero updated digest days, matching checksums, and no second-run diff.

- [ ] **Step 8: Install and load launchd jobs**

Run:

```bash
bash install_ai_session_sync.sh
plutil -lint \
  '~/Library/LaunchAgents/com.harryd317.ai-session-collect.plist' \
  '~/Library/LaunchAgents/com.harryd317.ai-session-digest.plist'
launchctl print 'gui/501/com.harryd317.ai-session-collect'
launchctl print 'gui/501/com.harryd317.ai-session-digest'
```

Expected: both plists are valid and loaded; the collector's most recent exit code is 0.

- [ ] **Step 9: Verify one live incremental append**

After this Codex session receives one new visible message, run:

```bash
launchctl kickstart -k 'gui/501/com.harryd317.ai-session-collect'
```

Confirm exactly one new marker appears in today's Codex archive. A second kickstart must create no duplicate. Confirm the log contains counts but no message text.

- [ ] **Step 10: Publish through the existing Vault job**

Run:

```bash
launchctl kickstart -k 'gui/501/com.harryd317.vault-autosync'
git -C '~/obsidian-knowledge-base' fetch origin main
git -C '~/obsidian-knowledge-base' \
  rev-list --left-right --count origin/main...main
```

Expected: `0 0`. If publication fails, leave the Vault worktree intact and repair the existing publisher separately; never advance state by hand.

- [ ] **Step 11: Write delivery evidence and run final verification**

Create `docs/交付总结-AI会话双脑同步-20260719.md` with exact test counts, dry-run counts, date range, note counts by platform, redaction count, malformed-line count, Hermes day results, idempotence checksums, launchd labels, log path, recovery commands, and known limitations.

Run:

```bash
.venv/bin/python -m unittest test_ai_session_sync -v
.venv/bin/python test_rules.py && \
.venv/bin/python test_rules_mid.py && \
.venv/bin/python test_web.py && \
.venv/bin/python test_regime.py
git diff --check
git status --short
```

Stage only the AI session-sync implementation, tests, installer, plan/spec, and delivery summary. Leave all pre-existing untracked files untouched. Commit:

```bash
git add 'docs/交付总结-AI会话双脑同步-20260719.md'
git commit -m '交付: Codex与Claude会话经Hermes同步Obsidian'
```
