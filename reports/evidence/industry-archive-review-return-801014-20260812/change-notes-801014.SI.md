# 饲料（801014.SI）四尺审校返修说明

## 返修范围

只执行 Kimi 四尺意见列出的三项小修：

1. ③节把“2025年H1猪价承压（约14.6元/kg），H2有望反弹至18-20元/kg”改为“2025H1猪价约14.6元/kg（历史参考）”。删除的是③节采用的过期预测；⑦节仍按原稿保存来源页面自身曾包含的信息范围，不把该预测作为档案判断。
2. ②节把“越南/印尼市占率15%/8%”补全为“海大集团：越南/印尼市占率15%/8%”。
3. ①节把 `volume_ratio=0.9852` 改为快照全精度 `volume_ratio=0.9852181371622244`。

`git diff --no-index` 仅显示上述三行正文变化。

## 不变证明

- ①节剔除 `volume_ratio` 数值后，返修前后 SHA-256 均为 `023f7b9a6f75d1db724fe8e49093cecdfbf4d8c399bbc5cb33439a445edad7af`。
- 生产 `skeleton_json` 返修前后 SHA-256 均为 `b59c07708b3961d8b8c428625329947dd1fc24674f2e119d22ad3aafd09f1bc6`。
- 温度计生产 sidecar 返修前后文件 SHA-256 均为 `e4ea0f2354180c5f815e1b996be9dba935f21bd95d547baf817c4e9cb27f794e`。
- 原稿 SHA-256：`9759ce2c32cc382c686d8938e68aa62f911a3faec4465b3d70d45d9c3f3d4506`。
- 返修稿 SHA-256：`0f51e7bdbba36a702b490e81d5297a27b38c793ec21fa87371abc50bdf02b2ef`。

## 校验

`s8_research.archives.validate_archive_v2` 使用 `industry-archive-v2.2-web` 重跑：`ok=true`、缺节 `0`、错误 `0`、核心股 `5`。
