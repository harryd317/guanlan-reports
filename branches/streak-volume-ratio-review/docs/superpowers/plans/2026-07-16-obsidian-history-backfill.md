# Obsidian Historical Backfill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Backfill every committed screener fact from `112102a..d621afb` into Obsidian with a 71-row audit trail, source-document mirror, readable topic notes, and corrected living documentation.

**Architecture:** Freeze the historical range at `d621afb`, derive coverage from Git, mirror every changed Markdown source, then write topic notes that cite commits and source files. Keep raw evidence separate from summaries and review the finished vault with Hermes before enabling continuous automation.

**Tech Stack:** Git, Markdown, POSIX shell utilities, existing `~/.local/bin/hermes-review`, Obsidian vault Git repository.

## Global Constraints

- Source: `~/云慧养/zmu_gitee/screener`.
- Vault: `~/obsidian-knowledge-base`.
- Frozen range: `112102a..d621afb`; expected count: exactly 71.
- Later commits `ea3354d` and `77ec779` belong to incremental sync.
- Preserve the user's untracked source file `AGENTS.md`.
- Exclude code, databases, logs, HTML, screenshots, `.env`, `.token`, and credentials.
- Preserve old conclusions and label them completed, superseded, or disproved.
- Use `Asia/Shanghai` dates.
- Advance the historical cursor only after vault `main` equals `origin/main`.

---

### Task 1: Build the 71-commit coverage ledger

**Files:**
- Create: `~/obsidian-knowledge-base/10-项目/screener/2026-07-12至16-同步清单.md`
- Modify: `~/obsidian-knowledge-base/10-项目/screener/README.md`

**Interfaces:**
- Consumes: source Git range `112102a..d621afb`.
- Produces: one immutable row per commit; later tasks fill topic and status columns.

- [ ] **Step 1: Verify the frozen boundary**

```bash
git -C '~/云慧养/zmu_gitee/screener' rev-list --count 112102a..d621afb
git -C '~/云慧养/zmu_gitee/screener' rev-parse 112102a c4d4338 d621afb
```

Expected:

```text
71
112102a8e09f8d16d16685d7ebe7f59912ebfa57
c4d433875cdbc1fa23a9fd68a07b0ff320696ee6
d621afb336bd4cec26634bcd755e64650278065e
```

- [ ] **Step 2: Render and inspect all rows**

```bash
git -C '~/云慧养/zmu_gitee/screener' \
  log --reverse --date=iso-local \
  --format='| `%h` | %ad | %s | 待分类 | 待映射 | 已登记 |' \
  112102a..d621afb
```

Expected: 71 rows; first `c4d4338`, last `d621afb`.

- [ ] **Step 3: Create the ledger**

Start with:

```markdown
# screener 历史缺口同步清单（2026-07-12～07-16）

> 源范围：`112102a..d621afb` · 71 次提交
> 首个缺失：`c4d4338` · 固定终点：`d621afb`
> 本表是覆盖审计，不代替主题笔记和源材料。

| commit | 时间 | 标题 | 类型 | 证据文件 | 主题笔记 | 状态 |
|---|---|---|---|---|---|---|
```

Classify titles with these rules:

- `修复` or `补刀` → `修复`;
- `研究`、`实验`、`回测`、`审判` or `复盘` → `研究`;
- `CLAUDE.md补记` or `交付总结` → `文档`;
- all remaining feature commits → `交付`.

Use `git show --name-only --format= <sha> -- '*.md'` for evidence. Leave topic as `待映射`; set state to `已登记`.

- [ ] **Step 4: Link the audit surfaces from README**

```markdown
## 同步审计

- [[2026-07-12至16-同步清单]]：71 次历史缺口的逐提交覆盖表。
- [[变更流水]]：自动同步启用后的增量账本。
- `源材料/`：从 screener 镜像的只读 Markdown 证据。
```

- [ ] **Step 5: Verify uniqueness and count**

```bash
python3 - <<'PY'
from pathlib import Path
import re
p = Path('~/obsidian-knowledge-base/10-项目/screener/2026-07-12至16-同步清单.md')
shas = re.findall(r'^\| `([0-9a-f]{7})` \|', p.read_text(), re.M)
assert len(shas) == len(set(shas)) == 71
assert shas[0] == 'c4d4338' and shas[-1] == 'd621afb'
print('coverage 71/71')
PY
```

Expected: `coverage 71/71`.

- [ ] **Step 6: Commit the ledger**

