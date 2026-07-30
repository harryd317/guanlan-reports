# Global AI Handoff Loop Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Upgrade the existing Codex/Claude-to-Obsidian collector into a global, automatically classified, privacy-preserving, source-linked handoff loop whose eight stages are independently receipted and whose Codex → Claude → Codex continuation is proven end to end.

**Architecture:** Keep platform JSONL files as the only raw source. A deterministic source boundary emits visible main-session messages; a fail-closed privacy boundary produces safe messages; a rule-first router groups safe turn segments into projects; archive and handoff writers publish one sanitized original plus project-linked derivatives. A local receipt ledger hashes every stage, the existing Vault sync remains the only Git publisher, and explicit global Codex/Claude instructions invoke a handoff reader that records authoritative consumption before later feedback closes the cycle.

**Tech Stack:** Python 3.9+ standard library, `unittest`, local Hermes OpenAI-compatible HTTP API with tools disabled, Bash, launchd, Obsidian Markdown, Git, Codex global `AGENTS.md`, and Claude Code `SessionStart` hooks.

## Global Constraints

- Implement `docs/superpowers/specs/2026-07-21-global-ai-handoff-loop-design.md` and preserve the safety properties of `docs/superpowers/specs/2026-07-19-ai-session-sync-design.md`.
- Start implementation in an isolated worktree created from `main`; do not modify either existing worktree in place.
- Merge `codex/ai-session-sync` as the tested baseline before extending it. Its current acceptance baseline is 50 passing tests under `/usr/bin/python3 test_ai_session_sync.py`.
- Read raw Codex and Claude JSONL only from their original locations. Never copy raw JSONL, hidden reasoning, tool traffic, attachments, platform instructions, or subagent content into state, Vault, logs, Hermes prompts, or Git.
- Preserve the privacy order: parse visible main-session content → sanitize → independent audit → classify → archive. No classifier or writer may accept a raw-message type.
- Use separate implementations for redaction and post-redaction audit. A defect in one rule set must remain detectable by the other.
- Treat exact identity and financial data as private even when they are not credentials. Preserve stock codes, prices, cost bases, stop lines, percentages, rules, decisions, and evidence.
- Use `ZoneInfo("Asia/Shanghai")` for all dates, idle windows, fallback schedules, receipts, and status rendering.
- Keep runtime compatible with macOS `/usr/bin/python3` 3.9.6 and the Python standard library. Serialize `项目注册表.yaml` as canonical JSON syntax, which is valid YAML 1.2, so runtime does not depend on PyYAML.
- Use atomic sibling-file replacement with `fsync` for managed Markdown, registry, state, and pointer files. Append-only receipt files must lock, flush, and `fsync` before returning success.
- Do not establish a second Git publisher. Extend `screener-vault-sync` so it also publishes dirty AI-managed Vault paths and then calls the receipt observer.
- Any failed required stage returns a nonzero exit code, leaves the previous valid artifact in place, and does not write the next-stage receipt.
- `--dry-run` must not create locks, keys, logs, state, quarantine records, Vault files, launchd files, global instruction blocks, or Git commits.
- Do not change or restart the screener web service, trading rules, strategy parameters, production databases, or trading schedules. The four project regression suites remain mandatory after Python changes.
- Do not permanently delete prior AI-sync artifacts during migration. Move them into a timestamped, user-recoverable backup and require a production activation checkpoint before replacing managed Vault directories.
- Use managed begin/end markers when editing global instruction files. If non-empty `~/.codex/AGENTS.override.md` exists, update that active global file; otherwise update `~/.codex/AGENTS.md`. Preserve all content outside the managed block.
- Claude hooks receive authoritative `session_id`, `transcript_path`, and `cwd` on stdin. The hook may bind identity but must emit no handoff body into `SessionStart` context.
- Codex consumption requires the authoritative `CODEX_THREAD_ID`. Missing or ambiguous identity may show paths, but must not create a `consumed` receipt.
- Keep commits task-scoped. Run the task’s focused test before each commit and never claim completion without fresh verification output.

---

## Planned File Structure

### Runtime modules

- Modify `ai_session_sync.py`: top-level CLI, locks, orchestration, catch-up, failure codes, and structured log counters.
- Modify `ai_session_sync_core.py`: shared time/hash/atomic-I/O primitives, safe state loading, and compatibility exports.
- Modify `ai_session_sync_sources.py`: Codex/Claude main-session discovery and visible-message parsing only.
- Create `ai_session_sync_models.py`: raw, sanitized, classified, project, receipt, and report dataclasses with no I/O.
- Create `ai_session_sync_privacy.py`: S0/S1/S2/S3 sanitization, HMAC pseudonyms, amount bands, quantity hiding, and privacy policy versioning.
- Create `ai_session_sync_audit.py`: independently authored post-sanitization scanner and quarantine metadata writer.
- Create `ai_session_sync_projects.py`: project registry, Git metadata probing, deterministic routing, aliases, dynamic creation evidence, and last-known-good registry recovery.
- Create `ai_session_sync_classify.py`: turn segmentation, Hermes fallback classification, low-confidence queue, nightly reconciliation, and content-type validation.
- Create `ai_session_sync_hermes.py`: tool-safety preflight plus validated text/JSON calls shared by classification and summarization.
- Create `ai_session_sync_archive.py`: one-copy sanitized originals, stable message anchors, classification index, and archive receipts.
- Modify `ai_session_sync_digest.py`: project-scoped daily digest generation and source-link validation.
- Create `ai_session_sync_handoff.py`: rolling current handoff, global overview, freshness/rate-limit state, and output validation.
- Create `ai_session_sync_ledger.py`: eight-stage receipt ledger, failure counters, status derivation, status-page rendering, and publish observation.
- Create `ai_session_handoff.py`: Claude identity binding, project resolution, startup catch-up, pointer manifest creation, and read receipts.
- Create `ai_session_sync_migrate.py`: recoverable backup, historical safety scan, temporary rebuild, activation, and rollback.

### Publisher and installation

- Modify `obsidian_sync.py`: propagate staged failures, publish dirty AI-managed paths, and invoke the publish observer.
- Modify `obsidian_sync_core.py`: return structured publish metadata and expose restricted managed-path detection.
- Modify `install_obsidian_sync.sh`: install the publisher integration without assigning to `HOME` or swallowing observer failures.
- Modify `install_ai_session_sync.sh`: install all modules and both CLIs, create launchd jobs, merge global instruction blocks, and merge the Claude hook.

### Focused tests

- Keep `test_ai_session_sync.py` as the original 50-test regression suite, updating imports only when a compatibility boundary intentionally changes.
- Create `test_ai_session_sync_sources.py`.
- Create `test_ai_session_sync_privacy.py`.
- Create `test_ai_session_sync_projects.py`.
- Create `test_ai_session_sync_classify.py`.
- Create `test_ai_session_sync_archive.py`.
- Create `test_ai_session_sync_handoff.py`.
- Create `test_ai_session_sync_ledger.py`.
- Create `test_ai_session_sync_migrate.py`.
- Create `test_ai_session_sync_e2e.py`.
- Create `test_obsidian_sync.py`.

### Documentation

- Modify `README.md`, `AGENTS.md`, and `MEMORY.md` with operator commands, boundaries, recovery steps, and verified delivery facts.
- Create `docs/交付总结-全局AI接力闭环-20260721.md` with test evidence, migration evidence, hashes, receipts, and remaining operational caveats.

---

## Stable Contracts Used Across Tasks

Implement the following type boundary first and keep later tasks consistent with it:

```python
@dataclass(frozen=True)
class RawMessage:
    platform: str
    session_id: str
    message_id: str
    timestamp: datetime
    role: str
    text: str
    cwd: str
    source_path: str

@dataclass(frozen=True)
class SafeMessage:
    raw_id: str
    platform: str
    session_id: str
    message_id: str
    timestamp: datetime
    role: str
    text: str
    cwd: str
    sensitivity: str
    redaction_kinds: Tuple[str, ...]

@dataclass(frozen=True)
class Classification:
    primary_category: str
    project_id: str
    topic_tags: Tuple[str, ...]
    content_type: str
    sensitivity: str
    confidence: float
    classification_source: str
    scope: str

@dataclass(frozen=True)
class ClassifiedSegment:
    segment_id: str
    messages: Tuple[SafeMessage, ...]
    classification: Classification

@dataclass(frozen=True)
class Receipt:
    receipt_id: str
    cycle_id: str
    project_id: str
    stage: str
    input_hash: str
    artifact_hash: str
    parent_handoff_hash: str
    created_at: str
    metadata: Dict[str, str]
```

`RawMessage` may be consumed only by `ai_session_sync_privacy.sanitize_message`. Hermes, project routing, archive, handoff, and ledger artifact functions accept only `SafeMessage` or `ClassifiedSegment`.

---

### Task 1: Integrate the tested collector baseline

**Files:**
- Merge from branch: `codex/ai-session-sync`
- Verify: `ai_session_sync.py`
- Verify: `ai_session_sync_core.py`
- Verify: `ai_session_sync_sources.py`
- Verify: `ai_session_sync_digest.py`
- Verify: `install_ai_session_sync.sh`
- Verify: `test_ai_session_sync.py`

**Interfaces:**
- Preserve the existing `collect`, `digest`, and `backfill` commands until replacement commands have regression coverage.
- Preserve the existing 50-test baseline before refactoring.

- [ ] **Step 1: Create the implementation worktree**

Invoke `superpowers:using-git-worktrees`, then create a branch named `codex/global-ai-handoff-loop` from `main`. Confirm the new worktree starts clean:

```bash
git status --short --branch
```

Expected: the new branch name and no changed paths.

- [ ] **Step 2: Merge the existing implementation history**

```bash
git merge --no-ff codex/ai-session-sync -m "集成: 合入AI会话同步安全基线"
```

Expected: a merge commit that adds the four runtime modules, installer, original tests, and prior delivery note while retaining the 2026-07-21 design.

- [ ] **Step 3: Re-run the exact baseline**

```bash
/usr/bin/python3 test_ai_session_sync.py
```

Expected:

```text
Ran 50 tests

OK
```

- [ ] **Step 4: Confirm the merge did not touch trading behavior**

