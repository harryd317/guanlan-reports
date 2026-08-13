# 通用设备复尺与饲料返修 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 精确回投两份 Kimi 审校原文，使通用设备进入待段睿终审，并完成饲料三项最小返修、校验、队列回投和镜像验证。

**Architecture:** 审校原文先作为 `review/` 下的不可改写证据提交并精确同步镜像，取得可验证提交号后再调用既有审校状态机。饲料返修从生产队列当前草稿派生，只改三处正文；生产骨架与温度计 sidecar 不写，所有生产写入限于 `archive_queue` 的指定行业行。

**Tech Stack:** Git、SQLite、Python `s8_research.archives`、HTTPS 报告镜像、SHA-256。

## Global Constraints

- 通用设备不得自行发布，只能停在 `awaiting_review` 且 `review_json` 已存在的待段睿终审状态。
- 饲料只执行三项审校返修，不触发 AI 重新生成。
- 饲料骨架和温度计除①节量比文本外不得改变；生产 sidecar 全程只读。
- 最终报告必须提交，并以远端全新克隆验证后的镜像提交哈希作为完成条件。

---

### Task 1: 精确回投两份审校原文

**Files:**
- Create: `review/通用设备-复尺意见-20260812.md`
- Create: `review/饲料-四尺审校意见-20260812.md`

**Interfaces:**
- Consumes: 用户消息分隔线内的原文。
- Produces: review API 可校验的镜像文件及提交哈希。

- [ ] **Step 1:** 用 `apply_patch` 写入两份原文，不包含分隔线，不改写正文。
- [ ] **Step 2:** 逐字复核文件并记录 SHA-256。
- [ ] **Step 3:** 提交源 Git，使用精确路径镜像同步。
- [ ] **Step 4:** 全新克隆镜像，核对 HEAD、路径和 SHA-256。

### Task 2: 推进通用设备至待段睿终审

**Files:**
- Create: `reports/evidence/industry-archive-second-review-20260812/queue-801072.SI.json`

**Interfaces:**
- Consumes: Task 1 的镜像提交哈希、通用设备复尺意见。
- Produces: `archive_queue.801072.SI` 的四尺 `review_json`、镜像哈希和 `external_review` 审计记录。

- [ ] **Step 1:** 读取并记录生产 S8 库写前哈希和队列行。
- [ ] **Step 2:** 把原文按既有 `facts/specification/logic/conclusion` 四字段映射，调用 `record_external_review`。
- [ ] **Step 3:** 读取写后行，断言 `status=awaiting_review`、`review_json` 存在、镜像哈希匹配、`archive_id` 为空。
- [ ] **Step 4:** 记录写后数据库哈希；不调用 `publish_reviewed_archive`。

### Task 3: 饲料三项最小返修与证明

**Files:**
- Create: `reports/evidence/industry-archive-review-return-801014-20260812/饲料-801014.SI-kimi-v2.2-返修稿.md`
- Create: `reports/evidence/industry-archive-review-return-801014-20260812/change-notes-801014.SI.md`
- Create: `reports/evidence/industry-archive-review-return-801014-20260812/validation-801014.SI.json`

**Interfaces:**
- Consumes: 生产队列当前饲料草稿、骨架、温度计快照和用户三项返修清单。
- Produces: 仅三处语义改动的返修稿和可机读证明。

- [ ] **Step 1:** 从当前已提交初稿复制返修稿，确认与生产 `ai_draft` 仅差末尾换行。
- [ ] **Step 2:** 删除 ③ 节过期预测半句，给海外市占率补“海大集团”主语，把①节量比改为 `0.9852181371622244`。
- [ ] **Step 3:** 运行精确差异检查，断言除三处外无正文变化。
- [ ] **Step 4:** 重跑 `validate_archive_v2`，要求 `ok=true`、缺节 0、错误 0、核心股不超过 5。
- [ ] **Step 5:** 计算①节剔除量比字段后的前后 SHA-256，并计算生产 `skeleton_json` 与 sidecar 前后 SHA-256。

### Task 4: 饲料审校退回、返修回投与最终验收

**Files:**
- Create: `reports/evidence/industry-archive-review-return-801014-20260812/queue-flow-801014.SI.json`
- Create: `docs/验收-通用设备复尺与饲料返修-20260812.md`

**Interfaces:**
- Consumes: Task 1 饲料审校镜像哈希、Task 3 返修稿及校验结果。
- Produces: 饲料返修稿镜像哈希、待复审生产队列状态和最终验收报告。

- [ ] **Step 1:** 用审校镜像哈希调用饲料 `record_external_review`，随后按“改后再审”调用 `return_archive_for_revision`。
- [ ] **Step 2:** 通过 `draft_from_skeleton` 写入返修稿；除 `archive_queue.801014.SI` 外不写生产。
- [ ] **Step 3:** 提交返修证据与验收报告，精确同步镜像并全新克隆验证。
- [ ] **Step 4:** 用返修镜像哈希调用 `mark_draft_mirrored_for_review`，断言饲料停在 `awaiting_review`、未发布。
- [ ] **Step 5:** 运行档案与镜像测试、项目全量回归，记录结果和最终数据库哈希。
- [ ] **Step 6:** 提交最终队列证据和报告补充，执行最终精确镜像同步与全新克隆复核。
