---
url: /posts/84288c819ccf4ed05c52da2133ae645e/
title: 625 - 迁移与 Bridge 两批页面的结构清点：谁写了步骤、谁只有一句话
date: 2027-05-08T10:00:00+08:00
updated: 2027-05-08T10:00:00+08:00
author: cmdragon

summary:
  摘要：迁移与 Bridge 两批文档一共 **21 页**（`migration/` 11 页 ＋ `bridge/` 10 页），但**篇幅差了两个数量级**。按标题层级逐页数下来：**migration 侧除去每页末尾的 `## Sitemap`，实质有 31 个 `##` 小节、25 个 `###`、5 个 `####`；bridge 侧是 33 个 `##`、22 个 `###`、`####` 一个都没有**。最厚的是 `migration/component-options`（11 个 `##`）与 `migration/configuration`（7 个），最薄的是四页只写着"一个步骤小节 ＋ Sitemap"的页面（`auto-imports`、`runtime-config`、`server`、`bundling`）。**顺带纠正一件被反复引用的事：官方从没给过"迁移一共多少条变更"这个总数**——任何"×× 条"的数都得自己写明是按什么数的，这一篇给的就是按标题层的计数口径与结果。

categories:
  - 前端开发

tags:
  - nuxt4
  - 迁移文档
  - bridge文档
  - 文档结构
  - 升级迁移
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

有人问"从 2 升到 4 到底有多少处要改"。这类问题在文档里没有答案页——不过**把 21 页的标题层级数一遍，至少能知道哪几页值得点进去。**

## 计数口径先说清

下面的数字全部按 Markdown 标题层级数：**`##` 是页内一级小节、`###` 是子小节、`####` 再下一层**。每页末尾都挂着一个 `## Sitemap`，**已在统计中扣除**（migration 11 个、bridge 10 个）。所以"实质 `##`"= 数到的 `##` 总数减去页数。

## migration：11 页

| 页面 | `##`（含 Sitemap） | `###` | `####` |
| --- | --- | --- | --- |
| overview | 2 | 0 | 0 |
| configuration | 7 | 3 | 3 |
| auto-imports | 2 | 0 | 0 |
| meta | 2 | 3 | 0 |
| pages-and-layouts | 6 | 7 | 2 |
| component-options | 11 | 3 | 0 |
| plugins-and-middleware | 3 | 2 | 0 |
| runtime-config | 2 | 0 | 0 |
| bundling | 2 | 0 | 0 |
| server | 2 | 1 | 0 |
| module-authors | 3 | 7 | 0 |

合计 **42 / 25 / 5**，扣除 11 个 Sitemap 后**实质 `##` 是 31 个**。

从这张表能读出三件事：

- **两节承担了主要篇幅**：`component-options`（11 个 `##`，十个组件选项一人一节，另加一个 `## Sitemap`）与 `configuration`（7 个 `##`，且是唯一有 `####` 的页之一）。
- **四页薄到只有一步**：`auto-imports`、`runtime-config`、`server`、`bundling` 的 `##` 都是 2，也就是"一个内容小节 ＋ Sitemap"——**这几页就是标题党式的一页一句话**（比如 auto-imports 的迁移只有一条编号）。
- **结构层级不统一**：`configuration` 用了 `####`（三条禁用规则挂在 `### Migration` 下），`meta` 用 `###`（useHead、Meta-components、Options API），`pages-and-layouts` 三种层级都有。**同一批文档没有统一的层级规范，所以按标题深度做导航会翻车。**

## bridge：10 页

| 页面 | `##`（含 Sitemap） | `###` |
| --- | --- | --- |
| overview | 4 | 4 |
| configuration | 3 | 2 |
| typescript | 3 | 1 |
| bridge-composition-api | 4 | 6 |
| plugins-and-middleware | 4 | 0 |
| nuxt3-compatible-api | 8 | 3 |
| meta | 5 | 2 |
| runtime-config | 2 | 0 |
| nitro | 6 | 4 |
| vite | 4 | 0 |

合计 **43 / 22**，扣除 10 个 Sitemap 后**实质 `##` 是 33 个**，`####` 一个都没有。

三点观察：

- **最厚的是 `nuxt3-compatible-api`（8 个 `##`）**——因为它一个旧 API 一节；其次是 `nitro`（6 个，全是操作步骤小节）。
- **`vite`、`runtime-config`、`plugins-and-middleware` 三节最短**，`vite` 只有 3 个内容小节 ＋ 一条前置警告。
- **Bridge 的层级比 migration 规整**：全部只用两层（`##` 与 `###`），没有 `####`；**它的"目录"作用在 overview 的八条 Upgrade Steps 清单上**（那八条链接顺序就是阅读顺序）。

## 为什么别引用"总条数"

两批文档里都**没有任何一页写过"共多少条变更"**：`migration/overview` 只列了三条大改动，`bridge/overview` 只列了八块能力。所以网上流传的"×× 条变更"都是各人自己数的结果——**上面这两张表就是可复算的一种口径**：实质 `##` 31（migration）＋ 33（bridge）＝ **64 个小节**，`###` 47、`####` 5。

要换口径也可以，但**必须写明数的是什么**：按小节数、按编号步骤数（各页步骤条数不等，`configuration` 三组各 3 条、`pages-and-layouts` 两组各 3 条）、还是按代码块数——**这几个数互不相等，谁也不能冒充"官方统计"。**

## 自查

- 21 页：migration 11、bridge 10；**实质 `##` 64 个（31 ＋ 33）、`###` 47、`####` 5（全在 migration 侧）**。
- 最厚两节：`migration/component-options`（11 个 `##`）、`bridge/nuxt3-compatible-api`（8 个）。
- 最薄若干页只有一步或一段：migration 的 `auto-imports`/`runtime-config`/`server`/`bundling`，bridge 的 `vite`/`runtime-config`。
- **官方从未给出变更总条数**；引用任何"×× 条"时先写明口径。
- 层级规范两批不一致，**别用标题深度做自动导航**。

## 下一篇

下一篇 626 把"升级后该跑什么"这件事按出处收齐：`nuxi dev` 与 `nuxi build`/`nuxi generate` 的试跑要求写在 Bridge 的 Nitro 页、dev 与 build **双验**写在 Bridge 的 overview、**测试前必须 `nuxi prepare`**（否则 `TS5083`）写在 Bridge 的 TypeScript 页、`nuxt typecheck`（走 `vue-tsc`）写在迁移的配置页、跑测试的命令与依赖写在测试那一节——**文档没有集中的"升级验收清单"，这一篇只做归拢，不补没有的步骤**。

## 本文依据

- https://nuxt.com/docs/4.x/migration/overview 与同目录下 configuration、auto-imports、meta、pages-and-layouts、component-options、plugins-and-middleware、runtime-config、bundling、server、module-authors 共 11 页（逐页 `##`／`###`／`####` 标题计数；`## Sitemap` 为每页末尾固定小节）
- https://nuxt.com/docs/4.x/bridge/overview 与同目录下 configuration、typescript、bridge-composition-api、plugins-and-middleware、nuxt3-compatible-api、meta、runtime-config、nitro、vite 共 10 页（同上口径；bridge 侧无 `####`）
- https://nuxt.com/docs/4.x/migration/configuration（唯一列出编号步骤最多的页之一，"九步"口径见本系列 605 篇所依据的原文）
- https://nuxt.com/docs/4.x/bridge/overview（八条 Upgrade Steps 清单，作为 bridge 侧的推荐阅读顺序）
