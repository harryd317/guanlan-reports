# 产业档案中心与全局零滚动实施计划

> 使用用户已确认的产业档案中心原型，全程按 TDD 执行。

**目标：** 将真实只读档案、温度计和 S8 队列组装为档案中心与统一行业页，同时使所有主/子页面在两个冻结视口下页面级零滚动。

**架构：** 保留单文件 hash SPA 和既有 GET API。后端只补充档案元数据、S8 队列展示状态与离线入库门禁；前端合并四个真实只读返回。长内容只在固定页壳的指定内容区滚动。

---

## 任务 1：冻结设计与原型

**文件：**
- 新增 `docs/原型-产业档案中心-20260804.html`
- 新增 `docs/superpowers/specs/2026-08-04-industry-archive-center-design.md`
- 修改 `docs/superpowers/specs/2026-08-01-guanlan-frontend-rewrite-design.md`

1. 校验附件和入库原型 SHA-256 完全一致。
2. 将全局零滚动、子页承载更多内容、时钟复用和数据日展示边界写入唯一前端设计规范。

## 任务 2：档案元数据与 v2 状态机—RED

**文件：**
- 新增 `test_industry_archive_center.py`
- 修改 `test_industry_archives.py`
- 修改 `test_s8_research.py`
- 修改 `test_reviewer.py` 或新增等价提示词契约测试

1. 先写 `summary` 解析与无摘要诚实空态测试。
2. 写队列原始状态→四态显示、进度和稳定排队位置测试。
3. 写 v2 七节、核心股不超过 5、带日期证据和样板深度测试。
4. 写“无镜像哈希不得待终审”、“无人工入库文件不得已发布”测试。
5. 单独运行新测试，确认因缺实现而失败。

## 任务 3：档案元数据与 v2 状态机—GREEN

**文件：**
- 修改 `industry_archives.py`
- 修改 `s8_research/db.py`
- 修改 `s8_research/archives.py`
- 修改 `s8_research/read_model.py`
- 修改 `reviewer.py`
- 修改 `docs/产业档案/*.md`

1. 解析可选 `summary`，返回目录和详情。
2. 为 `archive_queue` 增加向后兼容的镜像哈希/时间列，不新建第四张 S8 写表。
3. 实现统一 v2 校验器、AI 提示词、镜像后待终审迁移和人工发布迁移。
4. 丰富队列 GET 返回的 `display_status`、`progress_step`、`queue_position`、`queue_total`，不增加写 API。
5. 单独运行新测试直到通过。

## 任务 4：前端契约—RED

**文件：**
- 修改 `test_guanlan_frontend.py`
- 修改 `test_industry_archives.py`
- 修改 `test_production_equivalent.py`

1. 先写档案实心/虚线卡、四段进度、两态行业页和七节大纲测试。
2. 写行业统一路由、四类入口和来源返回测试。
3. 写产业级文案清理、时钟固定宽度、状态复用和数据日页脚消失测试。
4. 写连阳摘要/子页、个股详情返回和 S8 更多子页测试。
5. 写主页只保留账户 POST、档案与队列 UI 无 POST 测试。
6. 单独运行并确认失败。

## 任务 5：单页实现—GREEN

**文件：**
- 修改 `static/home.html`

1. 增加 `view-streak-list`、`view-s8-focus-list`和统一 `view-industry`。
2. 将档案子页签替换为封面卡库，保留紧凑温度计入口，所有行业点击统一进行业页。
3. 实现已发布正文和研究中大纲/真实原始名单。
4. 拆分 S8 产业级小卡与个股/复核子页，清理黑话和产业级 90 天文案。
5. 实现连阳摘要、完整名单子页及非股票池标的只读详情降级。
6. 恢复中国标准时间走秒，仅映射 `/api/intraday` 市场状态。
7. 使所有页壳固定高度，只对设计指定的内容区开放内部滚动。
8. 运行前端和档案契约测试直到通过。

## 任务 6：全局零滚动自动化

**文件：**
- 修改 `scripts/t7_equivalent_env.py`
- 修改 `test_production_equivalent.py`

1. 新增全路由探针页清单和双视口矩阵校验。
2. 对 document 与活动页壳同时要求 `scrollHeight === clientHeight`、`scrollWidth === clientWidth`。
3. 使用有效与故意溢出样本确认校验器会先红后绿。

## 任务 7：回归与代码评审

1. 运行新增小测试。
2. 运行 `test_industry_archives.py` 和 `test_s8_research.py`。
3. 运行 `test_guanlan_frontend.py` 和 `test_production_equivalent.py`。
4. 运行项目规定四组完整回归。
5. 执行 `git diff --check`、演示数据扫描、读写路径扫描和代码评审。

## 任务 8：生产等效验收

1. 记录生产 plist 与全部生产库启动前哈希。
2. 一致性快照生成只读副本，记录副本哈希。
3. 使用生产 plist 克隆和本分支 `start.sh` 在独立端口启动。
4. 对全部主/子页两视口执行零滚动探针；保存今天、档案库、行业已发布/研究中八张截图。
5. 核对控制台、健康接口、plist、生产库和副本库启动后哈希。

## 任务 9：合并、部署与镜像

1. 提交源分支最终报告与证据。
2. 合并 `main`，运行部署前最后回归。
3. 通过 `./restart.sh` 重启生产。
4. 重做全路由零滚动探针、八张生产截图、健康检查和数据库哈希核对；任一失败按上一部署版本回滚。
5. 提交生产验收报告和截图证据。
6. 同步全部 `docs/` 和 `reports/` 到镜像，用全新克隆核对关键产物与 SHA-256，回报镜像提交哈希。