```bash
git diff main...HEAD -- screener.py rules_mid.py params.py storage.py service.py
```

Expected: no output.

The merge commit created in Step 2 is this task’s commit.

---

### Task 2: Enforce the main-session and visible-content source boundary

**Files:**
- Create: `ai_session_sync_models.py`
- Modify: `ai_session_sync_sources.py`
- Modify: `ai_session_sync_core.py`
- Create: `test_ai_session_sync_sources.py`
- Modify: `test_ai_session_sync.py`

**Interfaces:**
- `discover_session_files(...) -> Tuple[DiscoveredSession, ...]`
- `parse_codex_session(path: Path) -> ParseReport`
- `parse_claude_session(path: Path) -> ParseReport`
- `strip_platform_envelopes(text: str) -> str`
- `ParseReport` includes `main_session`, `messages`, `bad_lines`, and `excluded_counts`.

- [ ] **Step 1: Write failing source-boundary tests**

Add fixtures that assert all of the following:

```python
class SourceBoundaryTests(unittest.TestCase):
    def test_codex_session_meta_with_subagent_is_rejected_as_a_whole(self):
        report = parse_codex_session(self.codex_subagent)
        self.assertFalse(report.main_session)
        self.assertEqual(report.messages, ())
        self.assertEqual(report.excluded_counts["subagent_session"], 1)

    def test_codex_strips_managed_envelopes_but_keeps_the_real_request(self):
        report = parse_codex_session(self.codex_dynamic_context)
        self.assertEqual(report.messages[0].text, "请修复同步失败")

    def test_claude_excludes_meta_tool_thinking_and_attachment_blocks(self):
        report = parse_claude_session(self.claude_mixed_blocks)
        self.assertEqual(tuple(item.text for item in report.messages), ("可见问题", "可见回答"))
        self.assertEqual(report.excluded_counts["meta_record"], 1)
        self.assertEqual(report.excluded_counts["non_text_block"], 3)
```

Also test a Codex file in a subagent directory, a Claude `subagents/` path, system/developer roles, collaboration events, tool results, a message made entirely of platform envelopes, malformed middle lines, and an incomplete final line.

- [ ] **Step 2: Run the focused tests and confirm failure**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_sources.py
```

Expected: failures because `main_session`, `excluded_counts`, Codex subagent metadata checks, and envelope stripping do not exist.

- [ ] **Step 3: Add immutable raw-message and parse-report models**

In `ai_session_sync_models.py`, implement `RawMessage`, `DiscoveredSession`, and `ParseReport`. `ParseReport.excluded_counts` must be a copied dictionary so callers cannot mutate parser state. Keep the existing `ai_session_sync_core.Message` dataclass and its parser/archive call sites unchanged for this task so the 50-test baseline remains runnable; Task 3 migrates those call sites atomically from legacy `Message` to `RawMessage` and `SafeMessage`.

- [ ] **Step 4: Preflight the whole session before accepting messages**

For Codex, inspect all `session_meta` rows before emitting any message. Reject the complete file when `payload.source.subagent` exists or is non-empty. For Claude, reject paths containing a `subagents` component. Do not rely on filename patterns alone.

Only accept:

- Codex `response_item.payload.type == "message"`, role `user` or `assistant`, and `input_text`/`output_text` blocks.
- Claude rows whose type is `user` or `assistant`, `isMeta` is not true, and whose message content contains `text` blocks.

Count every deterministic exclusion by category without recording excluded text.

- [ ] **Step 5: Remove only recognized managed envelopes**

Implement balanced removal for these complete XML-style blocks when they appear in a user message: `recommended_plugins`, `environment_context`, `permissions instructions`, `app-context`, `apps_instructions`, `plugins_instructions`, `skills_instructions`, and `developer`. Preserve text before, between, and after those blocks. If an opening tag has no closing tag, exclude that block and record `malformed_platform_envelope`; do not archive its remainder.

- [ ] **Step 6: Preserve incomplete-source retry behavior**

Return bad line numbers separately. Mark an unterminated final JSON line as retryable so the collector does not record that byte extent as complete. A later append that completes the row must yield exactly one new message.

- [ ] **Step 7: Run source and legacy regressions**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_sources.py test_ai_session_sync.py
```

Expected: all source-boundary tests pass and the original 50 tests remain green.

- [ ] **Step 8: Commit**

```bash
git add ai_session_sync_models.py ai_session_sync_sources.py ai_session_sync_core.py test_ai_session_sync_sources.py test_ai_session_sync.py
git commit -m "安全: 只接收Codex与Claude可见主会话"
```

---

### Task 3: Build the fail-closed S0–S3 privacy boundary

**Files:**
- Create: `ai_session_sync_privacy.py`
- Create: `ai_session_sync_audit.py`
- Modify: `ai_session_sync_models.py`
- Modify: `ai_session_sync_sources.py`
- Modify: `ai_session_sync_core.py`
- Modify: `ai_session_sync.py`
- Create: `test_ai_session_sync_privacy.py`
- Modify: `test_ai_session_sync.py`

**Interfaces:**
- `sanitize_message(raw: RawMessage, key: bytes) -> SanitizationResult`
- `load_pseudonym_key(path: Path, create: bool) -> bytes`
- `audit_safe_text(text: str) -> AuditResult`
- `write_quarantine_record(root: Path, raw: RawMessage, findings: Sequence[str]) -> Path`
- `SanitizationResult.safe_message` is `None` when the independent audit fails.

- [ ] **Step 1: Write failing privacy tests from the approved policy**

Cover credentials, identities, finance, preserved decision data, and isolation:

```python
class PrivacyTests(unittest.TestCase):
    def test_server_chan_sendkeys_and_cookie_are_fully_removed(self):
        text = "SENDKEY=SCT1234567890abcdefghijkl Cookie: sid=abcdefghijklmno"
        result = sanitize_text(text, self.key)
        self.assertNotIn("SCT1234567890", result.text)
        self.assertNotIn("sid=", result.text)

    def test_identity_alias_is_stable_hmac_not_plain_hash(self):
        first = sanitize_text("邮箱 trader@example.com", self.key).text
        second = sanitize_text("邮箱 trader@example.com", self.key).text
        self.assertEqual(first, second)
        self.assertRegex(first, r"邮箱账户-[0-9A-F]{4}")

    def test_finance_is_banded_but_prices_and_percentages_survive(self):
        text = "总资产 1,234,567 元，持仓市值 300,000 元，现价 38.03，成本价 91.60，止损 35.20，仓位 24.3%"
        safe = sanitize_text(text, self.key).text
        self.assertIn("100万–300万元", safe)
        self.assertIn("仓位约24.3%", safe)
        self.assertIn("现价 38.03", safe)
        self.assertIn("成本价 91.60", safe)
        self.assertIn("止损 35.20", safe)

    def test_independent_audit_catches_a_redactor_miss_and_quarantines_metadata_only(self):
        raw = self.raw("Authorization\tBearer\tmissed-secret-value-123456789")
        result = sanitize_message(raw, self.key)
        self.assertIsNone(result.safe_message)
        record = json.loads(write_quarantine_record(self.quarantine, raw, result.audit_findings).read_text())
        self.assertNotIn(raw.text, json.dumps(record))
```

Add tests for OpenAI/Anthropic/OpenRouter-style keys, Server酱 `SCT` and `sctp` forms, webhooks, passwords, credentials, PEM/OpenSSH keys, high-entropy strings, Base64, phone, email, bank card, ID card, broker account, recognizable account name, exact cash, buying power, P&L, market value, option quantity, missing denominator, and cross-currency values.

- [ ] **Step 2: Run the privacy suite and confirm failure**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_privacy.py
```

Expected: import errors and assertion failures because the privacy and audit modules do not exist.

- [ ] **Step 3: Implement S0 credential deletion**

Define explicit `CredentialRule` entries in `ai_session_sync_privacy.py`. Replace the entire matched value with a category marker. Include named assignments containing `API_KEY`, `TOKEN`, `SECRET`, `PASSWORD`, `PASSWD`, `SENDKEY`, or `CREDENTIAL`; authorization headers; cookies; private keys; webhooks; provider-key prefixes; high-entropy tokens; and long Base64.

Never include a matched prefix or suffix in the replacement. Return only rule category names in reports.

- [ ] **Step 4: Implement S1 HMAC pseudonyms**

Generate 32 random bytes with `secrets.token_bytes(32)` only when `create=True`; write through a `0600` file descriptor and verify the final mode. Derive aliases with:

```python
def pseudonym(kind: str, value: str, key: bytes) -> str:
    normalized = unicodedata.normalize("NFKC", value).strip().casefold()
    digest = hmac.new(key, (kind + "\x1f" + normalized).encode("utf-8"), hashlib.sha256)
    return "{}-{}".format(kind, digest.hexdigest()[:4].upper())
```

Map supported identities to Chinese alias kinds. Never write the HMAC key or original identity to state, logs, Vault, or test failure messages.

- [ ] **Step 5: Implement S2 financial coarsening and S3 preservation**

Use the ten approved amount bands. Parse currency and labeled account totals first. Convert a same-currency position market value to a percentage only when a reliable total exists in the same message segment. Convert P&L to return only when a compatible denominator is present. Otherwise retain the currency plus band. Replace exact security/option/fund quantities with `[数量已隐藏]`.

Protect values attached to `现价`, `成本价`, `买点`, `止损`, `均线`, `目标价`, `仓位`, `收益率`, `回撤率`, or `风险预算` before applying amount rules, then restore them byte-for-byte.

- [ ] **Step 6: Implement the independent audit and quarantine**

Author `AUDIT_RULES` independently in `ai_session_sync_audit.py`; do not import redaction regexes. Audit every sanitized message and every later Hermes output. If S0, unaliased S1, or forbidden exact S2 data remains, write only this schema under a `0700` directory:

```json
{
  "platform": "codex",
  "session_id": "session-id",
  "message_id": "message-id",
  "finding_categories": ["authorization_token"],
  "source_path": "/local/source/path",
  "policy_version": "2",
  "created_at": "2026-07-21T12:00:00+08:00"
}
```

The local source path is allowed only in quarantine metadata. Logs and notifications must omit it.

- [ ] **Step 7: Move sanitization out of source parsing**

Make source adapters return `RawMessage`. Make the collector transform each raw message to `SafeMessage`, then discard the raw object before routing. Remove the legacy `Message` dataclass only after its archive call sites accept `SafeMessage`. Keep a compatibility `redact_sensitive(text)` wrapper for digest regression tests, implemented through the new privacy policy with an injected deterministic test key.

For `--dry-run`, use an in-memory key if the real key does not exist and clearly mark aliases as preview output; do not create the key.

- [ ] **Step 8: Prove no downstream raw-type access**

Add an AST-based test that fails if `ai_session_sync_classify.py`, `ai_session_sync_archive.py`, `ai_session_sync_digest.py`, `ai_session_sync_handoff.py`, or `ai_session_sync_hermes.py` imports `RawMessage` after those files exist. Until then, check all existing downstream modules.

- [ ] **Step 9: Run privacy, source, and original tests**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_privacy.py test_ai_session_sync_sources.py test_ai_session_sync.py
```

