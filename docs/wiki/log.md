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

## [2026-10-16] feat | 免费模型标注 free:true，用量窗口只让总积分参与路由上限（0.1.4）

## [2026-10-16] decision | 沉淀用量窗口路由口径的 ADR：四方案对比、aside/models 依据与实机双账号核对

## [2026-10-16] decision | 沉淀分支模型 ADR：删除无独有内容的临时分支，改用 main + develop 双分支，规则写入 CONTRIBUTING.md

## [2026-10-16] query | 实测 gateway/config 四个 id 的注册与计费归属：全部为 benefit，带日期后缀与裸形是两个不同模型；补全 free 标注并移除未注册的 GLM-5（0.1.5）

## [2026-10-16] fix | 免费模型按后端公布的原始 id 下发，去重改按原始 id；出厂目录与静态兜底同步补全（0.1.5）
