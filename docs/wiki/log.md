# Wiki 操作日志

> 只追加、不修改历史。可用 `grep "^## \[" docs/wiki/log.md | tail -5` 看最近记录。
> 前缀格式：`## [YYYY-MM-DD] <init|feat|fix|query|lint|chore> | <简述>`

## [2026-10-14] init | wiki-memory hook 自动创建知识库骨架

## [2026-10-14] fix | 修复 CodeArts TM.00001041 并发会话上限（排队重试、并发闸门、稳定会话 id），新增 entity/decision/query 三页

## [2026-10-14] fix | 补齐 SSE 内嵌排队错误的检测与重放、视觉模型过滤大小写修正（0.1.2）

## [2026-10-14] chore | 记录 magpie 侧并发与模型配置（accountConcurrency 3、queueLimit 8、rpm off）

## [2026-10-14] chore | 向上游提交 PR #4（仅代码，wiki 不入 PR），在 query 页记录提交状态与权限限制

## [2026-10-16] fix | 修复 CodeArts 非流式请求不计费：签名前强制 stream:true 并折叠 SSE 回 JSON，用量面板补 benefit 每日 token 免费额度（0.1.3）

## [2026-10-16] query | 沉淀非流式不计费的单变量实测证据与两条计费池口径，关联 entity 页与排队重试决策