Expected: all tests pass; log-capture assertions contain categories and counts but no fixture secrets, exact identities, or exact financial values.

- [ ] **Step 10: Commit**

```bash
git add ai_session_sync_privacy.py ai_session_sync_audit.py ai_session_sync_models.py ai_session_sync_sources.py ai_session_sync_core.py ai_session_sync.py test_ai_session_sync_privacy.py test_ai_session_sync.py
git commit -m "隐私: 建立分级脱敏与独立隔离闸门"
```

---

### Task 4: Implement the project registry and deterministic router

**Files:**
- Create: `ai_session_sync_projects.py`
- Modify: `ai_session_sync_models.py`
- Modify: `ai_session_sync_core.py`
- Create: `test_ai_session_sync_projects.py`

**Interfaces:**
- `load_registry(vault_path: Path, fallback_path: Path) -> RegistryLoadResult`
- `save_registry(path: Path, fallback_path: Path, registry: ProjectRegistry) -> None`
- `probe_git_context(cwd: Path) -> GitContext`
- `route_deterministic(message: SafeMessage, context: GitContext, registry: ProjectRegistry) -> Optional[Classification]`
- `observe_project_evidence(...) -> RegistryDecision`

- [ ] **Step 1: Write failing registry and routing tests**

Test the fixed categories, canonical paths, exact priority, creation, aliasing, and fallback:

```python
class ProjectRegistryTests(unittest.TestCase):
    def test_same_git_root_wins_before_alias_or_semantics(self):
        route = route_deterministic(self.safe("讨论别的主题"), self.screener_git, self.registry)
        self.assertEqual(route.project_id, "screener")
        self.assertEqual(route.confidence, 1.0)
        self.assertEqual(route.classification_source, "git_root")

    def test_new_git_root_creates_one_stable_project(self):
        first = observe_project_evidence(self.registry, self.new_git_evidence)
        second = observe_project_evidence(first.registry, self.new_git_evidence)
        self.assertEqual(len(second.registry.projects), len(first.registry.projects))

    def test_corrupt_registry_uses_last_good_and_disables_creation(self):
        result = load_registry(self.corrupt, self.last_good)
        self.assertTrue(result.degraded)
        self.assertFalse(result.allow_project_creation)
```

Also test Git worktree `.git` files, normalized SSH/HTTPS remotes, `$HOME` path rendering, exact aliases, duplicate remotes, canonical ID collisions, the six fixed top categories, and registry byte idempotence.

- [ ] **Step 2: Run the focused tests and confirm failure**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_projects.py
```

Expected: import errors because project registry interfaces do not exist.

- [ ] **Step 3: Implement canonical registry storage**

Represent `项目注册表.yaml` using sorted, indented JSON with a terminal newline. JSON is valid YAML 1.2 and provides a strict standard-library parser. Store version, six categories, projects, aliases, roots, remotes, `created_by`, and creation evidence. Convert paths below the user home to `$HOME/...` on write and expand only a leading `$HOME/` on read.

Before replacing the Vault registry, write the same validated bytes to the local last-known-good path. On parse or schema failure, return the local copy with `degraded=True`, disable creation/merge, and cause the caller to return nonzero after completing safe read-only routing.

- [ ] **Step 4: Probe Git identity without shell parsing**

Walk parent directories to find a `.git` directory or worktree pointer file. Use `subprocess.run(["git", "-C", root, "config", "--get", "remote.origin.url"], ...)` with an argument list, five-second timeout, captured output, and no shell. Normalize equivalent SSH and HTTPS remotes to a host/path identity while retaining a safe display value.

- [ ] **Step 5: Implement deterministic priority**

Apply exact Git root, exact remote, exact registered cwd, registered alias, then platform project metadata. A deterministic hit produces confidence `1.0` and cannot be overridden by Hermes. New Git roots create a stable slug immediately; same root or remote merges into the existing record.

Store semantic project candidates but do not create them in this task.

- [ ] **Step 6: Run focused and privacy regressions**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_projects.py test_ai_session_sync_privacy.py
```

Expected: all tests pass and registry files contain no absolute username, raw identity, or exact finance fixture.

- [ ] **Step 7: Commit**

```bash
git add ai_session_sync_projects.py ai_session_sync_models.py ai_session_sync_core.py test_ai_session_sync_projects.py
git commit -m "分类: 建立项目注册表与确定性路由"
```

---

### Task 5: Add turn segmentation, Hermes fallback, and anti-fragmentation

**Files:**
- Create: `ai_session_sync_hermes.py`
- Create: `ai_session_sync_classify.py`
- Modify: `ai_session_sync_digest.py`
- Modify: `ai_session_sync_projects.py`
- Modify: `ai_session_sync_models.py`
- Create: `test_ai_session_sync_classify.py`
- Modify: `test_ai_session_sync.py`

**Interfaces:**
- `segment_turns(messages: Sequence[SafeMessage]) -> Tuple[TurnSegment, ...]`
- `classify_segment(segment, registry, client) -> ClassificationDecision`
- `request_hermes_json(prompt: str, schema_name: str) -> Dict[str, object]`
- `reconcile_pending(queue, registry, client, now) -> ReconcileReport`

- [ ] **Step 1: Write failing segmentation and classification tests**

Include these contract tests:

```python
class ClassificationTests(unittest.TestCase):
    def test_contiguous_user_and_assistant_messages_form_one_turn(self):
        segments = segment_turns((self.user_a, self.assistant_a, self.user_b, self.assistant_b))
        self.assertEqual(tuple(len(item.messages) for item in segments), (2, 2))

    def test_deterministic_route_never_calls_hermes(self):
        client = Mock(side_effect=AssertionError("Hermes must not run"))
        decision = classify_segment(self.screener_segment, self.registry, client)
        self.assertEqual(decision.classification.project_id, "screener")

    def test_low_confidence_hermes_result_enters_temporary_queue(self):
        decision = classify_segment(self.unknown_segment, self.registry, self.hermes(0.84))
        self.assertEqual(decision.classification.project_id, "general-temporary")
        self.assertTrue(decision.queued)
```

Also test cross-midnight turns, multiple projects in one session, allowed content types, invalid categories, unknown project IDs, confidence boundaries `0.84`/`0.85`, prompt input containing safe text only, three-related-segment promotion across two sessions or two dates, failure to promote insufficient evidence, and semantic alias merge at `0.95` only.

- [ ] **Step 2: Run the classification tests and confirm failure**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_classify.py
```

Expected: import errors for the Hermes and classifier modules.

- [ ] **Step 3: Extract one tool-less Hermes client**

Move authentication, `/v1/toolsets` preflight, HTTP request, timeout, and response extraction from `ai_session_sync_digest.py` into `ai_session_sync_hermes.py`. Every call must first prove no enabled toolset, then send `"tools": []` and `"tool_choice": "none"`.

Add `request_hermes_json` that strips a single Markdown code fence if present, parses one JSON object, and rejects additional prose. The module must never accept `RawMessage`.

- [ ] **Step 4: Implement stable turn segmentation**

Start a segment at each user message and include following assistant messages until the next user message. An orphan assistant message becomes its own segment with content type `conversation`. Compute `segment_id` from ordered safe message IDs and classification policy version. A turn crossing Shanghai midnight belongs to the initiating user message’s day while every original message remains in its own date archive.

- [ ] **Step 5: Validate Hermes classification output**

Require exactly these fields: `primary_category`, `project_id`, `suggested_project_name`, `topic_tags`, `content_type`, `sensitivity`, `confidence`, `scope`, and `alias_of`. Validate category and content-type enumerations, confidence range, safe slug syntax, and maximum tag count. Reject an existing-project assignment below `0.85`; route it to `general-temporary` and queue its evidence.

- [ ] **Step 6: Implement dynamic non-code projects and aliases**

During nightly reconciliation, create a semantic project only after at least three related queued segments span at least two session IDs or two Shanghai dates. Use a stable slug derived from normalized suggested name plus a collision suffix. Merge semantic aliases only at confidence `0.95` or higher. Update future routing and indices; never rewrite historical originals.

- [ ] **Step 7: Run classifier, digest, and original regressions**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_classify.py test_ai_session_sync_projects.py test_ai_session_sync.py
```

Expected: all tests pass; the legacy digest tests prove the extracted Hermes client retains zero-tool preflight and repair behavior.

- [ ] **Step 8: Commit**

```bash
git add ai_session_sync_hermes.py ai_session_sync_classify.py ai_session_sync_digest.py ai_session_sync_projects.py ai_session_sync_models.py test_ai_session_sync_classify.py test_ai_session_sync.py
git commit -m "分类: 按对话段自动归项并抑制项目碎片"
```

---

### Task 6: Add the hash-chained eight-stage receipt ledger

**Files:**
- Create: `ai_session_sync_ledger.py`
- Modify: `ai_session_sync_models.py`
- Modify: `ai_session_sync_core.py`
- Create: `test_ai_session_sync_ledger.py`

