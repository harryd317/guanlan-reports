# 产业档案 Prompt v2.2 与动物保健Ⅱ返修部署验收

日期：2026-08-08

分支：`ui/today-two-lines-final` → `main`

结果：通过并已部署。动物保健Ⅱ改后稿仍为“改后待再审”，没有发布，没有对 S8 档案门槛生效。

## 1. 交付结论

- 四尺审校指出的 5 处事实问题已修正：温度计读数缺失、“20 日涨幅/单日涨幅”矛盾、生物股份自指竞争对手、重复公司名笔误、过时搜索快照被误称为“最新”。
- Prompt 已从 `industry-archive-v2.1-web` 升级为 `industry-archive-v2.2-web`，强制温度计快照置顶、三类周期指标显式缺失、申万/东财口径分开、来源分级和过时快照警示。
- 纠正后草稿通过七节、核心股不超过 5 只、来源与日期、数字边界和未发布警示校验；状态仍是待再审。
- 上轮等效环境的旧源 API 错误根因已固化修复：LaunchAgent 克隆现在将 `INDUSTRY_THERMOMETER_DB_PATH` 指向独立温度计副本，不再误指 S3 副本。等效日志中无 `no such table: industry_thermometer_snapshots`。

## 2. 代码与提交

| 用途 | 提交 |
|---|---|
| 设计与 TDD 计划 | `f3f1d62` |
| Prompt v2.2、骨架与验证器 | `6059522` |
| 四尺审校、改后稿与旧源回归测试 | `9d3fc30` |
| 生产等效证据 | `6f4ff0f` |
| 合并 `main` | `56b2dce5a44db124de680fbd91546c7059e7fd2d` |
| 部署前回滚锚点 | `1fc05d51da952b4ccbb3e57e22d125d22102aa2b` |

主要改动文件：`reviewer.py`、`s8_research/archives.py`、`s8_research/hook.py`、`service.py`、`scripts/t7_equivalent_env.py` 及对应测试。

## 3. 动物保健Ⅱ改后稿

温度计真实快照：

- 数据日期：2026-08-04；申万二级代码：`801018.SI`；档位：温；亮灯数：2。
- 量比灯与成交占比灯亮；量比 `1.0326508831`，全市场排名 38，不是前 5。
- 猪价、养殖利润和批签发在当前骨架中均显式记为缺失，不用搜索缓存冒充。
- 东财 `BK1254` 2025-07-21 数据只标为“参考行情快照”，与申万口径分开，并前置时效警示。
- 核心股 5 只；证据分为“研究报告/行情终端”和“自媒体”，自媒体只用于资金异动旁证。

证据：

- `reports/evidence/industry-archive-v2.2-20260808/review/动物保健Ⅱ-四尺审校意见-20260807.md`
- `reports/evidence/industry-archive-v2.2-20260808/review/动物保健Ⅱ-改后待再审初稿.md`
- `reports/evidence/industry-archive-v2.2-20260808/validation.json`
- `reports/evidence/industry-archive-v2.2-20260808/thermometer-snapshot-801018.SI.json`

## 4. TDD 与全量回归

| 测试 | 结果 |
|---|---:|
| 先写失败测试 | RED：1 失败 + 7 错误，证明旧骨架、旧验证器不满足 v2.2 |
| 聚焦修复 | 5/5 |
| 前端/档案/等效/S8 核心 unittest | 168/168 |
| `test_web.py` | 838/838 |
| `test_rules.py` | 8/8 |
| `test_rules_mid.py` | 20/20 |
| `test_regime.py` | 9/9 |
| 镜像同步单测 | 28/28 |
| `compileall` / `git diff --check` | 通过 |
| 新增 Kimi/GitHub 敏感模式扫描 | 0 命中 |

全量回归中只有已知 `ResourceWarning` 和测试夹具的网络降级日志，没有断言失败。

## 5. 生产等效环境

- 生产 plist 仅读克隆，原文件字节未改；SHA-256 始终为 `4546524c15d5391442c3b74acaf567147f2fe12af6207c23fc3ca7bee9de01cc`。
- 克隆使用相同 `start.sh`、相同工作目录语义和静态文件服务方式，独立端口 `18810`。
- `screener`、`market`、温度计、S8 副本全部为 `0444`只读；生产库和副本启动前后 11 项 SHA-256 全部一致。
- 健康检查 `ok=true`、`match=true`；温度计和档案队列 API 成功；首页响应 `Cache-Control: no-store`。
- 全路由零页面级滚动：12 路由 × 3 视口 = 36/36；双视口关键截图 10/10；控制台 0 错误。

证据目录：`reports/evidence/industry-archive-v2.2-20260808/equivalent/`。

## 6. 生产部署与数据边界

部署命令：`./restart.sh`。部署后：

- `/api/health` 返回 `ok=true`、`match=true`，启动与磁盘指纹一致。
- Prompt 版本为 `industry-archive-v2.2-web`。
- 动物保健Ⅱ队列状态是 `skeleton_ready`，前端显示“研究中”；`prompt_version/channel/model` 均为空，证明改后稿未写入生产队列、未发布。
- 全路由零滚动 36/36；生产双视口截图 10/10；控制台 0 错误。
- `market.db`、`industry_thermometer_sidecar.db` 和生产 plist 在部署前/部署后门禁时字节 SHA-256 一致。随后全量测试关闭 SQLite 连接时将 `market.db-wal` 的既有页检查点回主文件，使主文件哈希变为 `fa4ebb4b…`。已用主键逐行 `SQLite IS` 比较部署前逻辑快照与当前库：6 张表的行数、每个字段和每一行全部一致，`all_tables_identical=true`；这是物理 checkpoint，没有业务数据写入。
- `screener.db` 原始文件哈希从 `9fc0abb1…` 变为 `8e21c748…`，但部署前快照与部署后生产库的表结构、行数和逐表内容哈希全部一致；变化仅限 SQLite 物理页。
- `S8_research.db` 原始文件哈希从 `9b7d9a10…` 变为 `08600e58…`；唯一逻辑变化是已批准可写表 `archive_queue` 增加 7 个生成/审校/审计字段。表行数部署前后都为 21，无业务行增删；其他表无变化。

生产证据目录：`reports/evidence/industry-archive-v2.2-20260808/production/`。温度计 API 如实标注数据日 2026-08-04、最新行情日 2026-08-07 和 `is_stale=true`；这是部署前后一致的数据新鲜度状态，不是本次回归。

## 7. 回滚

部署后健康、API、页面、滚动断言和数据边界全部通过，未触发回滚。如后续需要回滚，对合并提交 `56b2dce5` 执行可追溯 `git revert -m 1`，然后运行 `./restart.sh`；`archive_queue` 的新列为添加式、向后兼容，旧代码会忽略它们，不需要破坏性改库。

## 8. 镜像交付

本报告与全部脱敏证据提交后，使用 HTTPS 同步到 `guanlan-reports`，并以全新克隆检查本报告、改后稿、四尺意见、等效/生产证据和源提交元数。最终镜像提交哈希在交付回报中给出；没有该哈希不视为完成。
