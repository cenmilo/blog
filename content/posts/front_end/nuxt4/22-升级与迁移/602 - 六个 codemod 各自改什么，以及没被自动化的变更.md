---
url: /posts/13ff338fab1ceaf56fdbc4a1b97c96db/
title: 六个 codemod 各自改什么，以及没被自动化的变更
date: 2027-04-30T11:00:00+08:00
updated: 2027-04-30T11:00:00+08:00
author: cmdragon

summary:
  摘要：升级指南的 27 项变更里，**只有 6 项给了单项 codemod 命令**，其余 **21 项只能手改**。六条与它们对应的变更是：`nuxt/4/file-structure`（New Directory Structure）、`nuxt/4/default-data-error-value`（`useAsyncData`／`useFetch` 的默认 `data` 与 `error` 值）、`nuxt/4/deprecated-dedupe-value`（`refresh` 的 `dedupe` 布尔值被移除）、`nuxt/4/shallow-function-reactivity`（浅层数据响应）、`nuxt/4/absolute-watch-path`（`builder:watch` 里的绝对路径）、`nuxt/4/template-compilation-changes`（模板编译变更）。三处细节要认出来：**六条都用 `codemod@latest` 而那条 recipe 用的是钉死的 `codemod@0.18.7`**；**只有目录结构那条写的是 "automate this migration"，其余写 "this step"**；**shallow-function-reactivity 前面多了 "If you need to,"**——它是六条里唯一被加上前提的。

categories:
  - 前端开发

tags:
  - nuxt4
  - codemod
  - 迁移自动化
  - 目录结构变更
  - 升级迁移
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

跑完那条 recipe，改了一堆文件，然后你以为迁移结束了——但这一页 27 项变更里只有 6 项有自动化命令，剩下 21 项得自己找。

## 六条命令与它们的对应关系

每条都在对应变更小节的末尾，包在一个 `::tip` 块里：

| codemod | 对应变更小节 | 提示句原文 |
| --- | --- | --- |
| `nuxt/4/file-structure` | New Directory Structure | "You can **automate this migration** by running `npx codemod@latest nuxt/4/file-structure`" |
| `nuxt/4/default-data-error-value` | Default `data` and `error` values in `useAsyncData` and `useFetch` | "You can automate this step by running …" |
| `nuxt/4/deprecated-dedupe-value` | Removal of deprecated `boolean` values for `dedupe` option when calling `refresh` | "You can automate this step by running …" |
| `nuxt/4/shallow-function-reactivity` | Shallow Data Reactivity in `useAsyncData` and `useFetch` | "**If you need to,** you can automate this step by running …" |
| `nuxt/4/absolute-watch-path` | Absolute Watch Paths in `builder:watch` | "You can automate this step by running …" |
| `nuxt/4/template-compilation-changes` | Template Compilation Changes | "You can automate this step by running …" |

三处句式差异不是排版问题，是官方给的强度差别：

- **只有 `file-structure` 用 "this migration"**——它改的是目录结构这种整项目范围的事，其余五条改的是"某个具体写法"。
- **`shallow-function-reactivity` 前面挂着 "If you need to"**——六条里唯一被加前提的一条，意思是**这项自动化不是人人该跑**，要不要改取决于你有没有依赖旧的重层响应行为。
- **六条一律写 `codemod@latest`，而那条总 recipe 写的是钉死的 `codemod@0.18.7`**（因为 codemod/codemod **#1710**）。**同一页里两种版本策略并存，文档没有解释为什么单项不钉版本。** 稳妥做法是先按页面上的原样跑，遇到问题再换成钉版本的形式。

## 六条命令汇总

下面这组是把分散在六个 `::tip` 块里的命令抄在一起，方便对照（原文每条都写在各自变更小节的末尾）：

```bash
npx codemod@latest nuxt/4/file-structure
npx codemod@latest nuxt/4/default-data-error-value
npx codemod@latest nuxt/4/deprecated-dedupe-value
npx codemod@latest nuxt/4/shallow-function-reactivity
npx codemod@latest nuxt/4/absolute-watch-path
npx codemod@latest nuxt/4/template-compilation-changes
```

## 有 codemod 的那一项到底改什么

六条里唯一影响整个项目布局的是目录结构，把它的记载完整读一遍是值得的，因为它同时是**唯一标明 Significant 的变更**。

