---
title: CodeArts TM.00001041 并发会话上限排查
type: query
tags: [码道, 并发会话, TM.00001041, 排查]
created: 2026-10-14
updated: 2026-10-14
sources: []
status: active
---

# CodeArts TM.00001041 并发会话上限排查

来源：GitHub issue #1
`[Bug] CodeArts models unusable at any normal rate — upstream TM.00001041 kills every request`
（https://github.com/yuweihao17/codearts/issues/1），2026-10-11 报告，仍为 open。

## 问题

模型列表正常，但任何正常调用频率下聊天请求都失败：

```
HTTP 400
{"error":{"message":"CodeArts: 400 Bad Request: {\"error_code\":\"TM.00001041\",
 \"error_msg\":\"并发会话数已达上限(3个)，请关闭部分会话后重试。\"}"}}
```

issue 实测：空闲后单发 1 次成功；并发 6 时 1 成功 5 失败；并发 8 时 2 成功 6 失败。
空闲 90 秒后单发又能成功，排除凭据过期、模型下线与账号封禁。

## 根因

1. **请求每次新开上游会话且从不释放**：`index.mjs` 中 `Chat-Id` 每请求随机、
   `Session-Id` 每个 loader 随机，没有会话关闭调用。上游会话在请求结束后仍存活
   60–90 秒并计入账号 3 会话上限，于是计数单调上涨直到每次请求都被拒。
2. **`TM.00001041` 完全未处理**：`signedFetch` 只特判 401/403，其余非 2xx 原样透传，
   请求路径上没有重试或退避。
3. magpie 的 `accountConcurrency: { "signed in": 2 }` 帮不上忙：它限制在途请求数，
   不清理仍然计数在服务端的会话。

## 解法

按 [排队重试与并发闸门决策](../decisions/codearts-session-queue-retry.md) 实施：

- 账号级并发闸门（在途 ≤ 3），发起前先排队；
- `400` + `TM.00001041` 每 10 秒重试，上限 180 次，重试前查
  `api/v1/queue/status`，`queue_full` / `error` 立即结束；
- `Session-Id` / `Chat-Id` 由登录时生成、随凭据持久化并跨续期保留的 `session_id` 派生；
- 等待期间响应 `AbortSignal` 中止。

## 涉及模块

- [CodeArts provider 插件](../entities/codearts-plugin.md)：`signedFetch`、
  `queryQueueStatus`、`createConcurrencyGate`、`accountSessionIds`、loader 的上下文构造。
- magpie 侧：`maxConcurrency`（每账号在途请求上限）与账号故障切换；`providers.json`
  的 `pickedFrom` 是宿主配置，不在本仓库。

## 复发预防

- 单测锁定四条判据：`isQueueErrorResponse` 只认 400 + `TM.00001041`；重试成功与
  耗尽两条路径的调用次数；终态状态立即结束；闸门峰值等于 3（`index.test.mjs`
  的 `upstream concurrency & queueing` 分组）。
- 稳定会话 id 由 `session and chat ids stay stable for one account` 与
  `sign-in mints a session id and refresh keeps it` 锁定，含凭据轮换与续期场景。
- 未覆盖的同类通道：HTTP 200 + SSE 内嵌 `error_code` 形态的排队错误
  （`InferHub.ModelArts.81111.429`）。若线上出现「思考后无输出」，先查这条。

## 存疑

- 会话存活时长（60–90 秒）与队列调度行为来自 issue 报告与参考实现注释，本机无凭据
  未复现。
- 次要问题：`openpangu-2.0-flash` 在账号有配额时仍返回 `content: null`、0 token
  （issue 记录）。该模型的可见性由 magpie 的 `providers.json` 决定，不在本仓库，
  本轮未处理。
