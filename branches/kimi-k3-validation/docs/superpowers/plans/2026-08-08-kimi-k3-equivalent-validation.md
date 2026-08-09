# Kimi K3 Production-Equivalent Validation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Prove that `kimi-k3` can generate the 动物保健Ⅱ archive through the existing production-equivalent v2.2 pipeline, then switch only the external model setting if every frozen gate passes.

**Architecture:** Keep production code, LaunchAgent plist, and databases unchanged. Run the production LaunchAgent clone from the authoritative production working directory with read-only SQLite snapshots and one disposable S8 writer copy; record only secret-free request/response metadata. Test `tool_choice="required"` separately, then run the full archive generation with the unchanged K2.6 skeleton and Prompt path using only `KIMI_MODEL=kimi-k3` as an environment override.

**Tech Stack:** Python 3.11, FastAPI service functions, SQLite read-only snapshots, macOS LaunchAgent clone, Moonshot Chat Completions, `$web_search`, repository mirror sanitizer.

**Execution result (2026-08-09):** The plan ran to its atomic model gate. K3 passed the first real `$web_search` round but its Chat Completions tool continuation returned HTTP 400, so no draft was accepted, the external model stayed on K2.6, and production was not changed. See `docs/验收-档案生成切换K3-20260808.md`.

## Global Constraints

- Do not trigger production archive generation or write any production database.
- Never print, persist, compare, or mirror the Kimi key value or any fragment.
- Keep the production LaunchAgent plist byte-identical.
- The full K3 run must use the same skeleton builder, `industry-archive-v2.2-web` Prompt, seven-section validator, and core-stock limit as K2.6.
- Record every retry, skip, degradation, API failure category, and validation failure.
- Change `$HOME/.config/guanlan/kimi.env` only after all K3 gates pass; modify only `KIMI_MODEL`, preserving mode `0600` and all other values.
- A successful task requires a committed final report and a fresh-clone-verified report-mirror hash.

---

### Task 1: Freeze the production-equivalent boundary

**Files:**
- Create at runtime: `.runtime/kimi-k3-validation-20260808/**`
- Create: `reports/evidence/kimi-k3-validation-20260808/before-hashes.json`

**Interfaces:**
- Consumes: production plist `com.rui.stock-screener.web.plist`, production working directory, `scripts.t7_equivalent_env.snapshot_sqlite`, and `clone_launch_agent`.
- Produces: read-only copies of `screener.db`, `market.db`, thermometer sidecar, and `S8_research.db`; a disposable S8 writer; LaunchAgent clone on an independent port.

- [ ] **Step 1: Record production hashes and the current safe model state**

Record the plist and four database SHA-256 values. In a LaunchAgent clone process, report only model name, config readability, mode, and key character length.

- [ ] **Step 2: Create consistent read-only SQLite snapshots**

Use SQLite backup, run `PRAGMA integrity_check`, set protected copies to `0444`, and create a separate disposable S8 writer from the protected S8 copy.

- [ ] **Step 3: Start the exact LaunchAgent clone**

Use the production `WorkingDirectory` and `start.sh`, set an independent label and port, point every database environment variable to the copies, set `SCREENER_EQUIVALENT_MODE=1`, and set `KIMI_MODEL=kimi-k3` only in the clone.

- [ ] **Step 4: Verify health and isolation**

Require `ok=true`, matching boot/disk fingerprints, production port still healthy, and no changed production or protected-copy hash.

### Task 2: Evaluate `tool_choice="required"`

**Files:**
- Create: `reports/evidence/kimi-k3-validation-20260808/tool-choice-required.json`

**Interfaces:**
- Consumes: official Kimi Chat Completions contract, clone process configuration, and `$web_search` declaration.
- Produces: a secret-free compatibility result with HTTP category, finish reason, returned tool names, duration, and usage totals.

- [ ] **Step 1: Freeze the official documentation evidence**

Record that the official Chat API lists `required`, that the official web-search guide declares `$web_search` as `builtin_function`, and that K3 is the recommended current model.

- [ ] **Step 2: Run one minimal real request**

Send `model=kimi-k3`, one `$web_search` tool declaration, and `tool_choice="required"`. Record only safe metadata and token counts; never persist the request authorization value or response body.

- [ ] **Step 3: Decide whether to enable it**

Enable only if the real API accepts it and it can be limited to the pre-search round without forcing the final drafting round back into a tool loop. Otherwise retain the existing fail-closed prompt-plus-observed-tool gate and document why.

### Task 3: Run the full K3 archive generation

**Files:**
- Create: `reports/evidence/kimi-k3-validation-20260808/real-generation.json`
- Create: `reports/evidence/kimi-k3-validation-20260808/动物保健Ⅱ-kimi-k3-v2.2-等效初稿.md`

