---
title: CodeArts 每日签到接口与领取状态
type: query
tags: [CodeArts, 签到, checkin, 积分, magpie, 接口]
created: 2026-10-26
updated: 2026-10-26
sources: []
status: active
---

# CodeArts 每日签到接口与领取状态

## 问题

magpie 需要每天在码道领取一次「每日领取」积分。码道没有公开文档，需要确认：

1. 客户端「限时福利 → 每日领取」实际调用的端点与请求体。
2. 怎么判断今天已领取，避免重复调用。
3. 领取成功 / 被拒 / 无活动时分别返回什么。

## 根因

magpie 的 `auth.checkin(getAuth)` 钩子要求插件自己实现签到；插件此前没有该钩子，
所以账号每天的积分没有自动领取。签到与对话请求不同：它走 `snap-access` 的 ops
接口，响应是 `{code, message, data}` 而不是 SSE，也不能走带并发闸门的
`signedFetch`。

## 实测（2026-10-26，账号 `codearts`，签名与路由头）

### 活动列表

`GET {SNAP_HOST}/v1/ops/delivery?channel=IDE`（签名请求，附未签名的
`Agent-Type: PromptCenter`、`X-Language: zh-cn`）返回 200：

```json
{"code":0,"message":"ok","data":{"channel":"IDE","serverTime":1793027733416,
 "items":[{"campaignId":1,"title":"每日签到领积分","type":"USER_LOGIN",
           "benefitAmount":1000,"claimable":false,"status":"CONFIRMED"}, …]}}
```

`items` 里同时有新用户注册、邀请、学生认证等活动，签到只取
`type === "USER_LOGIN"` 的那一项（实测 `campaignId: 1`，`benefitAmount: 1000`）。

### 领取与确认

- 未领取时该活动为 `claimable: true`、`status: "ENTRY"`。
- `POST {SNAP_HOST}/v1/ops/claim`，体 `{ campaignId, channel: "IDE" }`；成功返回
  `data.id` 与 `data.benefitAmount`。
- `data.id !== null` 时必须再 `POST {SNAP_HOST}/v1/ops/confirm`，体
  `{ campaignId }`，否则积分停在待确认。

### 已领取与拒绝

今日已领取时的实测返回：

| 调用 | 结果 |
| --- | --- |
| `GET v1/ops/delivery?channel=IDE` | `type USER_LOGIN` 项 `claimable: false`、`status: "CONFIRMED"`、`pendingCount: 0` |
| `POST v1/ops/claim`（已领取） | `{"code":…,"message":"尚未到达权益刷新时间，暂时无法领取"}` |

结论：`claimable: false` 且 `status` 属于 `CLAIMED` / `CONFIRMED` / `CONSUMED`
即为今天已领取，插件直接回 `done`，不发 claim 请求。claim 被拒时重读一次活动列表，
若此时已变成已领取状态（可能刚被 IDE 或另一个宿主领走），同样回 `done`。

## 解法

- `snapRequest(method, url, cred, fetcher, payload)`：通用签名请求，返回
  `{ok: true, data}` 或 `{ok: false, message}`，永不抛错；`code !== 0` 视为业务拒绝，
  非 2xx 用 `describeHttpFailure` 压成 `HTTP 401: APIG.0301 …` 一行。
- `dailyCampaign` 读活动列表并挑出 `USER_LOGIN`；`isClaimedToday` 判已领取；
  `dailyCheckin` 串起「列表 → 领取 → 确认」，回
  `claimed`（带 `credit`）/ `done` / `inactive` / `ineligible` / `failed`。
- `auth.checkin` 只做凭据提取与委派；未登录直接回 `failed`，不联网。

## 涉及模块

- [CodeArts provider 插件](../entities/codearts-plugin.md)：`snapRequest`、
  `dailyCampaign`、`isClaimedToday`、`dailyCheckin`、`auth.checkin`。
- [免费额度体系下线与计费口径迁移](codearts-benefit-retired.md)：签到积分进入
  积分池的「赠送积分包」口径，与已下线的免费额度无关。

## 复发预防

- 判断「今天能不能领」用列表里的 `claimable` + `status`，不要用积分余额变化；
  余额受对话消费影响，不可作为签到判据。
- 签到不要复用对话的 `signedFetch`：那里有并发闸门与排队重试，ops 接口会卡住。
- 路由头 `Agent-Type` / `X-Language` 不能进签名（`SignedHeaders` 里出现它们会被拒）。
- 每日领取有时间窗（活动 `extra.startTime` / `endTime` 与「权益刷新时间」），
  magpie 每个北京日调用一次即可，插件不做本地去重，靠后端状态判断。
