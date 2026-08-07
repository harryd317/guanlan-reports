# Industry Archive Kimi Channel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a Kimi `$web_search` channel as the default archive-draft generator, preserve explicit Claude alternatives, expose generation metadata, and keep secrets out of storage, errors, evidence, and the public mirror.

**Architecture:** Keep Kimi behind an archive-only adapter in `reviewer.py`; read its secret from process environment or a repository-external file. Persist only channel, model, and prompt version in `archive_queue`. Reuse the current service lock, v2 validator, failure rollback, equivalent environment, and mirror gates.

**Tech Stack:** Python 3.11, standard-library `urllib`, FastAPI, SQLite, vanilla HTML/CSS/JavaScript, `unittest`.

## Global Constraints

- Work only on `ui/today-two-lines-final`; do not merge or deploy.
- Default archive channel is `kimi`; `claude_code` and `api` remain explicit configuration choices.
- Default model is `kimi-k3`, verified against Moonshot official documentation on 2026-08-05.
- Every Kimi request declares builtin `$web_search`; a draft without an actual search tool call is rejected.
- Never write `KIMI_API_KEY` to code, tests, logs, SQLite, evidence, Git, or mirror.
- Never fall back silently after a Kimi error.
- Keep prompt version `industry-archive-v2.1-web`, the same skeleton, seven sections, stock limit, numeric-source rule, warning, and publish gate.
- Keep production databases and the T1 snapshot untouched.

---

### Task 1: External Kimi configuration and web-search adapter

**Files:**
- Modify: `reviewer.py`
- Test: `test_industry_archive_center.py`

**Interfaces:**
- Produces: `kimi_config(environ=None) -> dict[str, str]`
- Produces: `_ask_kimi_archive(prompt, system=None, timeout=None, allow_web=False) -> str`
- Produces: `industry_archive_draft(skeleton) -> {ok, content|error, channel, model, prompt_version}`

- [ ] **Step 1: Write failing configuration tests**

Add tests that patch `reviewer.KIMI_CONFIG_PATH`, process environment, and an external temporary file. Assert environment precedence, `kimi-k3` and official base defaults, missing-key failure, and that the repo `.env` is not a Kimi key source.

- [ ] **Step 2: Run the configuration tests and verify RED**

Run:

```bash
../../.venv/bin/python -m unittest \
  test_industry_archive_center.KimiArchiveChannelTests.test_external_config_and_environment_precedence \
  test_industry_archive_center.KimiArchiveChannelTests.test_missing_key_has_fixed_secret_safe_error
```

Expected: failure because `kimi_config` and the Kimi channel do not exist.

- [ ] **Step 3: Implement the external configuration reader**

Read `KIMI_API_KEY`, `KIMI_MODEL`, and `KIMI_BASE_URL` from process environment first. Read an explicit `KIMI_CONFIG_PATH`, otherwise `~/.config/guanlan/kimi.env`. Accept lowercase model/base keys in the file. Return only normalized values; never log them.

- [ ] **Step 4: Write the failing HTTP-loop test**

Mock `_http_post_json` with two complete OpenAI-compatible responses: first `finish_reason=tool_calls` with `$web_search`, then `finish_reason=stop` with a v2 draft. Assert every posted payload contains:

```python
{"type": "builtin_function", "function": {"name": "$web_search"}}
```

Assert the second payload contains the full assistant message and matching tool message. Add cases for direct stop without search, unknown tool, malformed response, loop limit, and HTTP failure without key fragments.

- [ ] **Step 5: Run the HTTP-loop tests and verify RED**

Run:

```bash
../../.venv/bin/python -m unittest test_industry_archive_center.KimiArchiveChannelTests
```

Expected: failures name the missing Kimi adapter and default channel.

- [ ] **Step 6: Implement the minimal Kimi adapter**

Post to `{kimi_base_url}/chat/completions` with model, messages, non-streaming output, and `$web_search` tools. Loop at most eight completions. Preserve every assistant field, echo parsed search arguments as tool content, require at least one search call, and return only final message content. Use fixed configuration-oriented errors; never include response bodies or exception text.