**Interfaces:**
- Consumes: `service.api_s8_generate_archive`, the existing skeleton builder, Prompt v2.2, the disposable S8 writer, and K3 clone config.
- Produces: one persisted draft in the disposable queue plus safe request trace and usage data.

- [ ] **Step 1: Capture the exact pre-call skeleton fingerprint**

Hash canonical JSON for the generated K3 skeleton and compare it with the K2.6 evidence inputs where available. Record Prompt version and required skeleton keys.

- [ ] **Step 2: Instrument safe request metadata**

Wrap `_http_post_json` only inside the one-shot clone process. Record elapsed seconds, model, declared tools, `finish_reason`, returned tool names, response `usage`, and `$web_search` argument `usage.total_tokens`. Record only whether an authorization header exists.

- [ ] **Step 3: Invoke the real service generation function**

Call the same service function used by the page against the disposable S8 writer. Do not call the production endpoint. Record every attempt and its actual result.

- [ ] **Step 4: Enforce both gates**

Require a real `$web_search` tool call and a completed tool-result continuation before accepting content. Then require all seven sections, at most five core stocks, Prompt v2.2, injected thermometer/cycle fields, dated HTTPS sources, and the draft warning.

### Task 4: Compare K3 with the accepted K2.6 draft

**Files:**
- Create: `reports/evidence/kimi-k3-validation-20260808/quality-comparison.json`

**Interfaces:**
- Consumes: the K2.6 final draft/evidence and the K3 final draft/evidence.
- Produces: objective counts plus a concise qualitative comparison.

- [ ] **Step 1: Compute objective metrics**

Compare elapsed time where available, total/prompt/completion/search tokens, non-whitespace length, source count, dated HTTPS source count, core-stock count, missing-field honesty markers, and section validation.

- [ ] **Step 2: Review material quality differences**

Compare source authority and recency, skeleton discipline, code/industry identity discipline, cycle-position depth, falsification specificity, internal consistency, and unsupported-number risk. Record strengths and regressions without selecting the winner in advance.

### Task 5: Apply the success/failure model rule

**Files:**
- Modify on success only: `$HOME/.config/guanlan/kimi.env`
- Create: `reports/evidence/kimi-k3-validation-20260808/config-after.json`

**Interfaces:**
- Consumes: all Task 3 gates and current external config.
- Produces: either `KIMI_MODEL=kimi-k3` with mode `0600`, or an unchanged K2.6 setting with a recorded failure reason.

- [ ] **Step 1: Evaluate the full gate atomically**

Treat any missing search trace, incomplete tool continuation, validator error, core-stock overflow, source-date failure, or hidden degradation as failure.

- [ ] **Step 2: Update only the model line on success**

Perform a mode-preserving atomic replacement outside the repository. Do not expose any other line. On failure, do not write the config file.

- [ ] **Step 3: Confirm effective config in LaunchAgent context**

Use a fresh LaunchAgent clone probe. Report only resolved model, config readability, mode, parse status, and key character length. Confirm the production plist and databases remain unchanged.

### Task 6: Verify, document, commit, and mirror

**Files:**
- Create: `docs/验收-档案生成切换K3-20260808.md`
- Create: `reports/evidence/kimi-k3-validation-20260808/after-hashes.json`
- Create: `reports/evidence/kimi-k3-validation-20260808/secret-scan.json`

**Interfaces:**
- Consumes: all evidence and project completion rules.
- Produces: final Git commit and fresh-clone-verified mirror commit.

- [ ] **Step 1: Run the full regression suite**

Run the six-file core unittest suite, `test_web.py`, `test_rules.py`, `test_rules_mid.py`, `test_regime.py`, `test_report_mirror_sync.py`, `compileall`, and `git diff --check`.

- [ ] **Step 2: Recheck all hashes and stop the clone**

Require the production plist, all production databases, and all protected copies to match their before hashes. Stop the cloned agents and prove their ports are closed.

- [ ] **Step 3: Write the honest acceptance report**

Include official-document findings, every real attempt, durations and token use, K2.6 comparison, `tool_choice` decision, config result, regression results, hash table, and rollback method.

- [ ] **Step 4: Run the mirror secret gate**

Require zero blocked findings, including Kimi-style `sk-` patterns. Do not load or record the external Kimi key value in evidence.

- [ ] **Step 5: Commit and publish**

Commit only the plan, final report, and sanitized evidence. Push the `codex/kimi-k3-validation` docs/reports mirror over HTTPS, clone the remote fresh, confirm required files, and report the verified mirror hash.
