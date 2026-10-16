---
title: CodeArts 用量窗口只让总积分参与路由
type: decision
tags: [CodeArts, 用量, 路由, magpie, aside, free]
created: 2026-10-16
updated: 2026-10-16
sources: []
status: active
---

# CodeArts 用量窗口只让总积分参与路由

## 背景（现状与约束）

`statistics/plugin` 返回四个积分口径：总积分（`usageTotalPackageCredit`）、基础积分包、
按需积分包、赠送积分包；另有 `opengw/v1/user/tokens/balance` 的每日 token 免费额度。
[插件](../entities/codearts-plugin.md)把它们全部映射成 magpie 的 `windows`。

magpie 的额度上限判定逐个读取插件下发的窗口。插件文档对窗口字段的定义：
`aside: true` 表示「用完也不影响账号使用」，`models` / `notModels` 限定窗口统计哪些模型
id。magpie 二进制里的上限文案 `is at %.0f%% of a usage window, past its %d%% cap`
说明窗口列表里每一项都是上限来源。

实机观察（同日两账号）：

| 账号 | 总积分 | 基础积分包 | 赠送积分包 | 免费额度 |
| --- | --- | --- | --- | --- |
| `codearts#no74gu` | 70.86%（2.5万/3.5万） | **100%**（500/500） | 70.43% | 0% |
| `codearts` | 0.16%（10/6500） | 0% | 0.17% | 0.90% |

四个积分口径不是四份独立的钱：桶是总积分的组成部分。

## 备选方案（各方案优劣）

| 方案 | 优点 | 缺点 |
| --- | --- | --- |
| A. 所有窗口都参与路由（不写 `aside`） | 无需判断哪个是主口径 | 基础包 500 用满即把剩 3 万积分的账号判为不可用；免费池耗尽会阻断付费模型 |
| B. 只让「总积分」参与，其余 `aside: true`，免费池再用 `models` 限定 | 上限口径唯一，与厂商自身的额度归属一致 | 三个桶不再单独触发上限（本就是总积分的组成部分） |
| C. 插件自己按比例算一个复合上限，只下发一个窗口 | 面板简洁 | 丢掉各桶明细；`opengw` 的免费池与积分池互斥，无法合成一个百分比 |
| D. 靠模型 `free: true` 让宿主自行跳过上限 | 依赖宿主语义，插件无需处理窗口标记 | 文档未说明 `free` 参与路由判定，只说它「不占额度」并在选择器显示；把路由正确性押在未记载行为上 |

## 决策（选定方案）

选 **B**：

- `creditWindow(name, limit, remain)`：只有「总积分」不写 `aside`，其余三个桶一律
  `aside: true`。
- `benefitWindow(raw, ids)`：`aside: true` 且 `models: [...ids].sort()`，窗口只统计
  benefit 模型。
- 免费模型在列表里带 magpie 模型字段 `free: true`；判定继续走 `isBenefitModel`
  单一来源（与发送 `maas_type: benefit` 头同一处），出厂目录与实时列表都用它。

## 理由（决策依据）

1. 桶是总积分的组成，不是并列额度：让桶单独决定上限与厂商计费口径冲突。
2. `aside` 的文档定义正好是需要的语义——「用完也不影响账号使用」。
3. 免费池与积分池互斥（见
   [非流式请求不计费与免费额度口径](../queries/codearts-nonstream-metering.md)），
   免费池耗尽只影响免费模型，`models` 把这一点表达清楚。
4. 方案 D 不可靠：`free` 是展示与额度归属字段，路由行为未在文档中承诺。

### 宿主级 A/B 实测（沙箱 HOME，不碰真实配置）

`magpie quota --json` 的 `holdsWhy` 字段反映窗口是否参与路由上限：

| 版本 | 总积分 | 基础积分包 | 赠送积分包 | 免费额度 |
| --- | --- | --- | --- | --- |
| 0.1.3（旧） | `holdsWhy=low` | `holdsWhy=low` | `holdsWhy=low` | `holdsWhy=low` |
| 0.1.4（新） | `holdsWhy=low` | 无 | 无 | 无 |

旧版四个窗口全部参与上限判定；新版只有「总积分」保留。no74gu 账号基础包 100% 用满，
在新版下不再影响账号判定。

模型目录侧：`magpie plugin --json` 回显 `glm-5.3-flash` / `deepseek-v4.1-flash` 为
`free=true`，其余 7 个为 `free=false`；`magpie provider test codearts glm-5.3-flash` 与
`GLM-5.2` 均 ✓ chat，标注与计费头未出现不一致。

## 涉及文件

- 用量面板仍显示四个窗口，用户可见信息不减少；只有「总积分」影响账号是否被暂停。
- 新增用量窗口默认必须显式决定路由参与方式（不写 `aside` 即参与），避免误把
  子口径当主口径。
- `free` 标注与 benefit 头共用判定，不会出现「标免费但按积分计费」的不一致；
  若后端日后为 benefit 模型新增其他识别字段，需要同时更新 `isBenefitModel`。
- 待核验：magpie 对 `models` 的 id 匹配是否包含带日期后缀的原始 id。当前插件下发的
  模型 id 已去后缀，与 `maas_type: benefit` 实际发送的 id 一致，因此不涉及；
  若将来改为保留原始 id 下发，需回来核对。
