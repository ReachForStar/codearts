# CONTRIBUTING

本文规定分支模型与同步顺序。仓库只有一个远程：`fork`（`ReachForStar/codearts`）。

## 分支模型

| 分支 | 作用 | 谁能写 |
| --- | --- | --- |
| `main` | 已验证的可用代码，随时可发布 | 只接受 `develop` 的合并 |
| `develop` | 日常开发分支，任务在此完成 | 直接提交 |

两个分支都必须与 `fork` 上的同名分支保持一致。

## 同步顺序

开发任务按此顺序执行，不可跳步：

1. **先同步 `main`**：`git checkout main && git pull --ff-only fork main`。
   `main` 有未推送的新提交时先推送，再开始新任务。
2. **把 `main` 合进 `develop`**：`git checkout develop && git merge --ff-only main`。
   非快进说明 `develop` 有未合并的工作，改用 `git merge main` 产生合并提交。
3. **在 `develop` 上完成任务**：提交、跑 `node --test`、做实际验证。
4. **完成后合回 `main`**：`git checkout main && git merge --ff-only develop`，
   然后 `git push fork main` 与 `git push fork develop`。
5. **回到第 1 步**开始下一个任务。

## 强制要求

- 只允许快进推送。禁止 `git push --force`、`--force-with-lease`、`git rebase`
  改写已推送的历史、`git reset --hard`。
- 推送被拒（non-fast-forward）时立刻停止，查清分叉原因再处理，不用强推解决。
- 历史需要并入但内容以本地为准时，用 `git merge -s ours <远端>`，
  并在提交信息写明「保留本地实现」；合并后必须核对
  `git diff --stat <合并前提交> HEAD` 为空。
- 任务分支只在确有必要时创建（例如需要向上游提 PR）。任务分支合并进 `develop`
  后立刻删除本地与远端同名分支，不留长期分叉分支。
- 合并前 `node --test` 必须全绿。测试失败的状态不得进入 `main`。
- wiki 沉淀（`docs/wiki/`）随本次代码一并提交，不单独攒到任务收尾。
