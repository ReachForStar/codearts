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

## [2026-10-16] query | 实测 magpie 只在非 2xx 时切换账户：200 内嵌终态错误、400 凭据过期都不切换；按状态码换算（429/502/刷新重发/401 标记）修复账户故障切换（0.1.6）

## [2026-10-26] query | 实测码道免费额度体系下线：gateway/config 与 tokens/balance 返回 4000 invalid channel，旧 4 个 benefit id 全部失效（3 个未注册、deepseek-v4.1-flash 转积分计费且拒 benefit 头）

## [2026-10-26] fix | 删除 benefit 头、free 标注、免费额度窗口与两个失效端点；模型列表只读 v1/model/builtin，显示名不再去日期后缀，出厂目录换成当前 8 个模型（0.1.7）

## [2026-10-26] query | 沉淀收费口径迁移：IDE 内置列表与 builtin 接口逐项核对、倍率与错峰时段、复发预防

## [2026-10-26] feat | 实现 magpie 的 auth.checkin 每日签到：ops/delivery 找 USER_LOGIN → claim → confirm，回 claimed/done/inactive/ineligible/failed（0.1.8）

## [2026-10-26] query | 沉淀签到接口契约与实测：已领取判定（claimable=false + CONFIRMED）、claim 被拒文案、路由头不参与签名