**迁移步骤里文档点到的东西**（照原文）：

1. 把 `nuxt.config.ts`、`content/`、`layers/`、`modules/`、`public/`、`shared/`、`server/` 这些**留在项目根目录**，不要搬进 `app/`。
2. 第三方配置要跟着改，例如 `tailwindcss` 或 `eslint` 的配置（`@nuxtjs/tailwindcss` 会自动配好）。

**"不迁移也行"这句同样重要**：紧接着的正文写着 **migration is *not required***，如果你要保留原有目录结构，Nuxt 会自动检测；**但有一个例外——你已经设了自定义 `srcDir`**。这种情况下 `modules/`、`public/`、`shared/`、`server/` 会变成**从 `rootDir` 解析而不是从你的 `srcDir` 解析**，官方给的对策是显式设 `dir.modules`、`dir.public` 与 `serverDir`。

这一条对老项目最实用：**如果你设过 `srcDir`，升级后这些目录的解析基准会变**，表现通常是"模块目录突然不生效"。

另外这一项还配了 RFC 链接（nuxt/nuxt **#26444**）。

## 另外 21 项只能手改

27 减 6 等于 **21**。没自动化命令的那批里，最容易在业务代码里显形的有这几组：

- **数据获取族**：Singleton Data Fetching Layer、Respect defaults when clearing `data`、Alignment of `pending` value、Key Change Behavior、Parsed `error.data`——四条关于 `useAsyncData`／`useFetch` 的行为变更，**其中只有两条有 codemod**。
- **状态族**：Respect defaults when clearing `useState`。
- **构建器族**：Corrected Module Loading Order in Layers、Deduplication of Route Metadata、Shared Prerender Data、Absolute Watch Paths 的**上游写法**（`builder:watch` 里 `relative(srcDir, resolve(srcDir, path))` 那类改动，codemod 只覆盖路径本身，钩子逻辑还得自己看）。
- **类型与配置族**：Default TypeScript Configuration Changes、TypeScript Configuration Splitting、Removal of Top-Level `generate` Configuration、Removal of Experimental Features。
- **组件与模板族**：Normalized Component Names、Normalized Page Component Names、Directory index scanning、More Granular Inline Styles、New DOM Location for SPA Loading Screen、Scan Page Meta After Resolution。

判断标准很简单：**这一节末尾有没有那个 `::tip`。** 没有就是手改，按它自己的 Migration Steps 逐条做。

## 自查

- **只有 6 项变更有 codemod**，21 项没有；认可是不是自动化的看该节末尾有没有 `::tip` 命令。
- 六条一律 `codemod@latest`，总 recipe 是 `codemod@0.18.7`（因 #1710）——**两种版本策略并存，文档未解释**。
- `file-structure` 是唯一写 "automate this migration" 的一条；`shallow-function-reactivity` 是唯一带 "If you need to," 前提的一条。
- 目录结构这一项 **migration is not required**，但**已有自定义 `srcDir` 是例外**：`modules/`、`public/`、`shared/`、`server/` 改为从 `rootDir` 解析，对策是设 `dir.modules`、`dir.public`、`serverDir`。
- 第三方配置（`tailwind`、`eslint`）不会由 codemod 代改，文档明写要自己更新。

## 下一篇

下一篇 603 开始逐条读这些变更的作用与迁移步骤，先从"升级时最容易碰的两个键"入手：**`compatibilityDate` 与 `future.compatibilityVersion` 在升级过程中各自扮演什么角色**——前者锁的是 Nitro／Nuxt Image／其他模块的 presets 默认行为，后者一次改十九项 Nuxt 4 上的默认值，而**这两套机制的边界文档没有对照着讲过**。

## 本文依据

- https://nuxt.com/docs/4.x/getting-started/upgrade（六个 `::tip` codemod 提示块及其所在小节、"You can automate this migration by running…" 与 "You can automate this step by running…" 与 "If you need to, you can automate this step…" 三种句式、`codemod@latest` 与 recipe 里 `codemod@0.18.7`（含 #1710 注释）的版本差异、New Directory Structure 小节的迁移步骤 1-4 与 "migration is *not required*" 整段（自定义 `srcDir` 例外、`dir.modules`／`dir.public`／`serverDir` 对策、"raise an issue"）、RFC #26444 链接、27 个变更小节标题清单）