**Interfaces:**
- `make_cycle_id(project_id: str, segment_ids: Sequence[str], policy_hash: str) -> str`
- `append_receipt(path: Path, receipt: Receipt) -> bool`
- `record_stage(ledger, cycle_id, project_id, stage, input_hash, artifact_hash, parent_handoff_hash, metadata) -> Receipt`
- `derive_cycle_status(receipts: Sequence[Receipt], failures: Sequence[FailureRecord]) -> CycleStatus`
- `render_status_page(statuses: Sequence[CycleStatus]) -> str`

- [ ] **Step 1: Write failing ledger tests**

Test ordering, hashes, idempotence, status labels, and feedback lineage:

```python
class LedgerTests(unittest.TestCase):
    def test_cycle_is_closed_only_after_all_eight_stages(self):
        receipts = [self.receipt(stage) for stage in STAGES[:-1]]
        self.assertNotEqual(derive_cycle_status(receipts, ()).label, "已闭环")
        receipts.append(self.receipt("feedback_archived"))
        self.assertEqual(derive_cycle_status(receipts, ()).label, "已闭环")

    def test_feedback_requires_the_consumed_parent_handoff_hash(self):
        with self.assertRaises(StageOrderError):
            record_stage(self.ledger, "c1", "screener", "feedback_archived", "a", "b", "wrong", {})

    def test_duplicate_receipt_is_a_byte_noop(self):
        self.assertTrue(append_receipt(self.path, self.published))
        before = self.path.read_bytes()
        self.assertFalse(append_receipt(self.path, self.published))
        self.assertEqual(self.path.read_bytes(), before)
```

Also test out-of-order stage rejection, failed-stage counters, three-failure interruption, security isolation precedence, waiting-for-handoff, local-complete/cloud-pending, digest-pending, short hashes, malformed final JSONL recovery, and status pages containing no text, path, amount, or identity.

- [ ] **Step 2: Run and confirm failure**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_ledger.py
```

Expected: import error because the ledger module is absent.

- [ ] **Step 3: Implement deterministic cycles and ordered stages**

Define:

```python
STAGES = (
    "collected",
    "sanitized",
    "classified",
    "archived",
    "digested",
    "published",
    "consumed",
    "feedback_archived",
)
```

Compute `cycle_id` from project ID, sorted segment IDs, and the combined policy fingerprint. Each receipt’s `input_hash` must match the previous stage’s `artifact_hash`; `feedback_archived` must also match the handoff hash recorded by `consumed`.

Use a stable receipt ID derived from cycle, stage, artifact hash, and parent handoff hash. Reject conflicting duplicate stage receipts; silently skip byte-identical duplicates.

- [ ] **Step 4: Make JSONL writes durable and private**

Open `receipts.jsonl` with `0600`, acquire `flock`, reload existing IDs, append one canonical JSON line, flush, and `fsync`. On load, ignore only an incomplete final line; any malformed middle line returns a ledger-corrupt error and prevents new receipts.

Metadata values are restricted to platform, session ID, project ID, policy version, artifact relative path, Git commit, counts, and hash strings. Reject metadata keys containing `text`, `content`, `prompt`, `amount`, `identity`, `secret`, or `source_path`.

- [ ] **Step 5: Derive status without optimistic shortcuts**

Map the latest valid cycle to exactly these labels: `已闭环`, `等待接班`, `本地完成，云端待推送`, `摘要待补跑`, `安全隔离`, or `闭环中断`. A missing early stage remains `闭环中断`; a quarantined message remains `安全隔离`; three consecutive failures at one stage override a less severe waiting label.

Render `20-AI复核/全局/闭环状态.md` from project, label, last stage, Shanghai time, and eight-character hashes only. Treat the status page as a derived artifact outside the cycle hash to avoid publication recursion.

- [ ] **Step 6: Run focused tests**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_ledger.py
```

Expected: all ledger tests pass.

- [ ] **Step 7: Commit**

```bash
git add ai_session_sync_ledger.py ai_session_sync_models.py ai_session_sync_core.py test_ai_session_sync_ledger.py
git commit -m "闭环: 建立八阶段哈希回执与状态判定"
```

---

### Task 7: Write one sanitized original and project-linked indices

**Files:**
- Create: `ai_session_sync_archive.py`
- Modify: `ai_session_sync_core.py`
- Modify: `ai_session_sync.py`
- Modify: `ai_session_sync_models.py`
- Create: `test_ai_session_sync_archive.py`
- Modify: `test_ai_session_sync.py`

**Interfaces:**
- `message_anchor(message_id: str) -> str`
- `render_archive(existing: str, segments: Sequence[ClassifiedSegment]) -> str`
- `render_classification_index(day: str, segments: Sequence[ClassifiedSegment], vault: Path) -> str`
- `archive_segments(...) -> ArchiveReport`

- [ ] **Step 1: Write failing one-copy and anchor tests**

```python
class ClassifiedArchiveTests(unittest.TestCase):
    def test_multi_project_session_writes_one_original_and_two_index_links(self):
        report = archive_segments(self.two_project_segments, self.vault, self.ledger)
        originals = tuple((self.vault / "90-系统/AI会话/原文").rglob("*.md"))
        self.assertEqual(len(originals), 1)
        index = (self.vault / "90-系统/AI会话/分类索引/2026-07-21.md").read_text()
        self.assertIn("project_id: screener", index)
        self.assertIn("project_id: hermes", index)
        self.assertEqual(index.count("[[90-系统/AI会话/原文/"), 2)

    def test_second_archive_is_byte_and_receipt_idempotent(self):
        archive_segments(self.segments, self.vault, self.ledger)
        first = self.snapshot()
        archive_segments(self.segments, self.vault, self.ledger)
        self.assertEqual(self.snapshot(), first)
```

Also test project union in frontmatter, stable `^msg-<12 hex>` anchors, cross-midnight placement, segment-to-message links, classification fields, policy versions, old-marker migration, atomic-write failure, quarantined-message absence, and receipts recorded only after durable writes.

- [ ] **Step 2: Run and confirm failure**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_archive.py
```

Expected: import error for the archive module.

- [ ] **Step 3: Extract archive rendering from core**

Move archive path, marker parsing, merge, and Markdown rendering from `ai_session_sync_core.py` to `ai_session_sync_archive.py`. Keep compatibility imports in core for the original regression suite.

Render one file per platform/session/Shanghai day under `90-系统/AI会话/原文`. Frontmatter contains `redaction_policy`, `classification_policy`, sorted `projects`, platform, session ID, date, and `generated_by`.

Every message block ends with an Obsidian block anchor derived only from the message ID. Never use the source path or user text in filenames or anchors.

- [ ] **Step 4: Render link-only classification indices**

Write `90-系统/AI会话/分类索引/YYYY-MM-DD.md`. Each segment entry contains its classification fields and wikilinks to the relevant original message anchors. Do not copy message text into the index. Sort by timestamp, session ID, segment ID, and project ID for byte stability.

- [ ] **Step 5: Wire receipt order into collection**

For each accepted raw message/segment, record `collected`, then `sanitized`, then `classified`. Write archives and the day index atomically; only then record `archived` with a hash over all affected managed files. Quarantined messages record a failure and never receive `sanitized`.

Advance source state after a safe message is archived or a blocked message has a durable quarantine record. Store policy versions so a later policy upgrade retries previously quarantined fingerprints.

- [ ] **Step 6: Run archive and prior suites**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_archive.py test_ai_session_sync_ledger.py test_ai_session_sync_classify.py test_ai_session_sync_privacy.py test_ai_session_sync.py
```

Expected: all tests pass, including original active-append, state-loss, truncation, cross-midnight, and dry-run regressions.

- [ ] **Step 7: Commit**

```bash
git add ai_session_sync_archive.py ai_session_sync_core.py ai_session_sync.py ai_session_sync_models.py test_ai_session_sync_archive.py test_ai_session_sync.py
git commit -m "归档: 原文单份保存并生成项目锚点索引"
```

---

### Task 8: Generate project daily digests, rolling handoffs, and a safe global overview

**Files:**
- Modify: `ai_session_sync_digest.py`
- Create: `ai_session_sync_handoff.py`
- Modify: `ai_session_sync_hermes.py`
- Modify: `ai_session_sync_audit.py`
- Modify: `ai_session_sync_ledger.py`
- Create: `test_ai_session_sync_handoff.py`
- Modify: `test_ai_session_sync.py`

**Interfaces:**
- `collect_project_sources(vault, project_id, day) -> Tuple[ProjectSource, ...]`
- `digest_project_day(...) -> ProjectDigestReport`
- `update_current_handoff(...) -> HandoffReport`
- `update_global_overview(...) -> OverviewReport`
- `validate_project_digest(text, allowed_anchors) -> None`
- `validate_current_handoff(text, previous, allowed_anchors) -> None`

- [ ] **Step 1: Write failing project-output tests**

Test isolation and preservation:

```python
class HandoffTests(unittest.TestCase):
    def test_project_digest_reads_only_its_indexed_message_anchors(self):
        sources = collect_project_sources(self.vault, "screener", "2026-07-21")
        self.assertEqual({item.project_id for item in sources}, {"screener"})
        self.assertNotIn("Hermes private detail", "".join(item.content for item in sources))

    def test_current_handoff_cannot_drop_an_unrevoked_constraint(self):
        with self.assertRaises(ValueError):
            validate_current_handoff(self.output_without_constraint, self.previous, self.allowed)

    def test_invalid_or_sensitive_output_preserves_previous_artifacts(self):
        before = self.current.read_bytes()
        report = update_current_handoff(self.config, client=self.sensitive_client)
        self.assertFalse(report.updated)
        self.assertEqual(self.current.read_bytes(), before)
```

Also test the eight daily headings, current-handoff headings, every factual bullet requiring an allowed message anchor, unknown link rejection, cross-project link-without-copy, empty sections, old constraint carry-forward, sourced revocation, completed todo marking, global overview field whitelist, digest hashes, no-change skips, one repair call, chunking, audit failure, and `digested` receipt timing.