```bash
git -C '~/obsidian-knowledge-base' add \
  '10-项目/screener/2026-07-12至16-同步清单.md' \
  '10-项目/screener/README.md'
git -C '~/obsidian-knowledge-base' commit \
  -m 'screener: 建立71提交历史同步清单'
```

### Task 2: Mirror every changed Markdown source

**Files:**
- Create: `~/obsidian-knowledge-base/10-项目/screener/源材料/README.md`
- Create: `~/obsidian-knowledge-base/10-项目/screener/源材料/CLAUDE.md`
- Create: `~/obsidian-knowledge-base/10-项目/screener/源材料/docs/*.md` (31 files)

**Interfaces:**
- Consumes: 32 Markdown paths changed in the frozen range.
- Produces: byte-identical source evidence at commit `d621afb`.

- [ ] **Step 1: Generate the path manifest**

```bash
git -C '~/云慧养/zmu_gitee/screener' \
  -c core.quotePath=false diff --name-only --diff-filter=ACMRT \
  112102a..d621afb -- '*.md' > /tmp/screener-backfill-md.txt
wc -l /tmp/screener-backfill-md.txt
```

Expected: `32 /tmp/screener-backfill-md.txt`.

- [ ] **Step 2: Reject forbidden paths**

```bash
if rg -n '(\.env|\.token|\.db($|-)|positions\.csv|id_ed25519|web\.log)' \
  /tmp/screener-backfill-md.txt; then
  echo 'forbidden path detected'; exit 1
fi
```

Expected: no output.

- [ ] **Step 3: Create the mirror README**

```markdown
# screener 源材料镜像

> 本目录由同步流程管理。文件来自 screener Git，只用于事实追溯。

- 不在这里手工编辑；修改先发生在 screener，再由同步流程带入。
- 只收 Markdown，不含代码、数据库、日志、截图或凭证。
- 历史回填范围：`112102a..d621afb`。
- 主题笔记必须引用这里的文件或同步清单中的提交号。
```

- [ ] **Step 4: Copy frozen source versions**

```bash
SRC='~/云慧养/zmu_gitee/screener'
DST='~/obsidian-knowledge-base/10-项目/screener/源材料'
while IFS= read -r path; do
  mkdir -p "$DST/$(dirname "$path")"
  git -C "$SRC" show "d621afb:$path" > "$DST/$path"
done < /tmp/screener-backfill-md.txt
```

- [ ] **Step 5: Verify count and byte identity**

```bash
python3 - <<'PY'
from pathlib import Path
import subprocess
src = '~/云慧养/zmu_gitee/screener'
dst = Path('~/obsidian-knowledge-base/10-项目/screener/源材料')
paths = Path('/tmp/screener-backfill-md.txt').read_text().splitlines()
assert len(paths) == 32
for rel in paths:
    want = subprocess.check_output(['git', '-C', src, 'show', f'd621afb:{rel}'])
    assert (dst / rel).read_bytes() == want, rel
print('mirror 32/32')
PY
```

Expected: `mirror 32/32`.

- [ ] **Step 6: Commit the mirror**

```bash
git -C '~/obsidian-knowledge-base' add \
  '10-项目/screener/源材料'
git -C '~/obsidian-knowledge-base' commit \
  -m 'screener: 镜像历史缺口Markdown证据'
```

### Task 3: Write strategy, regime, and validation notes

**Files:**
- Create: `~/obsidian-knowledge-base/10-项目/screener/策略中心与市况系统.md`
- Create: `~/obsidian-knowledge-base/10-项目/screener/策略背书-失败实验与影子验证.md`
- Create: `~/obsidian-knowledge-base/10-项目/screener/2026-07-12至16-项目总览.md`
- Modify: `~/obsidian-knowledge-base/10-项目/screener/2026-07-12至16-同步清单.md`

**Interfaces:**
- Consumes: P1–P4 source mirror and commit ledger.
- Produces: self-contained architecture, evidence, and dated status notes.

- [ ] **Step 1: Write `策略中心与市况系统.md`**

Include these exact facts:

- P1: base protocol, registry, route_id, short/mid plugins, 11 orchestration takeover points, rule functions unchanged, Golden Master byte-identical, tests `8/20/439`.
- P2: independent `market.db`; point-in-time market history from 2015; direction and risk axes; 15-day confirmation; train period 2015–2021; OOS 2022 onward; about 4 flips/year, median 34 days, KW `p<1e-4`; six Chinese regimes.
- P4: daily shadow decisions in `strategy_switches`; no regime cell endorsed; record-only; activation requires 2026-10-09 review and new execution code.
- Evidence: mirrored P1/P2 plans, regime experiment, and commits `c4d4338`, `ea4eb70`, `d9a3af5`, `b4ab908`.

