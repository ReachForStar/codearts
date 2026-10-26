---
title: CodeArts provider 插件（opencode-codearts-auth）
type: entity
tags: [插件, magpie, opencode, oauth, 码道, 华为云]
created: 2026-10-14
updated: 2026-10-26
sources: []
status: active
---

# CodeArts provider 插件

## 职责

把华为云码道（CodeArts）模型接入 magpie / OpenCode：用 PKCE + DPoP 的 OAuth 登录
换取临时 AK/SK/security-token，再用 SDK-HMAC-SHA256 给每个模型请求签名。
码道网关是 OpenAI 兼容的 Chat Completions，但不接受 API Key。

## 关键文件

| 文件 | 说明 |
| --- | --- |
| `index.mjs` | 全部实现：签名（`signRequestHuawei` / `buildCanonicalRequest`）、PKCE 与 DPoP（`generatePkcePair` / `generateDpopKeyPair` / `signDpopJws`）、本地回调服务器（`startOAuthCallbackServer` / `startOAuthFlow`）、凭据续期（`requestToken` / `refreshCredential`）、请求签名与排队（`signedFetch` / `queryQueueStatus` / `createConcurrencyGate`）、模型目录（`fetchRemoteModels`）、用量（`usageFromStatistics`） |
| `index.test.mjs` | `node --test`，56 项，全部注入 `options.fetch`，不联网 |
| `README.md` | 安装、登录、使用、已知限制 |
| `DESIGN.md` | 参考实现映射、改造说明、并发与会话设计 |
| `registry-entry.json` | magpie 插件市场登记项 |

## 上游端点与运行契约

- 端点固定华为云 `cn-north-4`：Chat Completions 基址 `snap-access.../api/v2`；
  模型列表 `snap-access.../v1/model/builtin`（唯一来源）；用量
  `snap-access.../snap-manager/v1/statistics/plugin`；排队状态
  `snap-access.../api/v1/queue/status`；授权页 `codearts.huaweicloud.com/portal/authorize`；
  STS 换凭据 `sts.cn-north-4.../v1/oauth2/tokens`。
- magpie 契约：`loader(getAuth)` 返回 `{ baseURL, fetch }`；`fetch(input, init)` 的
  `init.signal` 在 agent 放弃请求时中止；非 2xx 会被网关当作厂商回答做故障切换
  （不是重试）。
- 凭据整包以 JSON 存在账号的 `access` 字段，含登录时生成、续期保留的 `session_id`
  （用于稳定的 `Session-Id` / `Chat-Id`）；续期有两条通道：`auth.refresh` 钩子，
  以及 loader 内针对忽略该钩子的宿主（OpenCode / 旧 magpie）的单飞兜底。
- **模型 id、显示名与收费状态都来自 `v1/model/builtin`**，插件不硬编码模型名。
  `parseModelInfo` 按原始 id 去重、`model_name` 原样展示：`deepseek-v4.1-flash` 与
  `deepseek-v4.1-flash-1017` 是后端并存的两个模型（倍率不同），去后缀合并会让列表
  出现两行同名。日期后缀只用于 `CONTEXT_WINDOWS` 查表。视觉模型（`-vl`）过滤。
- **免费额度体系已下线**（2026-10-26）：`maas_type: benefit` 头、
  `gateway/config` 免费模型集、`user/tokens/balance` 每日额度、`free: true` 标注与
  「免费额度」用量窗口全部移除，全部模型按积分计费。实测与迁移见
  [免费额度体系下线与计费口径迁移](../queries/codearts-benefit-retired.md)。
- **只有流式请求被计量**：非流式的 200 回答返回 tokens 却不动 `package_credit_used`。
  `signedFetch` 在签名前把对话体改写为 `stream: true`（请求体参与 payload hash，必须先改再签），
  再用 `parseSseEvents` / `completionFromChunks` 折叠回调用方要求的 JSON；流内无 `choices`
  时判为上游错误，原样透传。详见
  [非流式请求不计费](../queries/codearts-nonstream-metering.md)。
- 计费只有一个池：积分（`usageTotalPackageCredit` 等四个桶）。
  `auth.usage` 只映射积分口径，不再有 tokens 单位的窗口。
- **路由只看「总积分」窗口**：其余三个积分桶标 `aside: true`（用完不影响账号使用）。
  依据与实测见
  [用量窗口只让总积分参与路由](../decisions/codearts-usage-window-routing.md)。

## 重要变更记录

- 2026-10-14：修复 `TM.00001041`（并发会话数已达上限）导致的不可用——新增账号级
  并发闸门、排队重试与状态探测、稳定 `Session-Id` / `Chat-Id`；版本 0.1.1。
  详见 [CodeArts 并发会话上限排查](../queries/codearts-tm00001041-session-cap.md)与
  [排队重试与并发闸门决策](../decisions/codearts-session-queue-retry.md)。
- 2026-10-14：补齐同一条上限的第二个通道——HTTP 200 + SSE 内嵌 `error_code` 在
  交回响应前检测并重放请求；视觉模型过滤改为大小写不敏感（小写 `-vl` 此前漏过）；
  版本 0.1.2。
- 2026-10-16：修复插件全部对话请求脱离计费的问题（强制流式 + SSE 折叠回 JSON），
  用量面板增加 benefit 每日 token 免费额度窗口；版本 0.1.3。详见
  [非流式请求不计费与免费额度口径](../queries/codearts-nonstream-metering.md)。
- 2026-10-16：免费模型标注 `free: true`，用量窗口只让「总积分」参与路由上限；
  版本 0.1.4。详见 [用量窗口只让总积分参与路由](../decisions/codearts-usage-window-routing.md)。
- 2026-10-16：补全免费模型集合：`gateway/config` 的 4 个 id 全部按原始形态下发并标
  `free: true`（此前 2 个被去后缀改写后与付费 id 相撞，既无标注也调不通）；出厂目录
  移除未注册的 `GLM-5`；版本 0.1.5。详见
  [免费模型 id 与计费池归属](../queries/codearts-benefit-model-ids.md)。
- 2026-10-16：把上游软失败换算成 magpie 会切换账号的状态码：额度用尽（SSE 内嵌
  `4291`）→ 429、其他终态 SSE 错误 → 502、凭据过期（`APIG.0602`）→ 就地刷新后重发
  一次、刷新令牌失效（`STS5.1806`）→ 401 + `X-Magpie-Sign-In: expired`；版本 0.1.6。
  详见 [软失败不触发故障切换与状态码换算](../queries/codearts-soft-failure-failover.md)。
- 2026-10-26：免费额度体系下线，删除 benefit 头、免费标注、免费额度窗口与两个失效
  端点，模型列表只读 `v1/model/builtin`，显示名不再去日期后缀，出厂目录换成当前
  8 个模型；测试 56 项。详见
  [免费额度体系下线与计费口径迁移](../queries/codearts-benefit-retired.md)。