- [ ] **Step 7: Switch archive selection and verify GREEN**

Set archive default to `kimi`, add it to the web-capable archive choices, and keep Claude adapters unchanged. Run `KimiArchiveChannelTests` and existing `ArchiveDraftPromptTests`.

- [ ] **Step 8: Commit Task 1**

```bash
git add reviewer.py test_industry_archive_center.py
git commit -m "feat(archive): add Kimi web-search draft channel"
```

### Task 2: Persist channel/model metadata and preserve rollback

**Files:**
- Modify: `s8_research/db.py`
- Modify: `s8_research/archives.py`
- Modify: `s8_research/read_model.py`
- Modify: `service.py`
- Test: `test_industry_archive_center.py`
- Test: `test_s8_research.py`

**Interfaces:**
- Extends: `archive_queue.generation_channel TEXT`
- Extends: `archive_queue.generation_model TEXT`
- Extends: `complete_archive_generation(..., generation_channel=None, generation_model=None)`

- [ ] **Step 1: Write failing lifecycle tests**

Assert migrations add both columns. Assert success writes `kimi`, `kimi-k3`, and the frozen prompt version. Patch the service generator to fail with a secret-bearing internal exception and assert the row returns to `skeleton_ready`, the response tells the operator to check Kimi configuration, and neither stored nor returned error contains any key substring.

- [ ] **Step 2: Run lifecycle tests and verify RED**

Run:

```bash
../../.venv/bin/python -m unittest \
  test_industry_archive_center.ArchiveLifecycleTests \
  test_s8_research.S8DatabaseTests
```

Expected: missing columns, missing function parameters, and missing rollback behavior.

- [ ] **Step 3: Implement schema and lifecycle updates**

Add both nullable columns to schema and migrations. Clear old generation metadata when claiming a new run. Write metadata only after the v2 validator accepts the draft. Keep `fail_archive_generation` on `skeleton_ready` and scrub before truncation.

- [ ] **Step 4: Wire service metadata and fixed errors**

Pass `result["channel"]` and `result["model"]` into completion. Return them in a successful response. Preserve the current 409 failure response and single-flight claim; replace arbitrary exception text with a secret-safe Kimi configuration message for Kimi failures.

- [ ] **Step 5: Run lifecycle tests and verify GREEN**

Run the Task 2 tests, then the complete `test_industry_archive_center.py` and `test_s8_research.py` files.

- [ ] **Step 6: Commit Task 2**

```bash
git add s8_research/db.py s8_research/archives.py s8_research/read_model.py service.py \
  test_industry_archive_center.py test_s8_research.py
git commit -m "feat(archive): persist Kimi generation metadata"
```

### Task 3: Show generation provenance on the awaiting-review page

**Files:**
- Modify: `static/home.html`
- Test: `test_guanlan_frontend.py`
- Test: `test_guanlan_frontend_contract.py`

**Interfaces:**
- Consumes: queue fields `generation_channel`, `generation_model`, `prompt_version`
- Produces: visible text `生成通道：kimi · kimi-k3 · Prompt industry-archive-v2.1-web`

- [ ] **Step 1: Write a failing UI behavior test**

Render the awaiting-review state with metadata and assert the fixed warning remains and the provenance line appears beside it. Assert missing legacy metadata yields an honest placeholder without JavaScript errors.

- [ ] **Step 2: Run UI tests and verify RED**

```bash
../../.venv/bin/python -m unittest \
  test_guanlan_frontend.GuanlanFrontendTests \
  test_guanlan_frontend_contract.GuanlanFrontendContractTests
```

- [ ] **Step 3: Implement the provenance line**

Add a small inline metadata span within `.draft-warning`. Escape all three values. Keep the warning text and final-operation cards unchanged.

- [ ] **Step 4: Run UI tests and verify GREEN**

Run both UI files and verify no existing zero-scroll or warning contracts regress.

- [ ] **Step 5: Commit Task 3**