- [ ] **Step 2: Write `策略背书-失败实验与影子验证.md`**

Include:

- P3a OOS 8 cells all gray; both cards lose in “冲高有险”.
- P3b 36 combinations authorize zero switches; current middle-term gate beats no-gate.
- entry_B mild relaxations produce zero signals; the “parameters are merely too strict” claim is disproved; extreme `+2.64%` remains a redesign clue.
- defense v0.1 passes only 3/4 predeclared OOS drawdown gates and is terminated.
- partial-hold grid: 15/36 pass; selected `U30/f⅔/dd25`; `Δ+542.5pp`; excluding best year `+234.2pp`; shadow only.
- Evidence commits: `9b2cdfd`, `5cdb421`, `385febd`, `d2d04a6`, `5114831`.

- [ ] **Step 3: Write the dated overview**

Use dated sections:

- 07-12: P1 plugin layer.
- 07-13: OCR, outside ledger, fee alignment, P2 regime.
- 07-14: P3/P4, gate sentinel, defense failure, bull-stock evidence.
- 07-15: redesign, long-hold research, ladder/reaffirmation, tasks 17–21.
- 07-16: outside-stock analyst path through `d621afb`.

End state: tests `8/20/693/9`, real closed validation trades 0 unless production evidence says otherwise, three parameters frozen through 2026-10-09, P4 shadow-only.

- [ ] **Step 4: Map related ledger rows**

Replace `待映射` for P1–P4, defense, and long-hold experiment commits with one of these notes. Change state to `已摘要`.

- [ ] **Step 5: Commit**

```bash
git -C '~/obsidian-knowledge-base' add \
  '10-项目/screener/策略中心与市况系统.md' \
  '10-项目/screener/策略背书-失败实验与影子验证.md' \
  '10-项目/screener/2026-07-12至16-项目总览.md' \
  '10-项目/screener/2026-07-12至16-同步清单.md'
git -C '~/obsidian-knowledge-base' commit \
  -m 'screener: 沉淀策略中心与市况证据'
```

### Task 4: Write execution, research, and product notes

**Files:**
- Create: `~/obsidian-knowledge-base/10-项目/screener/截图记账与场外老仓.md`
- Create: `~/obsidian-knowledge-base/10-项目/screener/拿住牛票与阶梯留仓研究.md`
- Create: `~/obsidian-knowledge-base/10-项目/screener/全站重设计与事故复盘.md`
- Create: `~/obsidian-knowledge-base/10-项目/screener/2026-07-16-池外AI查询升级.md`
- Modify: `~/obsidian-knowledge-base/10-项目/screener/2026-07-12至16-同步清单.md`

**Interfaces:**
- Consumes: OCR/outside, long-hold, redesign, and outside-analysis evidence.
- Produces: four standalone notes and complete mapping for remaining rows.

- [ ] **Step 1: Write `截图记账与场外老仓.md`**

Include:

- OCR images remain in memory and never land on disk.
- Model rows cross-check code, price range, action, and date; confirmation still uses existing accounting routes.
- Missing code is filled only by exact unique normalized-name match.
- Recognized trade date drives accounting; future dates and sells before buys are rejected.
- Outside positions use strategy `manual`; they count toward total and industry exposure but stay out of strategy engines, validation statistics, and equity curves.
- Partial close, delete, clear-all, AI “留/走”, and historical review are supported.
- Cite commits `3fc85cc` through `1094a4d` and the mirrored OCR/outside plans.

- [ ] **Step 2: Write `拿住牛票与阶梯留仓研究.md`**

Include:

- Initial momentum-top-5 pyramid replay lost on 15/15 examples and was disproved.
- Second replay found first-capture + within 5% of platform + 40–90% momentum band reduced damage (`+1.4pp` versus `-160.8pp`) but did not authorize a live strategy.
- The predeclared 36-grid partial-hold experiment selected `U30/f⅔/dd25` and entered shadow tracking only.
- Capacity limits and reaffirmation cards were deployed without touching frozen entry/exit parameters.
- At least 20 shadow cases are required before structural conclusions.
- Cite `0f34a2c`, `81a1321`, `15bc020`, `f43b4b5`, `5114831`, `fec0842`.

