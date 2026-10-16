---
title: CodeArts 免费模型 id 与计费池归属
type: query
tags: [CodeArts, benefit, 免费额度, 模型 id, 计费, 排查]
created: 2026-10-16
updated: 2026-10-16
sources: []
status: active
---

# CodeArts 免费模型 id 与计费池归属

## 问题

模型列表里免费模型显示不全，且部分条目调用不通。两个疑问需要实测回答：

1. `opengw gateway/config` 公布的 4 个模型是否都是 benefit（免费额度）模型。
2. 带日期后缀的 id（`deepseek-v4-flash-0731`）与去后缀的裸 id
   （`deepseek-v4-flash`）是同一个模型还是两个。

## 根因

`normalizeModelId`（去掉末尾 `-NNNN`）原本同时用于**展示**与**下发 id**。
`gateway/config` 的 4 个模型里，`glm-5.3-flash`、`deepseek-v4.1-flash` 无后缀，
另外两个去后缀后落到 `deepseek-v4-flash`、`deepseek-v4-pro`。于是：

- 只有 2 个模型被判为 benefit 并标 `free: true`；
- 另 2 个免费模型以裸 id 下发，既没有 benefit 标注，也调不通（见下表）。

## 实测矩阵（2026-10-16，账号 `codearts`，请求头参与 SDK-HMAC-SHA256 签名）

请求直接打到 `snap-access.../api/v2/chat/completions`，逐个 id 分别带与不带
`maas_type: benefit`：

| id | 不带 benefit 头 | 带 benefit 头 |
| --- | --- | --- |
| `glm-5.3-flash` | `InferHub.002002009.404` 未注册 | 200，扣免费池 |
| `deepseek-v4.1-flash` | `81111.429` TPM 限流（已注册） | 200，扣免费池 |
| `deepseek-v4-flash-0731` | `InferHub.002002009.404` 未注册 | 200，扣免费池 |
| `deepseek-v4-pro-0813` | `InferHub.002002009.404` 未注册 | 200，扣免费池 |
| `deepseek-v4-flash`（裸） | 已注册（`81114.429` 限流） | `InferHub.4005.200` 不支持该头 |
| `deepseek-v4-pro`（裸） | `ModelArts.81009.404` | `InferHub.4005.200` 不支持该头 |

计费池归属（`statistics/plugin` 的 `package_credit_used` 与
`tokens/balance` 的 `monthly_tokens_used`，等待 75 秒结算后取值）：

| 模型 | 请求规模 | Δ积分 | Δ免费池 |
| --- | --- | --- | --- |
| `deepseek-v4-flash-0731` | max_tokens 400 | 0.00 | +504 |
| `deepseek-v4-pro-0813` | max_tokens 32 | 0.00 | +118 |
| `glm-5.3-flash` | max_tokens 32 | 0.00 | +65 |
| `deepseek-v4.1-flash` | max_tokens 32 | 0.00 | +83 |

结论：`gateway/config` 公布的 4 个 id **全部**是 benefit 模型并扣免费池；
带后缀与去后缀是后端**两个不同模型**（计费口径相反），不能合并成一行。

## 解法

- `parseModelInfo` 以原始 id 为行 id，去后缀只用于展示名与 `CONTEXT_WINDOWS` 查表；
  去重按原始 id（不再按归一化 id）。
- `fetchRemoteModels` 对 `gateway/config` 的每个模型都标 `benefit` 并记入
  `benefitModelIds`，与 `signedFetch` 发送 `maas_type: benefit` 的判定同源。
- `BENEFIT_MODEL_FALLBACK` 与 `CATALOG` 改用后端公布的原始 id。
- 出厂目录移除 `GLM-5`（接口对它返回 `InferHub.002002009.404`，
  列出只会得到一行调不通的模型）。`GLM-5.1` 实测 200，保留。

## 涉及模块

- [CodeArts provider 插件](../entities/codearts-plugin.md)：`parseModelInfo`、
  `fetchRemoteModels`、`isBenefitModel`、`BENEFIT_MODEL_FALLBACK`、`CATALOG`。
- [用量窗口只让总积分参与路由](../decisions/codearts-usage-window-routing.md)：
  免费额度窗口的 `models` 字段现在拿到的是原始 id。

## 复发预防

- 判断「某模型带什么头可调用」必须用**注册状态**（错误码）作判据，不要用积分或
  免费池的增量。增量会被账号上的并发活动（例如 magpie 宿主自身在跑）污染：
  同一模型在负载高时可能读到明显偏大的增量，与本次请求无关。
- 429/TPM 限流响应只能说明模型已注册，对计费口径没有信息量。
- `deepseek-v4.1-flash` 带不带 benefit 头都能进到限流层，此时只有「带头的请求
  是否扣免费池」能区分口径，不能凭无头的错误码判定它属于付费池。
- 上游参考实现 `Ebony-Vinyl/dsh-our-free-model` 的 `models.ts` 明确要求「带日期
  后缀的 id 归一化后绝不能标 benefit」。该注释记于 2026-09-23，与本次实测相反：
  它导致那两个免费模型以裸 id 下发而不可用。若将来出现分歧，以本页实测为准。