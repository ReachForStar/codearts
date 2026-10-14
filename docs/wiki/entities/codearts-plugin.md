---
title: CodeArts provider 插件（opencode-codearts-auth）
type: entity
tags: [插件, magpie, opencode, oauth, 码道, 华为云]
created: 2026-10-14
updated: 2026-10-14
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
| `index.test.mjs` | `node --test`，37 项，全部注入 `options.fetch`，不联网 |
| `README.md` | 安装、登录、使用、已知限制 |
| `DESIGN.md` | 参考实现映射、改造说明、并发与会话设计 |
| `registry-entry.json` | magpie 插件市场登记项 |

## 上游端点与运行契约

- 端点固定华为云 `cn-north-4`：Chat Completions 基址 `snap-access.../api/v2`；
  模型列表 `snap-access.../v1/model/builtin` 与 `opengw.developer.huaweicloud.com/api/v1/gateway/config`；
  用量 `snap-access.../snap-manager/v1/statistics/plugin`；排队状态
  `snap-access.../api/v1/queue/status`；授权页 `codearts.huaweicloud.com/portal/authorize`；
  STS 换凭据 `sts.cn-north-4.../v1/oauth2/tokens`。
- magpie 契约：`loader(getAuth)` 返回 `{ baseURL, fetch }`；`fetch(input, init)` 的
  `init.signal` 在 agent 放弃请求时中止；非 2xx 会被网关当作厂商回答做故障切换
  （不是重试）。
- 凭据整包以 JSON 存在账号的 `access` 字段，含登录时生成、续期保留的 `session_id`
  （用于稳定的 `Session-Id` / `Chat-Id`）；续期有两条通道：`auth.refresh` 钩子，
  以及 loader 内针对忽略该钩子的宿主（OpenCode / 旧 magpie）的单飞兜底。
- benefit（免费额度）模型必须带参与签名的 `maas_type: benefit` 头，集合由
  `gateway/config` 动态识别并有静态兜底。

## 重要变更记录

- 2026-10-14：修复 `TM.00001041`（并发会话数已达上限）导致的不可用——新增账号级
  并发闸门、排队重试与状态探测、稳定 `Session-Id` / `Chat-Id`；版本 0.1.1。
  详见 [CodeArts 并发会话上限排查](../queries/codearts-tm00001041-session-cap.md)与
  [排队重试与并发闸门决策](../decisions/codearts-session-queue-retry.md)。
- 2026-10-14：补齐同一条上限的第二个通道——HTTP 200 + SSE 内嵌 `error_code` 在
  交回响应前检测并重放请求；视觉模型过滤改为大小写不敏感（小写 `-vl` 此前漏过）；
  版本 0.1.2。