- [ ] **Step 3: Write `全站重设计与事故复盘.md`**

Include:

- Tasks 17–21 unified navigation and design language across ten pages.
- Pool became “我的池子 / 盘中监控 / 筛选结果”; ask and trade changed visual style without changing behavior.
- `/pro` returns 307 to `/pool`; the old page was deleted.
- A cross-line regex deleted a critical Home rendering node and caused a production white screen.
- Permanent safeguards: no cross-line regex for structural HTML edits, inspect staged diff, and run rendered E2E before production.
- Cite `45fc0f1` through `c2b3cfe` and delivery summaries 17–21.

- [ ] **Step 4: Write `2026-07-16-池外AI查询升级.md`**

Include:

- The old outside-stock review path injected no facts.
- The dedicated path injects company, industry, price/technical, and available financial context.
- Persona changed from discipline police to a Buffett/Duan Yongping-style company analyst; the screener itself remains trend-following and does not make value judgments.
- Concrete follow-ups use conversational mode instead of replaying a fixed five-section report.
- `outside_chats` keeps the latest eight same-stock, same-day turns and isolates different stocks/days.
- Cite `9497e19`, `e36f3e4`, `0983989`, `d621afb`.

- [ ] **Step 5: Complete ledger mapping**

Every remaining row must point to one of the seven topic notes or the dated overview. No `待映射` may remain; all rows become `已摘要`.

- [ ] **Step 6: Commit**

```bash
git -C '~/obsidian-knowledge-base' add \
  '10-项目/screener/截图记账与场外老仓.md' \
  '10-项目/screener/拿住牛票与阶梯留仓研究.md' \
  '10-项目/screener/全站重设计与事故复盘.md' \
  '10-项目/screener/2026-07-16-池外AI查询升级.md' \
  '10-项目/screener/2026-07-12至16-同步清单.md'
git -C '~/obsidian-knowledge-base' commit \
  -m 'screener: 补齐执行研究与产品演变'
```

### Task 5: Correct living documentation and stale status

**Files:**
- Modify: `~/obsidian-knowledge-base/10-项目/screener/交付历史.md`
- Modify: `~/obsidian-knowledge-base/10-项目/screener/系统全貌.md`
- Modify: `~/obsidian-knowledge-base/10-项目/screener/待办与冻结纪律.md`
- Modify: `~/obsidian-knowledge-base/30-知识/投资纪律-系统验证笔记.md`
- Modify: `~/obsidian-knowledge-base/30-知识/操盘手审计-核心结论.md`

**Interfaces:**
- Consumes: seven topic notes and dated overview.
- Produces: current living notes without erasing historical reasons.

- [ ] **Step 1: Update delivery history through `d621afb`**

Append dated sections for P1, OCR/outside ledger, P2–P4, defense failure, long-hold research, tasks 17–21, and outside analysis. Set the frozen endpoint baseline to `8/20/693/9`.

- [ ] **Step 2: Update the system map**

Apply these exact corrections:

- `/pro` → 307 redirect to `/pool`;
- add `/strategy` and its evidence role;
- add `market_data.py`, `regime.py`, `switcher.py`, `ladder_shadow.py`, and `profile.py`;
- distinguish strategy holdings from `manual` outside holdings;
- record the four-test command for rules, mid-rules, web, and regime;
- state that regime and switcher cannot execute trades.

- [ ] **Step 3: Correct stale todos and freeze statements**

Mark completed: account drawdown `-8%` circuit breaker, same-industry `≤30%` cap, `RISK_PCT=1%`, honest gap-risk wording, screenshot recognition, entry_B decomposition experiment, and performance statistics card.

Keep active: three parameters frozen through 2026-10-09; P4 activation prohibited before review; real-trade validation still zero unless live data proves otherwise; OpenAI and Server酱 credential rotation unresolved unless separately verified.

- [ ] **Step 4: Preserve recommendation history**

Change “应改为账户回撤熔断” to “已于 2026-07-12 改为账户回撤 -8% 熔断”; keep the false-positive analysis as the reason. Mark the audit's three recommendations implemented.

- [ ] **Step 5: Run stale-string checks**

```bash
V='~/obsidian-knowledge-base'
rg -n '截图识别记账（AI 视觉）|entry_B.*9 月做|/pro.*旧专业看板|待用户拍板.*回滚线|应改为账户回撤熔断' \
  "$V/10-项目/screener" "$V/30-知识"
```