- [ ] **Step 2: Run and confirm failure**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_handoff.py
```

Expected: import error for the handoff module and project-aware digest assertions failing.

- [ ] **Step 3: Change source collection from global day files to project anchors**

Read the managed classification index, resolve only entries for the requested project, and extract only the linked sanitized message blocks from managed originals. Reject missing, duplicate, unknown, or cross-project anchors. Hermes receives the safe extracted block plus its allowed link, never an entire unrelated session file.

- [ ] **Step 4: Generate one project daily digest**

Write `20-AI复核/项目/<project-id>/日报/YYYY-MM-DD.md` with the approved eight headings. Every non-`暂无` bullet must contain an allowed message-anchor wikilink. Reuse chunking, normalization, one repair attempt, independent privacy audit, atomic write, and source-hash skip logic.

- [ ] **Step 5: Generate a rolling current handoff**

Write `20-AI复核/项目/<project-id>/当前接力.md` with these exact headings:

```text
## 项目目标和当前阶段
## 已生效拍板与硬约束
## 最新完成项和证据
## 未完成待办及优先级
## 风险、失败尝试和禁止重复路径
## 最近有效来源
```

Provide Hermes with the previous handoff and latest valid project digest. Validate that each old constraint either remains byte-normalized equivalent or has an allowed source explicitly marking it revoked/replaced. Require completed todos to remain as sourced completion records for at least one daily cycle. Preserve the previous file on any failure.

- [ ] **Step 6: Generate a constrained global overview**

Write `20-AI复核/全局/当前总览.md`. Allow only project display name, project ID, status label, last activity time, sourced cross-project todos, and sourced global working preferences. Reject detailed project prose, exact finance, privacy finding names, raw paths, and links to project originals. Build status/activity deterministically; use Hermes only for already-classified `scope == "global"` constraints and cross-project todos.

- [ ] **Step 7: Record a single digested artifact set**

After the project digest, current handoff, and global overview all validate and write successfully, compute their hashes and record `digested`. A global-overview failure leaves the project artifacts valid but the cycle at `archived`; the next run retries only the missing stage.

- [ ] **Step 8: Run handoff and legacy digest tests**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_handoff.py test_ai_session_sync.py
```

Expected: all tests pass; old mixed-global behavior is replaced by assertions for per-project output.

- [ ] **Step 9: Commit**

```bash
git add ai_session_sync_digest.py ai_session_sync_handoff.py ai_session_sync_hermes.py ai_session_sync_audit.py ai_session_sync_ledger.py test_ai_session_sync_handoff.py test_ai_session_sync.py
git commit -m "接力: 生成项目日报当前状态与安全总览"
```

---

### Task 9: Orchestrate change-driven collection, digesting, retry, and alerts

**Files:**
- Modify: `ai_session_sync.py`
- Modify: `ai_session_sync_core.py`
- Modify: `ai_session_sync_handoff.py`
- Modify: `ai_session_sync_ledger.py`
- Modify: `test_ai_session_sync.py`
- Create: `test_ai_session_sync_e2e.py`

**Interfaces:**
- CLI commands: `run`, `collect`, `digest`, `reconcile`, `catch-up`, `status`, `observe-publish`, and `backfill`.
- `run_cycle(config, now, dry_run) -> PipelineReport`
- `projects_ready_for_digest(state, now) -> Tuple[str, ...]`
- `notify_failure(config, failure) -> None`

- [ ] **Step 1: Write failing orchestration tests**

Add fake-clock and fake-client tests for:

- collection every invocation;
- three minutes of project quiet before digest;
- no more than one Hermes digest per project in fifteen minutes;
- no Hermes call without material changes;
- four fixed fallback slots catching missed work;
- startup catch-up after simulated sleep;
- independent retry counters;
- nonzero return for any required failed stage;
- previous artifacts and cursors preserved after failure;
- dry run leaving the complete temp tree byte-identical;
- security isolation causing an immediate local alert;
- three consecutive failures causing a local notification;
- Server酱 payload containing project ID, stage, and error category only.

- [ ] **Step 2: Run the new orchestration tests and confirm failure**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_e2e.py
```

Expected: failures because `run`, readiness state, catch-up, observer, and alert interfaces are absent.

- [ ] **Step 3: Expand configuration and state without unsafe defaults**

Add state paths for registry fallback, pending classification, receipts, read receipts, bindings, quarantine, failure counters, and migration backups. Add per-project `dirty_since`, `last_message_at`, `last_hermes_at`, `last_digest_hash`, and `last_success_at`.

State schema upgrades must copy old source and digest cursors, write versioned new state atomically, and retain a timestamped local backup. A corrupt state returns nonzero instead of silently resetting to empty.

- [ ] **Step 4: Implement `run` and catch-up**

`run` acquires one process lock, collects all sources, reconciles due low-confidence records, digests ready projects, renders status, and wakes the existing Vault publisher when managed Vault bytes changed. It returns `1` if any required stage failed, even if later independent projects succeeded.

`catch-up --project <id>` runs collection and the selected project digest immediately when newer source material exists, respecting a 300-second overall deadline but bypassing the ordinary three-minute idle wait. It still respects privacy, validation, and receipt ordering.

- [ ] **Step 5: Implement failure-safe notifications**

Use `/usr/bin/osascript` with a fixed argument list for local notifications; do not interpolate untrusted text into AppleScript source. Send an immediate local notification for isolation and one after a stage’s third consecutive failure. If an existing Server酱 sender is configured, pass a pre-sanitized fixed-field payload. Notification failure increments its own counter but never changes a successful business-stage receipt.

- [ ] **Step 6: Make structured logs content-free**

Log timestamp, mode, status, project ID, stage, counts, duration, policy version, and exception class only. Add capture tests proving logs exclude raw text, safe text, source paths, exact amounts, identities, HMAC aliases, and credential categories.

- [ ] **Step 7: Run focused end-to-end regressions**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_e2e.py test_ai_session_sync_handoff.py test_ai_session_sync_archive.py test_ai_session_sync_ledger.py test_ai_session_sync.py
```

Expected: all tests pass and every injected stage failure returns `1` without a later-stage receipt.

- [ ] **Step 8: Commit**

```bash
git add ai_session_sync.py ai_session_sync_core.py ai_session_sync_handoff.py ai_session_sync_ledger.py test_ai_session_sync.py test_ai_session_sync_e2e.py
git commit -m "编排: 变更驱动补齐重试并显式报告失败"
```

---

### Task 10: Implement authoritative handoff binding, reading, and feedback lineage

**Files:**
- Create: `ai_session_handoff.py`
- Modify: `ai_session_sync_projects.py`
- Modify: `ai_session_sync_ledger.py`
- Modify: `ai_session_sync.py`
- Create: `test_ai_session_sync_handoff.py`
- Modify: `test_ai_session_sync_e2e.py`

**Interfaces:**
- `ai-session-handoff bind --platform claude --stdin-json`
- `ai-session-handoff read --platform <codex|claude> --cwd <path> [--session-id <id>]`
- `bind_claude_hook(payload, bindings_path) -> BindingResult`
- `read_handoff(config, platform, cwd, session_id, now) -> ReadResult`
- `detect_feedback_parent(message, read_receipts) -> str`

- [ ] **Step 1: Write failing identity and read-receipt tests**

```python
class HandoffClientTests(unittest.TestCase):
    def test_claude_binding_uses_hook_session_id_and_writes_no_context(self):
        output = bind_claude_hook(self.session_start_payload, self.bindings)
        self.assertEqual(output.stdout, "")
        self.assertEqual(output.binding.session_id, "claude-authoritative-id")

    def test_missing_codex_thread_id_may_show_paths_but_cannot_consume(self):
        result = read_handoff(self.config, "codex", self.cwd, "", self.now)
        self.assertFalse(result.consumed)
        self.assertIn("身份未确认", result.notice)

    def test_later_feedback_carries_consumed_handoff_hash(self):
        parent = detect_feedback_parent(self.feedback_message, self.read_receipts)
        self.assertEqual(parent, self.handoff_hash)
```

Also test malformed hook JSON, wrong hook event, transcript outside Claude root, ambiguous active bindings, binding expiry, deterministic project resolution, general-temporary fallback, project isolation, stale catch-up success, stale catch-up failure notice, pointer-manifest hashes, duplicate reads, resumed sessions, and feedback messages before versus after read time.

- [ ] **Step 2: Run and confirm failure**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_handoff.py
```

Expected: import error because the client does not exist.

- [ ] **Step 3: Bind Claude identity from SessionStart stdin**

Read one JSON object from stdin. Require `hook_event_name == "SessionStart"`, non-empty `session_id`, a transcript path below the configured Claude projects root, and an absolute cwd. Append a `0600` binding record containing hashes and paths but no transcript text. Print nothing on success so SessionStart injects no handoff body.

Select a Claude binding for `read` only when exactly one non-expired binding matches platform and cwd. Multiple candidates show an ambiguity notice and prohibit `consumed`.

- [ ] **Step 4: Resolve the current project and freshness**

Use Git root/remote and the registry in the same deterministic order as classification. Never infer project from handoff text. If the newest accepted source timestamp exceeds the project digest timestamp, run `ai-session-sync catch-up --project <id>` with a 300-second timeout. On failure, expose the previous valid paths plus `最新内容尚未归纳`; do not mark the stale cycle consumed.

- [ ] **Step 5: Produce a pointer manifest, not a copied handoff body**

Atomically write a `0600` local pointer manifest under `handoff-views/<platform>-<session-id>.json` containing absolute paths and SHA-256 hashes for current handoff, latest daily digest, and global overview. Print the manifest path and freshness notice. Do not copy Markdown content into state.

- [ ] **Step 6: Record consumption and close feedback**

After all three allowed files are readable and hash-verified, append `read-receipts.jsonl` with platform, authoritative session ID, project ID, handoff hash, digest hash, read time, and freshness. Record `consumed` against the published cycle.

When collection later accepts a message from that session after the read time, attach the read handoff hash as `parent_handoff_hash`. Once the feedback message is sanitized, classified back to the project, and archived, record `feedback_archived` for the parent cycle.

- [ ] **Step 7: Run handoff and ledger tests**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_handoff.py test_ai_session_sync_ledger.py test_ai_session_sync_e2e.py
```

Expected: all tests pass; no identity ambiguity produces a false consumption receipt.

- [ ] **Step 8: Commit**