```bash
git add static/home.html test_guanlan_frontend.py test_guanlan_frontend_contract.py
git commit -m "feat(ui): show archive generation provenance"
```

### Task 4: Extend mirror secret blocking for Kimi credentials

**Files:**
- Modify: `scripts/report_mirror_sync.py`
- Test: `test_report_mirror_sync.py`

**Interfaces:**
- Extends: generic `sk-` secret category to name Kimi explicitly
- Preserves: redacted finding output without secret values

- [ ] **Step 1: Write the failing secret test**

Create a synthetic Kimi-style `sk-` value only inside a temporary mirror fixture. Assert one blocked finding, a Kimi-aware category, and absence of the secret from `repr(result)`.

- [ ] **Step 2: Run the test and verify RED**

```bash
../../.venv/bin/python -m unittest \
  test_report_mirror_sync.ReportMirrorSyncTests.test_secret_gate_names_kimi_keys
```

- [ ] **Step 3: Implement and verify GREEN**

Rename the generic `sk-` category to include Kimi while retaining the broad pattern. Run all mirror tests.

- [ ] **Step 4: Commit Task 4**

```bash
git add scripts/report_mirror_sync.py test_report_mirror_sync.py
git commit -m "fix(mirror): identify Kimi keys in secret gate"
```

### Task 5: Conditional real integration, equivalent UI, regressions, and delivery

**Files:**
- Create: `docs/验收-档案生成kimi通道-20260805.md`
- Create: `reports/evidence/industry-archive-kimi-20260805/integration.json`
- Create only when a real call succeeds: `reports/evidence/industry-archive-kimi-20260805/draft.md`
- Update as generated evidence: `reports/evidence/today-two-lines-final/equivalent/global-zero-scroll.json`

**Interfaces:**
- Evidence contains channel, model, prompt version, section validation, source/date validation, draft SHA-256, and secret scan count; it never contains credentials.

- [ ] **Step 1: Detect Kimi configuration without printing values**

Call `reviewer.kimi_config()` and print only `configured=true|false`, model, base host, and external config path. If absent, write `integration.json` with `status=skipped`, reason `未执行：缺 KIMI_API_KEY`, and the two supported configuration methods.

- [ ] **Step 2: Run real integration only when configured**

Use a real industry skeleton from the read-only equivalent data and call the Kimi archive adapter. Validate seven sections, stock count, at least one dated source in section seven, and prompt version. Save a redacted draft and verification JSON. Scan both for the live key in memory before writing evidence.

- [ ] **Step 3: Re-run the production-equivalent zero-scroll matrix**

Start the cloned LaunchAgent on an independent port with read-only database copies. Fully reload every frozen route at 393×852, 430×900, and 1280×900. Require 36/36 document dimension equality, zero console errors, and unchanged production/copy hashes. Stop the clone and verify its port is closed.

- [ ] **Step 4: Run all required regressions**

```bash
../../.venv/bin/python -m unittest \
  test_guanlan_frontend.py test_guanlan_frontend_contract.py \
  test_industry_archive_center.py test_production_equivalent.py \
  test_s8_research.py test_ui_fossil_cleanup.py
../../.venv/bin/python test_web.py
../../.venv/bin/python test_rules.py
../../.venv/bin/python test_rules_mid.py
../../.venv/bin/python test_regime.py
../../.venv/bin/python -m unittest test_report_mirror_sync.py
PYTHONPYCACHEPREFIX=/tmp/guanlan_kimi_compile python3 -m compileall -q reviewer.py service.py s8_research scripts
git diff --check
```

- [ ] **Step 5: Write and commit the final report**

Record configuration names, external path, channel logic, integration outcome, exact regression counts, zero-scroll matrix, unchanged hashes, rollback, source commits, and “not merged/not deployed.” Stage no config file or secret.

- [ ] **Step 6: Sync and verify the public mirror**

Publish the branch through the existing HTTPS mirror alias, clone the remote into a new temporary directory, verify report/evidence hashes, run the full mirror secret scan with zero blocked findings, and report the new mirror HEAD.
