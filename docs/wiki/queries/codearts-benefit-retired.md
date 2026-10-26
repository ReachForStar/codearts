---
title: CodeArts 免费额度体系下线与计费口径迁移
type: query
tags: [CodeArts, benefit, 免费额度, 下线, 计费, 模型列表, 实测]
created: 2026-10-26
updated: 2026-10-26
sources: []
status: active
---

# CodeArts 免费额度体系下线与计费口径迁移

## 问题

用户报告「码道里的模型收费状态变了，现在没有免费模型和每日免费额度，
magpie 获取不到这些信息」。需要核对：

1. 免费（benefit）模型集与每日免费额度是否仍在。
2. 模型名称与收费状态能否继续由插件自动获取。

## 根因

码道于 2026-10-26 撤下免费额度体系，同时下线了两个供插件识别免费模型的数据源：

- `opengw.developer.huaweicloud.com/api/v1/gateway/config`（免费模型集）——
  签名请求返回 `{"error_code":"4000","error_msg":"invalid channel, group not found","result":null}`。
- `opengw.developer.huaweicloud.com/api/v1/user/tokens/balance`（每日免费额度）——
  同一错误 `4000 invalid channel, group not found`。

旧实现把 `gateway/config` 返回的每个模型都标成 benefit，并据此发送参与签名的
`maas_type: benefit` 头。该源一旦返回空 `result`，免费集就变成空，而静态兜底
`BENEFIT_MODEL_FALLBACK` 里的 4 个 id 已全部失效，于是「标注与计费口径」同时失真。

## 实测（2026-10-26，账号 `codearts`，请求头参与 SDK-HMAC-SHA256 签名）

### 端点可用性

| 端点 | 结果 |
| --- | --- |
| `opengw/api/v1/gateway/config` | `4000 invalid channel, group not found`，`result: null` |
| `opengw/api/v1/user/tokens/balance` | `4000 invalid channel, group not found` |
| `snap-access/v1/model/builtin` | 200，`count: 10`（8 个文本模型 + 2 个视觉模型） |
| `snap-access/snap-manager/v1/statistics/plugin` | 200，仅积分口径，无免费池字段 |

### 旧免费模型逐个复核（HTTP 均 200，错误在 SSE 首帧）

| id | 不带 benefit 头 | 带 benefit 头 |
| --- | --- | --- |
| `glm-5.3-flash` | `InferHub.002002009.404` 未注册 | `InferHub.4004.200 benefit not found` |
| `deepseek-v4.1-flash` | **正常返回**（已注册，积分计费） | `InferHub.4004.200 benefit not found` |
| `deepseek-v4-flash-0731` | `InferHub.002002009.404` 未注册 | `InferHub.4004.200 benefit not found` |
| `deepseek-v4-pro-0813` | `InferHub.002002009.404` 未注册 | `InferHub.4004.200 benefit not found` |

结论：4 个旧 benefit id 里 3 个已注销，唯一仍可调用的 `deepseek-v4.1-flash`
不带 benefit 头即可用，带该头反而被拒。免费池与 benefit 头都已不存在。

### 当前模型清单（`v1/model/builtin`，与 IDE「内置」列表逐项一致）

| id | `model_name` | 倍率（默认 / 错峰） |
| --- | --- | --- |
| `GLM-5.2` | GLM-5.2 | 0.70x |
| `glm-5.2-sft-harmony` | GLM-5.2-ArkTS-SPARK | 0.70x |
| `openpangu-2.0-pro` | OpenPangu-2.0-Pro | 0.70x |
| `openpangu-2.0-flash` | OpenPangu-2.0-Flash | 0.32x |
| `glm-5.3` | GLM-5.3 | 0.79x |
| `deepseek-v4.1-flash` | DeepSeek-V4.1-Flash | 0.18x / 0.09x |
| `kimi-k3` | Kimi-K3 | 2.38x |
| `deepseek-v4.1-flash-1017` | DeepSeek-V4.1-Flash-1017 | 0.18x / 0.12x |

`Qwen3-VL-235B` 与 `kimi-k2.6-vl` 是视觉模型（`supports_images: true`、
`display_enabled: false`），继续被 `-vl` 过滤。

## 解法

- 删除 benefit 体系全部代码：`BENEFIT_MODEL_FALLBACK`、`benefitModelIds`、
  `isBenefitModel`、`rememberBenefitIds`、`maas_type: benefit` 头、
  `benefitWindow`、`BENEFIT_BALANCE_URL`、`OPENGW_GATEWAY_CONFIG_URL`、
  模型上的 `free: true` 标注。
- `fetchRemoteModels` 只读 `v1/model/builtin`；模型 id、显示名全部取接口原值，
  插件不硬编码任何模型名。
- `parseModelInfo` 不再对显示名去日期后缀：`deepseek-v4.1-flash` 与
  `deepseek-v4.1-flash-1017` 是后端并存的两个模型，去掉后缀会让列表出现两行同名。
  日期后缀只用于 `CONTEXT_WINDOWS` 查表。
- 出厂 `CATALOG` 换成当前 8 个模型（含两个 DeepSeek 变体），移除已注销的
  `glm-5.3-flash`、`deepseek-v4-flash-0731`、`deepseek-v4-pro-0813` 与不再出现在
  列表里的 `GLM-5.1`。
- `usageFromStatistics` 只映射积分口径，不再读免费池。

## 涉及模块

- [CodeArts provider 插件](../entities/codearts-plugin.md)：`fetchRemoteModels`、
  `parseModelInfo`、`CATALOG`、`CONTEXT_WINDOWS`、`usageFromStatistics`、`modelOf`。
- [用量窗口只让总积分参与路由](../decisions/codearts-usage-window-routing.md)：
  「免费额度」窗口已随体系下线移除，`aside` 口径不变。
- [免费模型 id 与计费池归属](codearts-benefit-model-ids.md)：已被本页取代。

## 复发预防

- 判断「是否还有免费模型」只看两个信号：`gateway/config` 是否返回非空 `models`，
  以及免费 id 带 `maas_type: benefit` 是否仍返回 200。任一失效即视为体系下线。
- 模型清单与显示名一律以 `v1/model/builtin` 的 `model_name` 为准，不在代码里写死
  名称；新增或改名时插件无需改动。
- 收费状态（倍率）在 `credit[].ratio_display`，`time_slot_id` 区分默认与错峰时段；
  magpie 模型字段没有倍率位，插件不下发倍率，也不推断免费。