```bash
git add ai_session_handoff.py ai_session_sync_projects.py ai_session_sync_ledger.py ai_session_sync.py test_ai_session_sync_handoff.py test_ai_session_sync_e2e.py
git commit -m "接班: 绑定权威会话并归档反馈父链"
```

---

### Task 11: Install launchd jobs, global instructions, and the Claude identity hook safely

**Files:**
- Modify: `install_ai_session_sync.sh`
- Modify: `ai_session_handoff.py`
- Modify: `test_ai_session_sync.py`
- Modify: `test_ai_session_sync_handoff.py`

**Interfaces:**
- Installed commands: `~/.local/bin/ai-session-sync` and `~/.local/bin/ai-session-handoff`.
- launchd labels: `com.harryd317.ai-session-sync` and `com.harryd317.ai-session-fallback`.
- Managed instruction markers: `<!-- ai-session-handoff:begin -->` and `<!-- ai-session-handoff:end -->`.
- Managed Claude hook command contains the stable token `ai-session-handoff-managed-v1` for merge/idempotence detection.

- [ ] **Step 1: Write failing isolated-installer tests**

Extend installer tests to assert:

- all runtime modules and both wrappers are installed;
- the collector interval is 300 seconds;
- fallback slots are exactly 05:50, 11:50, 17:50, and 23:50;
- the active Codex global file is `AGENTS.override.md` when that file is non-empty, otherwise `AGENTS.md`;
- one managed instruction block is inserted and updated byte-idempotently;
- existing text before and after the block remains unchanged;
- existing Claude hooks remain intact;
- one `SessionStart` hook with `startup|resume|clear|compact`, absolute command, and ten-second timeout is merged;
- the hook command binds only and does not call `read`;
- invalid existing JSON aborts without rewriting settings;
- installed key/state/quarantine permissions are correct;
- dry run writes nothing;
- the script never assigns to `HOME`, `CODEX_HOME`, or a lowercase equivalent;
- install does not touch screener service labels or trading jobs.

- [ ] **Step 2: Run installer tests and confirm failure**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync.py test_ai_session_sync_handoff.py
```

Expected: installer assertions fail because the current installer has 900-second collection, no handoff wrapper, no global blocks, and no Claude hook.

- [ ] **Step 3: Install versioned runtime modules and wrappers**

Use `USER_ROOT`, `SCRIPT_ROOT`, `RUNTIME_LIB`, and other task-specific names; never assign to `HOME`. Copy the exact module list in the planned structure. Generate wrappers that invoke `/usr/bin/python3` and the installed absolute module path. Write wrappers atomically, set `0755`, and verify installed source SHA-256 against the worktree before loading jobs.

- [ ] **Step 4: Install the two schedules**

The five-minute job runs `ai-session-sync run` with `RunAtLoad`. The fallback job runs `ai-session-sync catch-up --all` at the four approved Shanghai-local wall-clock times. Both write to the content-free sync log. Use separate labels so one failure is observable without suppressing the other.

- [ ] **Step 5: Merge the active Codex global instruction block**

Codex reads the first non-empty global file in this order: `$CODEX_HOME/AGENTS.override.md`, then `$CODEX_HOME/AGENTS.md`; this discovery behavior must be reflected in installation and tests. Insert this managed content into the active file:

```markdown
<!-- ai-session-handoff:begin -->
## AI project handoff

Before beginning project work in a main task, run:

`ai-session-handoff read --platform codex --session-id "$CODEX_THREAD_ID" --cwd "$PWD"`

Read only the files named by the returned pointer manifest. If the command reports stale or failed handoff data, tell the user before relying on it. Do not create a consumed receipt without an authoritative task ID.
<!-- ai-session-handoff:end -->
```

Preserve all non-managed bytes except for one necessary newline at the insertion boundary. The implementation should link its operator documentation to [OpenAI’s AGENTS.md discovery guide](https://learn.chatgpt.com/docs/agent-configuration/agents-md).

- [ ] **Step 6: Merge Claude global instructions and SessionStart hook**

Insert an equivalent managed block into `~/.claude/CLAUDE.md`, instructing Claude to run `ai-session-handoff read --platform claude --cwd "$PWD"` and read the returned manifest paths.

Parse `~/.claude/settings.json` as JSON and append this hook only when no command contains the stable managed token:

```json
{
  "matcher": "startup|resume|clear|compact",
  "hooks": [
    {
      "type": "command",
      "command": "/absolute/path/ai-session-handoff bind --platform claude --stdin-json --managed-token ai-session-handoff-managed-v1",
      "timeout": 10
    }
  ]
}
```

The hook receives JSON on stdin and emits an empty stdout on success. Follow the Claude hook contract that `SessionStart` supplies authoritative session data and injects stdout into context; therefore no Markdown or pointer path may be printed by `bind`.

- [ ] **Step 7: Run installer idempotence and syntax tests**

```bash
bash -n install_ai_session_sync.sh
/usr/bin/python3 -m unittest -v test_ai_session_sync.py test_ai_session_sync_handoff.py
```

Expected: shell syntax succeeds; two isolated installs produce byte-identical managed files and exactly one managed hook.

- [ ] **Step 8: Commit**

```bash
git add install_ai_session_sync.sh ai_session_handoff.py test_ai_session_sync.py test_ai_session_sync_handoff.py
git commit -m "安装: 接入全局Codex与Claude显式接班"
```

---

### Task 12: Make the existing Vault publisher own AI publication and report real failures

**Files:**
- Modify: `obsidian_sync.py`
- Modify: `obsidian_sync_core.py`
- Modify: `install_obsidian_sync.sh`
- Modify: `ai_session_sync.py`
- Modify: `ai_session_sync_ledger.py`
- Create: `test_obsidian_sync.py`
- Modify: `test_ai_session_sync_ledger.py`

**Interfaces:**
- `publish_vault(...) -> PublishResult`
- `publish_pending_ai_changes(config, dry_run, logger) -> Optional[PublishResult]`
- `ai-session-sync observe-publish --commit <sha> --vault <path>`
- `observe_publish(...) -> PublishObservation`

- [ ] **Step 1: Write failing publisher tests**

Use temporary Git repositories and stub subprocesses to prove:

```python
class VaultPublisherTests(unittest.TestCase):
    def test_all_mode_returns_nonzero_when_mirror_or_digest_stage_fails(self):
        self.assertEqual(run(self.config, "all", self.head, False), 1)

    def test_dirty_ai_paths_publish_even_when_screener_cursor_is_at_head(self):
        result = publish_pending_ai_changes(self.config, False, self.logger)
        self.assertIn("20-AI复核/项目/screener/当前接力.md", result.changed_paths)

    def test_observer_failure_is_not_reported_as_publish_success(self):
        self.observer.return_value.returncode = 1
        self.assertEqual(run(self.config, "all", self.head, False), 1)
```

Also test no empty commits, pull/push failures, local commit retention after push failure, origin alignment, allowed managed path prefixes, status-only commits not opening new cycles, publish receipt after push only, and exact Git commit/path hashes.

- [ ] **Step 2: Run and confirm the known false-success behavior**

```bash
/usr/bin/python3 -m unittest -v test_obsidian_sync.py test_ai_session_sync_ledger.py
```

Expected: failures showing `obsidian_sync.run(..., mode="all")` currently returns success after staged failures and does not publish standalone AI Vault changes.

- [ ] **Step 3: Return structured publication evidence**

Make `publish_vault` return the pushed commit SHA and changed Vault-relative paths. It must pull/rebase, stage permitted changes, skip an empty index, commit, push, fetch, and verify `origin/main...main == 0 0`. A push failure retains the local commit and raises a typed error.

- [ ] **Step 4: Publish dirty AI-managed paths through the existing job**

At the end of every publisher run, inspect only:

```text
90-系统/AI会话/
20-AI复核/项目/
20-AI复核/全局/
```

If any are dirty, publish them with the same `publish_vault` function. This is an extension of the existing publisher, not a new Git-writing process. The derived `闭环状态.md` may be committed, but a status-only commit must not create a fresh content cycle.

- [ ] **Step 5: Observe the pushed commit before reporting success**

After push and alignment, invoke the installed AI CLI with an argument list:

```text
ai-session-sync observe-publish --commit <full-sha> --vault <absolute-vault>
```

The observer verifies the commit is an ancestor of `origin/main`, resolves each pending cycle artifact to its Git blob hash, and records `published`. It updates local ledger state only during the publisher process; the next ordinary sync renders the derived status page, preventing amend/push recursion.

- [ ] **Step 6: Propagate every staged failure**

Accumulate mirror, old screener digest, AI publication, push alignment, and observer errors. `all` returns `1` if any required stage fails, even when another stage succeeds. Remove branches that log an error and then fall through to `Sync complete` with return code `0`.

- [ ] **Step 7: Stop assigning to `HOME` in the publisher installer**

Rename the installer’s path root variable to `USER_ROOT`, preserve existing behavior, and add a regression assertion. Do not alter the screener web launchd job.

- [ ] **Step 8: Run publisher and ledger tests**

```bash
bash -n install_obsidian_sync.sh
/usr/bin/python3 -m unittest -v test_obsidian_sync.py test_ai_session_sync_ledger.py
```

Expected: all tests pass; injected mirror, digest, push, alignment, and observer failures return nonzero.

- [ ] **Step 9: Commit**

```bash
git add obsidian_sync.py obsidian_sync_core.py install_obsidian_sync.sh ai_session_sync.py ai_session_sync_ledger.py test_obsidian_sync.py test_ai_session_sync_ledger.py
git commit -m "发布: 由现有Vault任务确认AI产物上云"
```

---

### Task 13: Build recoverable migration, historical scanning, and rollback

**Files:**
- Create: `ai_session_sync_migrate.py`
- Modify: `ai_session_sync.py`
- Modify: `ai_session_sync_audit.py`
- Modify: `install_ai_session_sync.sh`
- Create: `test_ai_session_sync_migrate.py`

**Interfaces:**
- `ai-session-sync migrate plan --since <day>`
- `ai-session-sync migrate stage --since <day> --staging-vault <path>`
- `ai-session-sync migrate activate --manifest <path> --confirm <token>`
- `ai-session-sync migrate rollback --manifest <path> --confirm <token>`
- `scan_vault_and_history(vault, audit) -> SafetyScanReport`

- [ ] **Step 1: Write failing migration tests**

Cover backup, scan, stage, activate, and rollback:

```python
class MigrationTests(unittest.TestCase):
    def test_real_secret_in_history_stops_before_activation(self):
        report = scan_vault_and_history(self.vault_with_secret_history, audit_safe_text)
        self.assertFalse(report.safe_to_activate)
        self.assertEqual(report.findings[0].keys(), {"commit", "path", "category"})

    def test_activate_moves_old_managed_tree_to_recoverable_backup(self):
        result = activate(self.manifest, self.confirm_token)
        self.assertTrue(result.backup_root.exists())
        self.assertTrue((result.backup_root / "vault-managed").exists())

    def test_rollback_restores_old_state_jobs_and_vault_bytes(self):
        before = self.pre_activation_snapshot
        rollback(self.manifest, self.confirm_token)
        self.assertEqual(self.snapshot(), before)
