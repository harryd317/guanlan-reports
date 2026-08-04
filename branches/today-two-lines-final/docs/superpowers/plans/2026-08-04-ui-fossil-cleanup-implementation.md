# 界面化石清理实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 清除今天页和股票池的 S2、红利低波与连阳入口化石，冻结旧 AI 自动入池通道，同时保持数据、S2/S8 引擎和全局零滚动不变。

**Architecture:** 单页前端只对现有只读返回分组和过滤。后端保留旧自动研究实现，在 `reviewer` 和 `eod` 两个公共入口增加默认开启的冻结闸；不做数据库迁移。验收复用生产等效脚本和 20 页面双视口探针。

**Tech Stack:** Python 3、FastAPI、SQLite 只读审计、原生 HTML/CSS/JavaScript、`unittest`、Playwright 浏览器验收

## Global Constraints

- 只动展示层和旧自动入口冻结，不改变生产数据库内容、S2/S8 判定、风控、交易或行情管道。
- S2 遗产继续自然过期，不生成新待处理；旧代码与参数只归档不删除。
- 股票池入口来源仅为人工和 S8 信号。
- 所有页面在 `1440×900` 与 `390×844` 下保持 document 和活动页壳零滚动。
- 最终报告提交且镜像全新克隆验证后才可报告完成。

---

### Task 1: 前端行为契约

**Files:**
- Modify: `test_guanlan_frontend.py`
- Modify: `static/home.html`

**Interfaces:**
- Consumes: `/api/pool.pool[]`、`/api/s8/observations.observations[]`、`/api/brief`
- Produces: `partitionPoolRows(...)`、`formatExpiryCountdown(...)`、`legacyCodes(...)`

- [ ] **Step 1: 写失败测试**

新增模型测试，以字面 fixture 断言：S8、人工、S2 三组顺序；`s2_legacy` 只进遗产组；到期日分别产生“剩 3 天”“今日到期”“待自动退出”“到期日未知”。新增页面测试，断言 S2 `<details>` 无 `open`、今天页没有 `streak-summary` 和 `invest-value`，逻辑卡含“想法冻结 · 未建仓”与“观察学习用途”。

- [ ] **Step 2: 验证 RED**

Run: `~/云慧养/zmu_gitee/screener/.venv/bin/python test_guanlan_frontend.py`

Expected: 新增测试因分组函数、冻结徽标和新入口尚不存在而失败。

- [ ] **Step 3: 最小实现**

在 model script 中实现：

```javascript
function partitionPoolRows(poolPayload,intradayPayload,clusterPayload,s8Payload){
  const rows=buildPoolRows(poolPayload,intradayPayload,clusterPayload,s8Payload);
  return {
    s8:rows.filter(row=>row.source==='S8观察'),
    manual:rows.filter(row=>row.source!=='S8观察'&&row.strategy!=='s2_legacy'),
    legacy:rows.filter(row=>row.strategy==='s2_legacy')
  };
}
```

让 `buildPoolRows` 保留 `strategy` 和 `expireAt`。名单用三个固定分区渲染，遗产区使用默认关闭的 `<details>`。移除今天页红利英雄和连阳摘要；第三张逻辑卡增加 `data-open-streak` 按钮。

- [ ] **Step 4: 验证 GREEN**

Run: `~/云慧养/zmu_gitee/screener/.venv/bin/python test_guanlan_frontend.py`

Expected: 全部通过。

- [ ] **Step 5: 提交**

```bash
git add static/home.html test_guanlan_frontend.py
git commit -m "feat(ui): retire legacy surface cards"
```

### Task 2: 今天页 S2 展示隔离

**Files:**
- Modify: `test_guanlan_frontend.py`
- Modify: `static/home.html`

**Interfaces:**
- Consumes: `brief.cards[]`、`brief.waiting[]` 和 `pool.strategy`
- Produces: `legacyCodes(poolPayload)`，供 `buildTaskRows` 与 `buildWaitingRows` 排除 S2

- [ ] **Step 1: 写失败测试**

以一只 `s2_legacy` 和一只人工票的 fixture 调用 `buildTaskRows`、`buildWaitingRows`，断言结果只含人工票。

- [ ] **Step 2: 验证 RED**

Run: `~/云慧养/zmu_gitee/screener/.venv/bin/python test_guanlan_frontend.py GuanlanModelTests.test_today_excludes_s2_legacy_from_tasks_and_waiting`

Expected: 旧函数返回 S2 行，测试失败。

- [ ] **Step 3: 最小实现**

将池数据传入两个适配函数，在产生界面行前按标准化代码排除 `strategy === 's2_legacy'`。不改 `/api/brief` 或数据库。

- [ ] **Step 4: 验证 GREEN 并提交**

Run: `~/云慧养/zmu_gitee/screener/.venv/bin/python test_guanlan_frontend.py`

```bash
git add static/home.html test_guanlan_frontend.py
git commit -m "fix(ui): hide S2 legacy from today"
```

