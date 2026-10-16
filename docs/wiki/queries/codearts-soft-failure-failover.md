---
title: CodeArts 软失败不触发故障切换与状态码换算
type: query
tags: [CodeArts, magpie, 故障切换, 状态码, 凭据续期, 排查]
created: 2026-10-16
updated: 2026-10-16
sources: []
status: active
---

# CodeArts 软失败不触发故障切换与状态码换算

## 问题

两个 CodeArts 账户从不互相切换，用户侧看到的是：

- 请求返回 `Stream ended without finish_reason`（流没有 `finish_reason` 就结束）。
- 请求返回 `Bad request: the security token has expired` / `APIG.0602`，
  之后同一账号继续失败，另一个账户不接手。

路由日志（2026-10-16，667 条）显示 4 条 400 全部由临时凭据过期引起，其中
3 条只尝试 1 次、1 条尝试 2 次，几乎没有发生账户切换。

## 根因

magpie 官方插件文档「失败与故障切换」写明：

> 非 2xx 的响应就是厂商的回答，网关会像对待其他供应商一样处理：限流或服务端错误
> 会把请求转到下一个账号或模型，401 会标记账号。

也就是说，**magpie 只在非 2xx 时切换账户**。CodeArts 的三类软失败都发生在 200
通道里，因此对 magpie 不可见：

| 软失败 | 上游形态 | magpie 的判定 | 用户看到 |
| --- | --- | --- | --- |
| 额度用尽 | HTTP 200 + SSE 内嵌 `error_code: InferHub.4291.200` | 成功（答案） | 流无 `finish_reason` |
| 其他终态错误 | HTTP 200 + SSE 内嵌 `error_code` | 成功（答案） | 同上 |
| 临时凭据过期 | HTTP 400 + `APIG.0602` | 400「其他」类，不切换 | 报错后停在同一账户 |

其中凭据过期一条还叠加了第二个问题：插件此前只在 `cred.expires_at` 临近时刷新
（`REFRESH_LEAD_MS`），上游 400 后不重签，于是同一账户反复失败。

## 解法

按 magpie 的判定规则反推：把上游软失败换算成它会切换账户的状态码，同时尽量
保留厂商原话（插件可用的错误格式回答，agent 会显示厂商的原话）。

1. **SSE 内嵌终态错误 → 429 / 502**（`peekSseQueueError` + `classifySseHead`）：
   读到的首个事件里如果没有 `error_code`，但含内容（`delta.content` /
   `reasoning_content` / `tool_calls` / `finish_reason`），判定为正常答案；
   含 `error_code` 且可重试 → 走原有排队重试；含 `error_code` 且不可重试 →
   额度用尽（`4291`）回答 429，其余回答 502。两种都不是 2xx，magpie 都会把请求
   转到下一个账号或模型。已读字节照旧回放进转发流，正文完整。
   - 边界：上游可能先发一个只含 `role` 的空增量，再把终态错误放在下一个事件。
     判定以「有没有内容」为准，不按事件计数，因此不会把正常回答误判成错误，
     也不会在错误还没读完时就转发。
2. **凭据过期 → 就地刷新 + 重发一次**（`signedFetch` 内的 `renewCredential`）：
   HTTP 400 + `APIG.0602` 或 `security token has expired` 时先刷新凭据、重签、
   重发同一请求（请求体签名前已缓冲为字节，可原样重放）。刷新成功即透传上游回答。
3. **刷新令牌失效 → 401 + 标记账号**：`STS5.1806` / `invalid refresh token` /
   `refresh token has been used` 表示一次性刷新令牌已被用掉，账号只能重新登录。
   此时回答 401 并带 `X-Magpie-Sign-In: expired`，magpie 据此标记账号、停止往它
   发请求，并转到下一个账户。同一进程内后续请求直接回答 401，不再打网络。
4. **401 / 403 一律补 `X-Magpie-Sign-In: expired`**，与 1.4 版本以来的行为一致。

## 复发预防

- **刷新令牌是一次性的**：刷新后必须把新令牌写回账号（`persistCredential`）。
  任何不落盘的诊断调用都会把存储的令牌烧掉，账号只能重新登录。两个测试账户
  的刷新令牌都因诊断探测失效，实测只能改用重新登录。
- **不要只按状态码判断成功**：CodeArts 把额度用尽、终态错误放在 200 的 SSE 里，
  只看状态码会把失败当成功透传给调用方。
- **区分「过期」与「失效」**：过期可恢复（刷新后重发），失效不可恢复（401 标记）。
  对失效令牌反复刷新只会拖延，且会消耗一次网络往返。

## 涉及模块

- `index.mjs`：`peekSseQueueError`、`classifySseHead`、`sseErrorCode`、
  `isQuotaExhaustedCode`、`isExpiredCredentialResponse`、
  `isPermanentRefreshFailure`、`sseEventHasContent`、`signInExpiredResponse`、
  `signedFetch` 的重试分支、loader 的 `renewCredential` / `signInDead`。
- `index.test.mjs`：新增 4 项测试（429 换算、502 换算、空增量后正常回答、
  刷新重发与 401 标记）。

## 相关页面

- [CodeArts provider 插件](../entities/codearts-plugin.md)
- [CodeArts 并发会话上限排查](codearts-tm00001041-session-cap.md)
- [排队重试与并发闸门决策](../decisions/codearts-session-queue-retry.md)
- [非流式请求不计费与免费额度口径](codearts-nonstream-metering.md)