Expected: no active-status matches. Historical quotations may remain only when immediately labeled completed or superseded.

- [ ] **Step 6: Commit**

```bash
git -C '~/obsidian-knowledge-base' add \
  '10-项目/screener/交付历史.md' \
  '10-项目/screener/系统全貌.md' \
  '10-项目/screener/待办与冻结纪律.md' \
  '30-知识/投资纪律-系统验证笔记.md' \
  '30-知识/操盘手审计-核心结论.md'
git -C '~/obsidian-knowledge-base' commit \
  -m 'screener: 校正当前状态与冻结纪律'
```

### Task 6: Audit, review, and publish the backfill

**Files:**
- Create through existing tool: `~/obsidian-knowledge-base/20-AI复核/2026-07-16-screener历史缺口回填.md`
- Create after success: `~/.local/state/screener-vault-mirror.head`
- Create after success: `~/.local/state/screener-vault-digest.head`

**Interfaces:**
- Consumes: complete backfill.
- Produces: independent review record, clean remote, and frozen endpoint for incremental sync.

- [ ] **Step 1: Run coverage and secret scans**

```bash
V='~/obsidian-knowledge-base'
python3 - <<'PY'
from pathlib import Path
import re
p = Path('~/obsidian-knowledge-base/10-项目/screener/2026-07-12至16-同步清单.md')
t = p.read_text()
shas = re.findall(r'^\| `([0-9a-f]{7})` \|', t, re.M)
assert len(shas) == len(set(shas)) == 71
assert '待映射' not in t and '待摘要' not in t
print('coverage and mapping OK')
PY
rg -n '(sk-[A-Za-z0-9_-]{16,}|Bearer +[A-Za-z0-9._-]{16,}|API_SERVER_KEY=.+|SENDKEY=.+)' \
  "$V/10-项目/screener" "$V/30-知识" && exit 1 || true
git -C "$V" diff --check
```

Expected: `coverage and mapping OK`, no secret matches, no whitespace errors.

- [ ] **Step 2: Prepare and send the Hermes packet**

The packet must state: frozen range, 71/71 result, seven topic notes, five corrected living notes, stale/secret scan results, and that `ea3354d` plus later commits remain incremental.

Create `/tmp/screener-backfill-review.md` with `apply_patch` using this exact skeleton, replacing only the two scan-result lines with captured command output:

```markdown
# screener 历史缺口回填复核包

## 固定边界
- 范围：`112102a..d621afb`
- 首个缺失：`c4d4338`
- 固定终点：`d621afb`
- 覆盖：71/71，提交号唯一，顺序与 Git 一致
- `ea3354d`、`77ec779` 及其后的提交不混入历史范围，交给增量同步

## 新增知识
- 七篇主题笔记：项目总览、策略中心、市况背书、截图与老仓、拿住牛票、全站重设计、池外 AI
- 原始证据：32 个 Markdown 文件，与 `d621afb` 字节一致

## 校正的长期笔记
- 交付历史、系统全貌、待办与冻结纪律、投资纪律、操盘手审计，共五篇

## 自动检查
- stale scan：<粘贴实际结果；无匹配时写“无活动状态命中”>
- secret scan：<粘贴实际结果；无匹配时写“无命中”>
- vault diff check：通过

## 请裁决
逐项检查事实、数字、冻结边界、历史演变和敏感信息。第一行只写 `裁决:通过` 或 `裁决:打回`；打回时列出必须修正项。
```

```bash
~/.local/bin/hermes-review \
  'screener历史缺口回填' /tmp/screener-backfill-review.md
```

Expected first line: `裁决:通过`. If Hermes returns `裁决:打回`, fix every required item and rerun this task.

- [ ] **Step 3: Push and verify remote equality**

```bash
~/.local/bin/vault-autosync.sh
git -C '~/obsidian-knowledge-base' fetch origin main
git -C '~/obsidian-knowledge-base' rev-list \
  --left-right --count origin/main...main
```

Expected: `0 0`.

- [ ] **Step 4: Seed both cursors only after remote success**

Create `~/.local/state` if absent. Then use `apply_patch` to create both state files with this one-line content:

```text
d621afb336bd4cec26634bcd755e64650278065e
```

Files:

- `~/.local/state/screener-vault-mirror.head`
- `~/.local/state/screener-vault-digest.head`

Expected: both state files contain the full endpoint. The automation plan must ingest `ea3354d`, `77ec779`, and newer commits.