```

Also test Codex subagent artifact identification, polluted digest identification, no permanent deletion, state/launchd/global-file backups, staging-vault idempotence, policy-version rebuild, failed activation restoration, wrong confirmation token, scan output zero leakage, and dry-run zero writes.

- [ ] **Step 2: Run and confirm failure**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_migrate.py
```

Expected: import error because the migration module does not exist.

- [ ] **Step 3: Implement a read-only migration plan**

Inventory current AI state, managed Vault directories, AI launchd plists, active global instruction files, Claude settings, source-session counts, known subagent artifacts, and polluted digest candidates. The plan prints counts and destination paths only; it never prints message text, identities, amounts, aliases, credentials, or source paths.

- [ ] **Step 4: Scan current Vault and every reachable Git revision**

Use `git rev-list --all` and `git grep`/blob reads through argument-list subprocesses. Feed content directly to the independent audit in memory. Findings contain only commit SHA, Vault-relative path, and category. Any likely real credential sets `safe_to_activate=False`; activation stops and the operator must rotate the credential before rerunning. Placeholder/example classification must be conservative and covered by fixtures.

- [ ] **Step 5: Stage a full rebuild in an isolated Vault**

Copy no existing managed AI Markdown into staging. Re-run source discovery, privacy, classification, archive, digest, handoff, and status from original platform JSONL. Use a fake publisher receipt for staging only, clearly marked `environment=staging`, and exclude it from production status.

Run the rebuild twice and compare a sorted path/SHA-256 manifest. The second run must create no changed bytes, duplicate projects, duplicate anchors, duplicate receipts, or Hermes calls for unchanged projects.

- [ ] **Step 6: Activate with a recoverable transaction**

Before mutation, create `~/.local/state/ai-session-sync/migration-backups/<timestamp>/` with `0700`. Copy state, relevant launchd files, active global instruction files, Claude settings, and existing managed Vault directories into it. Record modes and SHA-256 in a `0600` manifest.

Pause only old AI session jobs. Move existing managed AI directories into the backup, move staged directories into the Vault, install new runtime/jobs/instructions, and run a local collect/status smoke test. On any failure, restore the backup automatically and return `1`.

- [ ] **Step 7: Implement explicit rollback**

Rollback unloads only new AI jobs, restores prior managed Vault directories, state, plists, global files, and Claude settings from the manifest, validates hashes, and reloads the previous AI jobs. It does not alter raw platform sessions, screener services, databases, or trading jobs.

- [ ] **Step 8: Run migration and safety tests**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync_migrate.py test_ai_session_sync_privacy.py test_ai_session_sync_archive.py
```

Expected: all tests pass; forced failures at every activation step restore the byte snapshot.

- [ ] **Step 9: Commit**

```bash
git add ai_session_sync_migrate.py ai_session_sync.py ai_session_sync_audit.py install_ai_session_sync.sh test_ai_session_sync_migrate.py
git commit -m "迁移: 可恢复重建会话资料并扫描历史泄露"
```

---

### Task 14: Prove the complete loop in isolated integration tests

**Files:**
- Modify: `test_ai_session_sync_e2e.py`
- Modify: `test_ai_session_sync_handoff.py`
- Modify: `test_ai_session_sync_migrate.py`
- Modify: `test_obsidian_sync.py`
- Modify: `test_ai_session_sync.py`

**Interfaces:**
- One temporary fixture spans Codex sources, Claude sources, state, staging Vault, bare Vault remote, Hermes classifier/digest stub, global files, launchd output, and both handoff clients.

- [ ] **Step 1: Add the full Codex → Claude → Codex fixture**

The first Codex main session must contain one explicit decision and one todo for `screener`, plus another turn for an unrelated project. The fixture must prove:

1. only the safe `screener` turn enters its project sources;
2. all stages through `published` are receipted after a real temporary Git push;
3. a Claude SessionStart binding and `read` record `consumed`;
4. later Claude feedback archives with the first handoff hash and records `feedback_archived`;
5. the Claude feedback creates a second project cycle;
6. a new Codex task ID reads the second handoff and records `consumed`;
7. later Codex feedback closes the second cycle;
8. both cycles display `已闭环`.

- [ ] **Step 2: Add failure-matrix integration cases**

Inject failures at independent audit, registry load, Hermes classification, Hermes digest, archive write, handoff validation, Git pull, Git push, publish observation, stale catch-up, identity binding, and feedback archiving. For each case assert the exact last valid stage, nonzero exit code, preserved prior artifact, and absence of the next receipt.

- [ ] **Step 3: Run the focused full suite**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync.py test_ai_session_sync_sources.py test_ai_session_sync_privacy.py test_ai_session_sync_projects.py test_ai_session_sync_classify.py test_ai_session_sync_archive.py test_ai_session_sync_handoff.py test_ai_session_sync_ledger.py test_ai_session_sync_migrate.py test_ai_session_sync_e2e.py test_obsidian_sync.py
```

Expected: every test passes with final `OK`.

- [ ] **Step 4: Run the mandated screener regressions**

```bash
.venv/bin/python test_rules.py
.venv/bin/python test_rules_mid.py
.venv/bin/python test_web.py
.venv/bin/python test_regime.py
```

Expected: all four commands exit `0` and print `OK`.

- [ ] **Step 5: Verify protected behavior and runtime syntax**

```bash
git diff main...HEAD -- screener.py rules_mid.py params.py storage.py service.py static
bash -n install_ai_session_sync.sh
bash -n install_obsidian_sync.sh
/usr/bin/python3 -m compileall -q ai_session_sync.py ai_session_sync_core.py ai_session_sync_models.py ai_session_sync_sources.py ai_session_sync_privacy.py ai_session_sync_audit.py ai_session_sync_projects.py ai_session_sync_classify.py ai_session_sync_hermes.py ai_session_sync_archive.py ai_session_sync_digest.py ai_session_sync_handoff.py ai_session_sync_ledger.py ai_session_handoff.py ai_session_sync_migrate.py obsidian_sync.py obsidian_sync_core.py
```

Expected: protected-path diff is empty; both shell scripts and all Python modules validate.

- [ ] **Step 6: Commit the integration matrix**

```bash
git add test_ai_session_sync.py test_ai_session_sync_sources.py test_ai_session_sync_privacy.py test_ai_session_sync_projects.py test_ai_session_sync_classify.py test_ai_session_sync_archive.py test_ai_session_sync_handoff.py test_ai_session_sync_ledger.py test_ai_session_sync_migrate.py test_ai_session_sync_e2e.py test_obsidian_sync.py
git commit -m "测试: 验证双向接力八阶段闭环"
```

---

### Task 15: Roll out through shadow, rebuild, activation, and real two-way acceptance

**Files:**
- Runtime outside repository after approval: `~/.local/lib/ai-session-sync/`, `~/.local/bin/`, `~/Library/LaunchAgents/`, active global Codex guidance, `~/.claude/CLAUDE.md`, `~/.claude/settings.json`, AI state, and managed Vault paths.
- Create during delivery: `docs/交付总结-全局AI接力闭环-20260721.md`.

**Interfaces:**
- Production commands and receipts implemented in Tasks 9–13.

- [ ] **Step 1: Run shadow classification with no Vault writes**

```bash
/usr/bin/python3 ai_session_sync.py backfill --since 2026-06-20 --dry-run
/usr/bin/python3 ai_session_sync.py reconcile --dry-run
```

Expected: counts for Codex/Claude main sessions, excluded subagents/platform records, privacy categories, deterministic/Hermes/temporary routes, proposed projects, and affected days. No lock, log, state, key, quarantine, Vault, or Git bytes change.

Review the report for project fragmentation and unsafe over-redaction. Any discrepancy returns to the responsible task and its tests before rollout.

- [ ] **Step 2: Run the production safety plan and historical scan**

```bash
/usr/bin/python3 ai_session_sync.py migrate plan --since 2026-06-20
```

Expected: `safe_to_activate=true`. If false, stop rollout, rotate each real credential outside this program, and rerun until the scan passes. Do not paste findings into the task transcript.

- [ ] **Step 3: Build and verify a temporary Vault twice**

```bash
AI_LOOP_STAGING="$(mktemp -d)"
AI_LOOP_MANIFEST="$AI_LOOP_STAGING/activation.json"
/usr/bin/python3 ai_session_sync.py migrate stage --since 2026-06-20 --staging-vault "$AI_LOOP_STAGING/vault" --manifest-out "$AI_LOOP_MANIFEST"
/usr/bin/python3 ai_session_sync.py migrate stage --since 2026-06-20 --staging-vault "$AI_LOOP_STAGING/vault" --manifest-out "$AI_LOOP_MANIFEST"
```

Expected: the second stage reports zero changed artifacts, zero duplicate receipts, and zero unnecessary Hermes calls. Retain the emitted activation manifest path.

- [ ] **Step 4: Production activation checkpoint**

Before changing launchd, global instructions, state, or managed Vault directories, show the user the backup destination, staged project count, quarantined-message count, historical-scan result, rollback command, and exact managed paths. Obtain explicit approval for this outside-workspace activation.

- [ ] **Step 5: Activate and validate installed hashes**

Read the one-time confirmation token from the `0600` staging manifest, then run the exact activation command:

