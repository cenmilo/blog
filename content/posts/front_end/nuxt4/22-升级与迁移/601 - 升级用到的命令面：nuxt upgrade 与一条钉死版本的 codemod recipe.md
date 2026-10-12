---
url: /posts/69aebb56650cff29d3f52465dbce56a1/
title: 升级用到的命令面：nuxt upgrade 与一条钉死版本的 codemod recipe
date: 2027-04-30T10:00:00+08:00
updated: 2027-04-30T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：升级指南里能直接敲的命令一共**三类**。第一类是升到最新版：**`nuxt upgrade`**，按包管理器给了五条写法（`npx`／`yarn`／`pnpm`／`bun x`／`deno x`）。第二类是试下一大版本：不是命令，而是在配置里设 `future.compatibilityVersion: 5`（**需 Nuxt 4.2+**），官方同时提醒 Nuxt 5 "currently in development"。第三类是自动化迁移：**`npx codemod@0.18.7 nuxt/4/migration-recipe`**——版本号被钉死，原因写在同一行注释里：**codemod/codemod issue #1710**；这一条"会按顺序执行全部 codemod，并给你取消勾选其中任何一项的机会"。报告问题用 `npx codemod feedback`，完整清单在 Codemod Registry，合作方是 Codemod 团队。

categories:
  - 前端开发

tags:
  - nuxt4
  - 升级命令
  - codemod
  - 包管理器
  - 迁移自动化
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

3.x 升上来之后满屏红字，第一反应是"有没有命令能自动改"。答案有，但它不在 `nuxt` 命令下面，而是另一个工具的 recipe——而且**官方给的那条命令把版本号钉死了**。

## 第一类：升到最新版

升级指南的 Upgrading Nuxt 一节只有一句指令："To upgrade Nuxt to the latest release, use the `nuxt upgrade` command."，然后按包管理器给五条：

```bash
npx nuxt upgrade
yarn nuxt upgrade
pnpm nuxt upgrade
bun x nuxt upgrade
deno x nuxt upgrade
```

同一节下面还有一条 **Nightly Release Channel**：想用最领先的构建、在正式发布前试功能，走 nightly 那一页的指引。**这条不属于升级路径**，它是"跟着主干跑"的另一套用法。

Migrating to Nuxt 4 一节的开头给的是显式装版本，同样是五条：

```bash
npm install nuxt@^4.0.0
yarn add nuxt@^4.0.0
pnpm add nuxt@^4.0.0
bun add nuxt@^4.0.0
deno add npm:nuxt@^4.0.0
```

两组的差别不在覆盖的包管理器（**都是 npm／yarn／pnpm／bun／deno 五条**），而在形式：升级那一组里 npm 走的是 `npx nuxt upgrade`（npx 执行而不是一条 npm 子命令），安装那一组则是 `npm install nuxt@^4.0.0`；bun 与 deno 两边都用 `x`。**这一页没有解释这两组命令的先后关系**——按字面读，`nuxt upgrade` 是"升到最新"，`nuxt@^4.0.0` 是"显式钉到 4 这个大版本"。

## 第二类：试下一大版本，不是命令