### Task 3: 冻结 AI 自动入池

**Files:**
- Create: `test_ui_fossil_cleanup.py`
- Modify: `reviewer.py`
- Modify: `eod.py`
- Modify: `test_web.py`

**Interfaces:**
- Produces: `reviewer.AUTO_POOL_CHANNEL_FROZEN: bool`
- Produces: `reviewer.auto_pool_channel_status() -> dict`
- Preserves: `reviewer.auto_pool_decision(pool_id, thesis_result)` 与 `eod._auto_research(trade_date)` 调用签名

- [ ] **Step 1: 写失败测试**

测试将 `storage.pool_list`、`storage.pool_transition` 和 `reviewer.generate_thesis` 替换为一调用即失败的哨兵。默认调用 `_auto_research` 必须返回：

```python
{"processed": 0, "results": [], "frozen": True, "reason": "AI自动入池已冻结（仅归档）"}
```

默认调用 `auto_pool_decision` 必须返回“已冻结（仅归档）”，且不触发任何哨兵。

- [ ] **Step 2: 验证 RED**

Run: `~/云慧养/zmu_gitee/screener/.venv/bin/python test_ui_fossil_cleanup.py`

Expected: 旧入口访问存储或生成研究，测试失败。

- [ ] **Step 3: 最小实现**

在 `reviewer.py` 定义冻结常量和状态函数，在 `auto_pool_decision` 函数第一行执行冻结返回。在 `eod._auto_research` 读取候选前执行同一状态检查。保留后续全部旧实现原文。

- [ ] **Step 4: 修订旧历史测试**

旧 T7.5 测试需要验证归档实现时，临时将 `reviewer.AUTO_POOL_CHANNEL_FROZEN=False`，结束后恢复 `True`；新增默认冻结测试成为生产契约。

- [ ] **Step 5: 验证 GREEN 并提交**

Run: `~/云慧养/zmu_gitee/screener/.venv/bin/python test_ui_fossil_cleanup.py`

Run: `~/云慧养/zmu_gitee/screener/.venv/bin/python test_web.py`

```bash
git add reviewer.py eod.py test_web.py test_ui_fossil_cleanup.py
git commit -m "fix(pool): freeze archived AI auto-entry"
```

### Task 4: 零滚动和专项回归

**Files:**
- Modify: `test_production_equivalent.py` only if the page matrix requires a selector update
- Modify: `scripts/t7_equivalent_env.py` only if the page matrix requires a selector update

**Interfaces:**
- Preserves: `GLOBAL_ZERO_SCROLL_PAGES` 的 10 页面、20 视口组合

- [ ] **Step 1: 运行专项与完整回归**

Run: `~/云慧养/zmu_gitee/screener/.venv/bin/python test_guanlan_frontend.py`

Run: `~/云慧养/zmu_gitee/screener/.venv/bin/python test_production_equivalent.py`

Run: `~/云慧养/zmu_gitee/screener/.venv/bin/python test_rules.py`

Run: `~/云慧养/zmu_gitee/screener/.venv/bin/python test_rules_mid.py`

Run: `~/云慧养/zmu_gitee/screener/.venv/bin/python test_web.py`

Run: `~/云慧养/zmu_gitee/screener/.venv/bin/python test_regime.py`

- [ ] **Step 2: 静态审计并提交**

Run: `git diff --check`

确认新增前端仍只有 `POST /api/account`，并确认没有数据库迁移文件。

### Task 5: 生产等效、部署和镜像

**Files:**
- Create: `docs/验收-界面化石清理-20260804.md`
- Create: `reports/ui-fossil-cleanup-20260804/*`
- Create: `docs/screenshots/ui-fossil-cleanup-20260804/{equivalent,production}/*.png`

**Interfaces:**
- Consumes: 生产 LaunchAgent 克隆、只读数据库副本、镜像 HTTPS 推送脚本
- Produces: 等效/生产双视口证据、生产哈希清单、最终镜像提交哈希

- [ ] **Step 1: 只读审计最后运行时间**

从生产日志中提取最后一条成功自动研究/自动入池记录；用只读 SQLite 查询相关池行 `updated_at` 交叉核对。记录查询命令和结果，不修改库。

- [ ] **Step 2: 生产等效验收**

克隆生产 plist，在独立端口使用同一 `start.sh` 和只读副本启动。采集股票池、今天页和选股逻辑双视口截图；执行 20/20 零滚动探针；核对启动前后 plist、生产库和副本哈希。

- [ ] **Step 3: 合并和部署**

合并功能分支到 `main`，运行最后回归，记录部署前哈希并重启生产。健康检查必须 `ok=true`。生产双视口与 20/20 零滚动必须通过；任一失败回滚上一部署提交。

- [ ] **Step 4: 报告和镜像**

提交最终报告与证据，运行镜像同步，随后全新克隆 `guanlan-reports`。核对报告、截图、JSON 数量与 SHA-256，回报全新克隆 HEAD。
