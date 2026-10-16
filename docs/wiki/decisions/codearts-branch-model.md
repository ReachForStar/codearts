---
title: CodeArts 仓库采用 main 与 develop 双分支模型
type: decision
tags: [Git, 分支模型, develop, main, fork]
created: 2026-10-16
updated: 2026-10-16
sources: []
status: active
---

# CodeArts 仓库采用 main 与 develop 双分支模型

## 背景（现状与约束）

2026-10-16 仓库同时存在三条分支，内容关系如下：

| 分支 | 提交 | 与 `main` 的关系 |
| --- | --- | --- |
| 本地 `fix/codearts-session-cap` | `b67b053` | 已是 `main` 的祖先，无独有提交 |
| `fork/stream-metering` | `8bfd8b5` | 与 `main` 同一提交 |
| `main` | `8bfd8b5` | — |

`b67b053` 引入的并发会话排队能力（`SSE_PEEK_LIMIT`、`peekSseQueueError`、
`queryQueueStatus`、`queueTerminalResponse`、`isRetryableQueueCode`、
`TM.00001041` 判定）在 `main` 中逐项存在，函数名出现次数一致，功能没有丢失。
排查记录见
[TM.00001041 并发会话上限排查](../queries/codearts-tm00001041-session-cap.md)。

远程侧：上游 `yuweihao17/codearts`（`origin`）由用户移除，仓库只剩 `fork`
（`ReachForStar/codearts`）。上游 `main` 停在 `c594984`，比本地落后 10 个提交；
`ReachForStar` 对上游无写权限，此前两个 PR 均已关闭。

长期存在的临时分支带来两处成本：同一改动要在多条分支重复推送；分支与 `main`
分叉后只能靠 `merge -s ours` 之类的特殊手段处理，见
[非流式请求不计费与免费额度口径](../queries/codearts-nonstream-metering.md)
所在提交的历史合并。

## 备选方案（各方案优劣）

| 方案 | 优点 | 缺点 |
| --- | --- | --- |
| A. 单 `main` 分支直接开发 | 无合并开销，分支永不分叉 | 未验证的中间提交直接进入可用分支 |
| B. `main` + `develop` 双分支，任务在 `develop` 完成后合回 `main` | 可用分支只含验证过的代码；任务之间不互相污染；两条分支都能推送备份 | 每个任务多两次合并操作 |
| C. 每个任务一条 `feature/*` 分支 | 并行任务隔离彻底 | 分支数量随任务线性增长，删分支与分叉处理成本回到现状 |
| D. 走 PR 流程（含上游） | 评审留痕 | 上游无写权限，PR 只能关闭；fork 内自评 PR 无额外收益 |

## 决策（选定方案）

选 **B**，规则落在 [CONTRIBUTING.md](../../../CONTRIBUTING.md)：

- 唯一远程 `fork`；`main` 只接受 `develop` 的合并，`develop` 承载日常提交。
- 任务开始前先把 `main` 合进 `develop`，任务完成后把 `develop` 合回 `main`，
  两条分支随后一起推送。
- 只允许快进推送；历史必须并入而内容以本地为准时，用 `merge -s ours` 并核对
  合并前后 `git diff --stat` 为空。
- 本次清理删除 `fix/codearts-session-cap`（本地）与 `stream-metering`（远端），
  删除依据是两者均无独有内容。

## 理由（决策依据）

1. 被删的两条分支都不含 `main` 没有的内容，删除不丢失功能，核对结果见「背景」表。
2. `develop` 承接「未验证不进 `main`」的边界，替代原先靠临时分支隔离未完成工作的做法。
3. 上游 PR 路线（方案 D）在权限上已不可行，`fork` 是唯一写入点，无需再为上游保留分叉分支。

## 后果（影响与后续）

- 后续任务不再新建一次性分支，直接在 `develop` 提交；确需隔离时任务结束即删。
- `main` 的历史只增加快进与合并提交，禁止改写，`fork/main` 可被
  `magpie plugin add https://github.com/ReachForStar/codearts` 直接安装。
- 上游 `yuweihao17/codearts` 与本地彻底脱钩；若将来恢复协作，需要重新添加远程并
  走 fork-based PR，届时先核对上游 `main` 是否已包含本地实现。
- 待确认：`CONTRIBUTING.md` 只约束本仓库，不代表上游 `yuweihao17/codearts` 的规则。
