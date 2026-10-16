---
type: index
updated: 2026-10-16
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

## 查询沉淀 queries

- [CodeArts TM.00001041 并发会话上限排查](queries/codearts-tm00001041-session-cap.md) — issue #1 的根因、解法、复发预防与上游 PR #4 提交状态
- [CodeArts 非流式请求不计费与免费额度口径](queries/codearts-nonstream-metering.md) — 网关只对 SSE 流式请求做用量归集；benefit 模型扣每日 token 免费额度而非积分