Testing Nuxt 5 一节明确写着 Nuxt 5 **currently in development**，而提前测试的手段**不是命令行**，是配置：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  future: {
    compatibilityVersion: 5,
  },
})
```

前提句是 "**From Nuxt 4.2+** you can test many of its breaking changes by setting `future.compatibilityVersion` to `5`."。这一档的作用范围是那**十九项**默认行为变化，且官方用 `::note` 划了边界：这清单只说 Nuxt 4 上这个 flag 改的东西，**Nuxt 5 自身的其它变更（例如升到 Nitro v3）不是这个 flag 能开出来的**。

所以顺序是：先升版本（第一类），再决定要不要开这一档（第二类），最后才谈改代码（第三类）。

## 第三类：那条钉死版本的 recipe

Migrating Using Codemods 一节的开场交代了合作来源："To facilitate the upgrade process, we have collaborated with the **Codemod** team to automate many migration steps with some open-source codemods."

一条命令跑全套：

```bash
# Using pinned version due to https://github.com/codemod/codemod/issues/1710
npx codemod@0.18.7 nuxt/4/migration-recipe
```

这一节的四条信息都要收下：

1. **版本号是钉死的**，理由写在命令上面那行注释里：**因为 codemod/codemod 的 issue #1710**，所以用 `codemod@0.18.7`。想跑最新版之前先去看那个 issue 的状态。
2. **五条包管理器写法齐全**：`npx codemod@0.18.7 …`、`yarn dlx …`、`pnpm dlx …`、`bun x …`、`deno x …`。**yarn 用的是 `dlx`，而 bun／deno 用的是 `x`**——这组差异是文档明确给的。
3. **它是交互式的**："This command will execute all codemods in sequence, **with the option to deselect any that you do not wish to run**." —— 按顺序执行，每一项都可以取消勾选。
4. **每个 codemod 都能单独跑**："Each codemod is also listed below alongside its respective change and can be executed independently."，完整清单与详情在 Codemod Registry。

出错时的反馈通道也给了：`::note` 写的是 "If you encounter any issues, please report them to the Codemod team with **`npx codemod feedback`**"——**报给 Codemod 团队，不是 Nuxt 仓库。**

## 用之前要想清楚的两件事

**"automate many migration steps" 里的 many 不是 all。** 27 项变更里只有 **6 项**给了单项 codemod 命令（下一篇逐个列出对应关系），**其余 21 项只能手改**。所以跑完 recipe 不等于迁移完成，仍然要按上一节那张地图去核对每一项。

**recipe 会改你的代码。** 文档给的承诺只是"可以取消勾选"，**没有说改完之后要不要复核**；考虑到 27 项里包含目录结构这种 Significant 级别的变更，跑之前把这些内容纳入版本控制是常识性前提——**注意这一步文档没有写，是操作建议而不是官方要求。**

## 自查

- 三类命令面：**升版本**（`nuxt upgrade` 五条写法）、**试下一大版本**（配置里的 `future.compatibilityVersion: 5`，**Nuxt 4.2+**）、**自动改代码**（`codemod@0.18.7 nuxt/4/migration-recipe`）。
- 钉版本的原因是 **codemod/codemod #1710**；执行是**顺序 ＋ 可取消勾选**；单项可独立跑；完整清单在 Codemod Registry；反馈走 `npx codemod feedback`。
- yarn 用 `dlx`、bun／deno 用 `x`；nightly 渠道是另一套用法，不在升级路径上。
- **只有 6 项变更有 codemod**，recipe 跑完仍需按 27 项逐条核对。

## 下一篇

下一篇 602 把这六项 codemod 与它们各自对应的变更配好对：`nuxt/4/file-structure`、`nuxt/4/default-data-error-value`、`nuxt/4/deprecated-dedupe-value`、`nuxt/4/shallow-function-reactivity`、`nuxt/4/absolute-watch-path`、`nuxt/4/template-compilation-changes`——顺带数出**另外 21 项变更没有自动化命令**，以及六条提示句里措辞最弱的那一条（"If you need to, you can automate this step…"）。

## 本文依据

- https://nuxt.com/docs/4.x/getting-started/upgrade（"use the `nuxt upgrade` command" 句与五条命令、Nightly Release Channel 一句、`nuxt@^4.0.0` 五条安装写法、"From Nuxt 4.2+…" 与十九项清单及末尾 `::note`、Migrating Using Codemods 整节：与 Codemod 团队合作的开场句、pinned 版本注释与 issue #1710、五条 recipe 写法、`::note` 的 `npx codemod feedback`、Codemod Registry 链接、"execute all codemods in sequence, with the option to deselect…" 与 "can be executed independently" 两句）
