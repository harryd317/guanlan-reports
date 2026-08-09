# 产业档案 Kimi 生成通道设计

日期：2026-08-05

分支：`ui/today-two-lines-final`

## 1. 目标与边界

产业档案生成通道新增 `kimi`，并把它设为默认值。`claude_code` 与 `api` 保留为操作者可选通道。切换只由 `ARCHIVE_AI_ENGINE` 配置决定；运行时禁止自动改用其他通道。

本批只改产业档案生成、元数据和待终审展示。它不接交易、不改 S8 信号门槛、不改风控、不写生产数据库，也不部署生产。

## 2. 官方协议基线

Moonshot 当前[模型列表](https://platform.kimi.com/docs/models)把 `kimi-k3` 列为最新旗舰模型，并说明旧 K2 型号的下线安排。因此，`kimi_model` 默认值为 `kimi-k3`，但操作者可以覆盖。

Moonshot 的[联网搜索说明](https://platform.kimi.com/docs/guide/use-web-search)规定：

- Chat Completions 地址使用 `https://api.moonshot.cn/v1/chat/completions`；
- 每轮请求在 `tools` 中声明 `type=builtin_function`、`function.name=$web_search`；
- 模型返回 `tool_calls` 后，调用方原样保留完整 assistant 消息；
- `$web_search` 的 arguments 原样作为对应 tool 消息回传；
- 循环继续，直到 `finish_reason=stop`。

实现采用这套协议。初稿必须至少产生一次 `$web_search` 调用；模型直接返回正文而未搜索时，系统拒绝该初稿。

## 3. 配置与密钥

Kimi 配置只来自服务器进程环境或仓库外文件：

| 配置 | 环境变量 | 外部文件键 | 默认值 |
| --- | --- | --- | --- |
| 密钥 | `KIMI_API_KEY` | `KIMI_API_KEY` | 无，缺失即失败 |
| 模型 | `KIMI_MODEL` | `kimi_model` 或 `KIMI_MODEL` | `kimi-k3` |
| 基址 | `KIMI_BASE_URL` | `kimi_base_url` 或 `KIMI_BASE_URL` | `https://api.moonshot.cn/v1` |
| 文件路径 | `KIMI_CONFIG_PATH` | 不适用 | `~/.config/guanlan/kimi.env` |

进程环境优先于外部文件。程序不创建配置文件，不把配置写入 SQLite，不读取 Git 跟踪文件中的密钥。配置文件沿用 `KEY=VALUE` 格式；操作者负责把权限设为 `0600`。

错误响应只报告固定的人话说明、HTTP 状态或异常类型，不拼接响应正文、请求头、URL 查询串或异常原文。日志不得记录密钥、Authorization 头或请求体。

## 4. Kimi 适配器

`reviewer.py` 增加产业档案专用 Kimi 适配器：

1. 读取并校验 Kimi 配置；
2. 组装现有 `industry-archive-v2.1-web` system prompt、同一数据骨架和既有研究条目；
3. 调用 OpenAI 兼容 Chat Completions；
4. 每轮都带 `$web_search` 声明；
5. 原样回传完整 assistant 消息和搜索 arguments；
6. 限制循环轮数，拒绝未知工具、畸形 JSON、空正文和未实际搜索的正文；
7. 返回 `content`、`channel=kimi`、实际 `model` 和 `prompt_version`。

Kimi 适配器不注册到通用 `_ask_engine` 主备链。产业档案生成按 `ARCHIVE_AI_ENGINE` 精确选择 `kimi`、`claude_code` 或 `api`；任何通道失败都直接结束本次生成。

## 5. 状态与元数据

`archive_queue` 新增两个可空字段：

- `generation_channel`：本篇初稿实际使用的通道；
- `generation_model`：本篇初稿实际使用的模型名。

成功完成七节校验时，系统把 `channel`、`model` 和既有 `prompt_version` 与初稿一并写入。开始重跑时清空旧生成元数据，避免展示上一版信息。

Kimi 密钥缺失、HTTP 失败、搜索未执行、循环超限或七节校验失败时，服务端沿用现有单飞锁和失败回收：状态从 `generating` 恢复为 `skeleton_ready`，保存经过截断且不含密钥片段的错误提示。系统不尝试 `claude_code` 或 `api`。

## 6. 前端透明展示

待终审页保留固定警示：

> AI 生成初稿 · 未经审校/终审不作为核心股依据

警示旁显示：

`生成通道：{channel} · {model} · Prompt {prompt_version}`

该信息只供终审操作人核对，不改变发布门槛。生成仍不等于发布。

## 7. 测试与验收

TDD 用例覆盖：

- 默认通道为 `kimi`，默认模型为官方当前 `kimi-k3`；
- 环境变量和仓库外配置的优先级；
- HTTP mock 收到 `$web_search` builtin tool；
- assistant/tool 循环保留完整消息并至少执行一次搜索；
- 密钥缺失和 HTTP 失败不回显任何密钥片段；
- 失败后队列恢复 `skeleton_ready`；
- 成功初稿保存 `channel/model/prompt_version`；
- 待终审页显示生成元数据；
- 镜像敏感信息扫描阻断 Kimi `sk-` 密钥。

若本机能从允许位置读到 `KIMI_API_KEY`，生产等效环境用同一行业做一次真实生成，验证七节、核心股数量和带日期来源，并把脱敏后的草稿与验证 JSON 写入 `reports/evidence/industry-archive-kimi-20260805/`。若密钥缺失，证据明确记录“未执行：缺 KIMI_API_KEY”及配置路径，不制造假成功。

既有聚焦套件、`test_web.py`、三组规则测试和 36 条零滚动矩阵必须全部复跑。最终报告提交后，同步公开镜像并从全新克隆验证镜像提交。

## 8. 回滚

运行回滚只需把 `ARCHIVE_AI_ENGINE` 显式改为 `claude_code` 或 `api`；代码和新增可空列可以保留。代码回滚可撤销本批提交；旧代码会忽略新增列。由于本批不部署，生产无需回滚动作。