```bash
AI_LOOP_CONFIRM="$(/usr/bin/python3 -c 'import json,sys; print(json.load(open(sys.argv[1], encoding="utf-8"))["confirmation_token"])' "$AI_LOOP_MANIFEST")"
/usr/bin/python3 ai_session_sync.py migrate activate --manifest "$AI_LOOP_MANIFEST" --confirm "$AI_LOOP_CONFIRM"
```

The CLI must reject a missing, reused, or mismatched confirmation token. Expected: backup complete, old AI jobs paused, managed Vault tree replaced, runtime installed, two new AI jobs loaded, global blocks merged, hook merged, and local smoke tests green.

Then verify every installed module hash against the committed worktree and validate launchd:

```bash
launchctl print "gui/$(id -u)/com.harryd317.ai-session-sync"
launchctl print "gui/$(id -u)/com.harryd317.ai-session-fallback"
```

Expected: both jobs are loaded and their last exit status is `0` after kickstart.

- [ ] **Step 6: Publish source main before claiming cloud readiness**

Use `superpowers:finishing-a-development-branch` to integrate the implementation branch into local `main` without squashing the audit trail. Re-run Task 14 verification on `main`, then:

```bash
git push origin main
git rev-list --left-right --count origin/main...main
```

Expected: push succeeds and alignment is `0 0`.

- [ ] **Step 7: Prove real Codex → Claude continuation**

In the installed system, ensure the current Codex cycle contains a harmless explicit decision and todo used only for acceptance. Wait for `published`, then start a persisted Claude main session in the trusted repository:

```bash
claude -p --max-budget-usd 1.00 --permission-mode dontAsk --allowedTools 'Bash(ai-session-handoff *),Read' "Follow the global handoff instruction. State the current project, the sourced user decision, and the next todo. Add one new, harmless conclusion for Codex to continue; do not edit files or call external services."
```

Expected: Claude’s SessionStart binding uses a new authoritative session ID; its answer accurately reflects only the current project handoff; a `consumed` receipt appears; the new conclusion is later archived with the parent handoff hash; the first cycle becomes `已闭环` after publication.

- [ ] **Step 8: Prove real Claude → Codex continuation**

After the Claude conclusion is digested and published, start a new persisted Codex task:

```bash
codex exec -C "$PWD" -s workspace-write --add-dir "$HOME/.local/state/ai-session-sync" --add-dir "$HOME/obsidian-knowledge-base" "Follow the global AI handoff instruction. State the current project, Claude's sourced conclusion, the active user decision, and the next todo. Produce one harmless feedback sentence; do not edit files or call external services."
```

Expected: the new process has a distinct `CODEX_THREAD_ID`; it reads only current-project files; `consumed` records the second handoff; its feedback later archives with the parent hash; the second cycle becomes `已闭环`.

- [ ] **Step 9: Confirm status, Git alignment, and no collateral changes**

```bash
ai-session-sync status
git rev-list --left-right --count origin/main...main
git -C "$HOME/obsidian-knowledge-base" rev-list --left-right --count origin/main...main
```

Expected: the two acceptance cycles are `已闭环`, both repositories report `0 0`, publisher receipts contain pushed commit SHAs, and protected screener files/databases/jobs match their pre-activation hashes or metadata.

If any acceptance fails, run the manifest rollback command, preserve logs/receipts, add a failing regression test, fix in the worktree, and repeat from shadow mode.

- [ ] **Step 10: Commit delivery evidence**

Write the delivery note with test commands and counts, source commit, installed hashes, migration backup path, scan outcome, project counts, quarantine counts, two real cycle IDs/short hashes, publish commits, read receipts, rollback command, and proof of protected-system non-change. Do not include session text, source paths, identities, exact finance, aliases, credentials, or full receipt hashes.

```bash
git add docs/交付总结-全局AI接力闭环-20260721.md
git commit -m "交付: 全局AI分类接力闭环验收"
git push origin main
```

Expected: source `main` remains aligned with `origin/main` after the evidence commit.

---

### Task 16: Update durable operator documentation and perform final verification

**Files:**
- Modify: `README.md`
- Modify: `AGENTS.md`
- Modify: `MEMORY.md`
- Modify: `docs/superpowers/plans/2026-07-19-ai-session-sync.md`
- Modify: `docs/交付总结-全局AI接力闭环-20260721.md`

**Interfaces:**
- Operator documentation for status, catch-up, dry run, scan, install, backup, rollback, privacy boundaries, and receipt meanings.

- [ ] **Step 1: Document the operational surface**

Add concise commands for:

```text
ai-session-sync run
ai-session-sync catch-up --project <project-id>
ai-session-sync status
ai-session-handoff read --platform codex --session-id "$CODEX_THREAD_ID" --cwd "$PWD"
ai-session-handoff read --platform claude --cwd "$PWD"
```

Explain the six status labels, eight receipts, one-publisher rule, privacy bands, local-only quarantine, active global-file selection, Claude hook behavior, startup stale fallback, migration backup, and rollback. State explicitly that the system never authorizes trades, code changes, or external messages from handoff content.

- [ ] **Step 2: Update shared facts without bloating startup context**

Add only stable delivery facts to `MEMORY.md`: architecture, installed commands, job labels, state/Vault roots, receipt definition, verified date, rollback manifest location, and current operational caveats. Do not paste the design or delivery report into memory.

Mark the prior 2026-07-19 plan as superseded by this global closed-loop implementation while retaining its historical record.

- [ ] **Step 3: Run the complete fresh verification once more**

```bash
/usr/bin/python3 -m unittest -v test_ai_session_sync.py test_ai_session_sync_sources.py test_ai_session_sync_privacy.py test_ai_session_sync_projects.py test_ai_session_sync_classify.py test_ai_session_sync_archive.py test_ai_session_sync_handoff.py test_ai_session_sync_ledger.py test_ai_session_sync_migrate.py test_ai_session_sync_e2e.py test_obsidian_sync.py
.venv/bin/python test_rules.py
.venv/bin/python test_rules_mid.py
.venv/bin/python test_web.py
.venv/bin/python test_regime.py
bash -n install_ai_session_sync.sh
bash -n install_obsidian_sync.sh
```

Expected: every command exits `0`; Python suites end in `OK`.

- [ ] **Step 4: Audit the final diff and repository state**

```bash
git diff origin/main...main --check
git status --short --branch
git rev-list --left-right --count origin/main...main
```

Expected: no whitespace errors, clean `main`, and `0 0` alignment after the final documentation commit is pushed.

- [ ] **Step 5: Commit and push documentation if it changed after acceptance**

```bash
git add README.md AGENTS.md MEMORY.md docs/superpowers/plans/2026-07-19-ai-session-sync.md docs/交付总结-全局AI接力闭环-20260721.md
git commit -m "文档: 固化AI接力闭环运维与回滚说明"
git push origin main
```

If the index is empty, skip the commit and do not create an empty Git commit.

---

## Design Coverage Map

| Approved design sections | Implementation tasks |
| --- | --- |
| 1–4: goal, prior-design changes, known gaps, principles | Global constraints; Tasks 1–3 and 12 |
| 5: eight component boundaries | Planned file structure; Tasks 2–10 |
| 6–7: classification model and project registry | Tasks 4–5 |
| 8: S0–S3 privacy and quarantine | Task 3; migration scan in Task 13 |
| 9: Obsidian structure and one-copy originals | Task 7 |
| 10: project daily/current/global outputs | Task 8 |
| 11: explicit automatic handoff and read receipts | Tasks 10–11 |
| 12: five-minute, idle, rate-limit, fallback, and wake scheduling | Tasks 9 and 11 |
| 13: eight receipts and visible status | Tasks 6, 10, and 12 |
| 14–15: failure recovery, versioning, and idempotence | Tasks 6, 9, and 12–14 |
| 16: recoverable migration | Tasks 13 and 15 |
| 17: unit, integration, and real end-to-end tests | Tasks 2–14 and Task 15 Steps 7–8 |
| 18: thirteen delivery gates plus non-interference | Tasks 14–16 |
| 19: shadow, rebuild, cutover, rollback | Tasks 13 and 15 |
| 20: explicit non-goals | Global constraints and handoff documentation in Task 16 |

The work remains one cohesive implementation because the privacy type boundary, project artifact hashes, publisher observation, consumption receipt, and feedback parent hash form one cross-module contract. Each numbered task still ends in a focused, independently testable commit.

## Implementation Self-Review Checklist

Before starting Task 1, the implementing worker must confirm:

- [ ] Every approved design section 1–20 maps to at least one task above.
- [ ] Privacy is enforced before both Hermes and Vault; the audit rules are independent.
- [ ] Codex and Claude subagents plus platform-owned dynamic content are excluded before sanitization.
- [ ] Multi-project sessions create one original and multiple link-only indices.
- [ ] Registry corruption, low confidence, semantic project creation, and alias merge all have deterministic fallback behavior.
- [ ] Project digest, rolling handoff, and global overview have separate validators and privacy limits.
- [ ] `published` comes from the sole existing Vault publisher after push/alignment, not from a local file write.
- [ ] `consumed` requires authoritative identity; `feedback_archived` requires the consumed parent handoff hash.
- [ ] No command can label a seven-stage cycle `已闭环`.
- [ ] Every failure-matrix case prevents the next receipt and returns nonzero.
- [ ] Migration is recoverable, historical scans do not print matched text, and activation has a user checkpoint.
- [ ] Real Codex → Claude → Codex acceptance is required before final delivery.
- [ ] No task changes trading semantics, production databases, or screener service/jobs.

Run a placeholder and consistency scan on this plan itself:

```bash
rg -n "T[B]D|T[O]DO|FIXM[E]|implement[ ]later|fill[ ]this[ ]in|待[定]|以后[再]说" docs/superpowers/plans/2026-07-21-global-ai-handoff-loop.md
rg -n "RawMessage|SafeMessage|ClassifiedSegment|Receipt|feedback_archived|CODEX_THREAD_ID|SessionStart" docs/superpowers/plans/2026-07-21-global-ai-handoff-loop.md
```

Expected: the placeholder scan has no output; the contract scan returns the planned definitions and uses above.
