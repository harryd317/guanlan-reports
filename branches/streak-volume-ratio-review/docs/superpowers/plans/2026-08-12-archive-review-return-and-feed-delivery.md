# Archive Review Return and Feed Delivery Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 回投通用设备四尺审校意见，完成四项受控返修并把生产队列停在待复审，同时按同一规范投递饲料审校材料。

**Architecture:** 公开镜像增加受脱敏策略保护的顶层 `review/` 通道；审校原文先独立提交并取得镜像哈希，再通过既有队列接口记录审校和退回。返修稿与饲料材料全部从生产 `archive_queue` 只读基线派生，返修完成后仅通过既有 `archive_queue` 状态函数写回该表。

**Tech Stack:** Python、SQLite、Git、Markdown、JSON、`s8_research.archives.validate_archive_v2`、`scripts/report_mirror_sync.py`。

## Global Constraints

- 通用设备骨架和①节一字不动。
- 统计局全年数字只采用国家统计局 2026-01-27 官方发布。
- 饲料生产数据库只读，不触发重新生成。
- 通用设备只写 `S8_research.db.archive_queue`；不发布档案。
- 密钥、私有路径和原始响应正文不得进入 Git 或公开镜像。
- 完成定义包含最终报告提交、镜像同步和全新克隆复核后的镜像提交哈希。

---

### Task 1: 永久化公开审校回投通道

**Files:**
- Modify: `scripts/report_mirror_sync.py`
- Modify: `test_report_mirror_sync.py`
- Create: `review/通用设备-四尺审校意见-20260809.md`

- [ ] **Step 1:** 增加失败测试，证明 `review/*.md` 会进入镜像且其他根目录仍被拒绝。
- [ ] **Step 2:** 运行 `test_report_mirror_sync.py`，确认新测试在旧实现上失败。
- [ ] **Step 3:** 将 `review/` 纳入允许根目录，并限制为 Markdown/JSON 文本交付物。
- [ ] **Step 4:** 逐字写入用户提供的四尺审校意见，运行镜像脱敏预检。
- [ ] **Step 5:** 提交并推送审校原文，使用全新克隆验证镜像路径与字节哈希。

### Task 2: 回投审校并退回返修

**Files:**
- Create: `reports/evidence/industry-archive-review-return-20260812/queue-before.json`
- Create: `reports/evidence/industry-archive-review-return-20260812/queue-review-return.json`

- [ ] **Step 1:** 只读记录生产库哈希和 `801072.SI` 队列基线。
- [ ] **Step 2:** 使用已验证镜像哈希调用既有 `/review` 接口，写入四尺意见。
- [ ] **Step 3:** 使用既有 `/finalize` 的 `return` 动作退回，确认状态为 `draft_ready`、未发布且审计日志新增两步。

### Task 3: 四项受控返修

**Files:**
- Create: `reports/evidence/industry-archive-review-return-20260812/通用设备-801072.SI-kimi-v2.2-返修稿.md`
- Create: `reports/evidence/industry-archive-review-return-20260812/validation-801072.SI.json`
- Create: `reports/evidence/industry-archive-review-return-20260812/change-notes-801072.SI.md`

- [ ] **Step 1:** 冻结原稿①节与骨架 SHA-256。
- [ ] **Step 2:** 删除中研普华冲突数字，②节替换为统计局 2025 全年数据和官方链接。
- [ ] **Step 3:** 压缩③节陈旧填充句并明示时效；修改⑥节骨架日期措辞。
- [ ] **Step 4:** 重跑 `validate_archive_v2(..., prompt_version='industry-archive-v2.2-web')`，要求 `ok=true`。
- [ ] **Step 5:** 对比返修前后①节和生产骨架，要求字节一致。

### Task 4: 饲料审校材料只读投递

**Files:**
- Create: `reports/evidence/industry-archive-review-801014-20260812/饲料-801014.SI-kimi-v2.2-初稿.md`
- Create: `reports/evidence/industry-archive-review-801014-20260812/thermometer-snapshot-801014.SI.json`
- Create: `reports/evidence/industry-archive-review-801014-20260812/generation-evidence-801014.SI.json`
- Create: `reports/evidence/industry-archive-review-801014-20260812/submission-manifest.json`

- [ ] **Step 1:** 从生产队列只读提取 `801014.SI` 初稿、骨架快照和生成元数据。
- [ ] **Step 2:** 重跑 v2.2 校验并如实记录搜索门禁、耗时和未持久化 token 字段。
- [ ] **Step 3:** 记录提取前后生产库哈希，要求一致。
- [ ] **Step 4:** 生成带文件 SHA-256 和源提交的 submission manifest。

### Task 5: 返修稿入队、总验收和镜像复核

**Files:**
- Create: `docs/验收-通用设备返修与饲料审校投递-20260812.md`
- Create: `reports/evidence/industry-archive-review-return-20260812/queue-after.json`

- [ ] **Step 1:** 使用既有队列函数写入返修稿，再以最终镜像哈希推进到 `awaiting_review`。
- [ ] **Step 2:** 确认通用设备未发布、饲料队列与生产稿未变化。
- [ ] **Step 3:** 运行项目四套回归、镜像同步测试、敏感信息扫描和 `git diff --check`。
- [ ] **Step 4:** 提交最终验收报告，合并主线并同步镜像。
- [ ] **Step 5:** 全新克隆镜像，核对审校原文、两套材料、验证结果和镜像提交哈希。
