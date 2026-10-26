---
type: index
updated: 2026-10-26
---

# Wiki 索引

> 人工/Agent 显式查阅用的完整目录；会话中的导航由 wiki-memory hook 注入的目录树提供。
> 格式：`[页面标题](相对路径) — 一行摘要`。

## 实体 entities

- [CodeArts provider 插件（opencode-codearts-auth）](entities/codearts-plugin.md) — magpie / OpenCode 的码道模型接入插件：OAuth 登录、请求签名、模型目录与用量

## 概念 concepts

## 源总结 sources

## 决策 decisions

- [CodeArts 会话排队重试与并发闸门](decisions/codearts-session-queue-retry.md) — 账号级并发闸门 + `TM.00001041` 排队重试 + 稳定会话 id
- [CodeArts 用量窗口只让总积分参与路由](decisions/codearts-usage-window-routing.md) — 免费额度窗口已随体系下线移除；其余窗口 `aside: true`，路由上限只看总积分
- [CodeArts 仓库采用 main 与 develop 双分支模型](decisions/codearts-branch-model.md) — 唯一远程 fork；任务在 develop 完成后快进合回 main，临时分支用完即删

## 查询沉淀 queries

- [CodeArts TM.00001041 并发会话上限排查](queries/codearts-tm00001041-session-cap.md) — issue #1 的根因、解法、复发预防与上游 PR #4 提交状态
- [CodeArts 非流式请求不计费与免费额度口径](queries/codearts-nonstream-metering.md) — 网关只对 SSE 流式请求做用量归集（免费池部分已成历史记录）
- [CodeArts 免费模型 id 与计费池归属](queries/codearts-benefit-model-ids.md) — 已被取代：2026-10-26 免费额度体系下线
- [CodeArts 免费额度体系下线与计费口径迁移](queries/codearts-benefit-retired.md) — `gateway/config` / `tokens/balance` 返回 4000；旧 4 个 benefit id 全部失效；模型列表与名称改由 `v1/model/builtin` 自动获取
- [CodeArts 软失败不触发故障切换与状态码换算](queries/codearts-soft-failure-failover.md) — magpie 只在非 2xx 时切换账户；额度用尽 429、其他终态 SSE 错误 502、凭据过期刷新后重发、刷新令牌失效 401 标记账号
