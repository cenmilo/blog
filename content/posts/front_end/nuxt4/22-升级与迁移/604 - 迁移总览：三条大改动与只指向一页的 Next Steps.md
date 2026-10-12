---
url: /posts/5d37daaaf5543c0164b11a9a0a2147fc/
title: 迁移总览：三条大改动与只指向一页的 Next Steps
date: 2027-05-01T10:00:00+08:00
updated: 2027-05-01T10:00:00+08:00
author: cmdragon

summary:
  摘要：`migration/overview` 是迁移文档十一页里的第一页，但它**只有三十来行**：一句定性（"Nuxt 3 is a complete rewrite of Nuxt 2, and also based on a new set of underlying technologies."）、**三条 significant changes**（Vue 2→Vue 3 且默认 Composition API 与 `script setup`；webpack 4＋Babel→Vite 或 webpack 5＋esbuild；运行时 Nuxt 依赖→用 nitropack 编译出的最小独立服务端）、一条 "This migration guide is **under progress**" 的 `::note`、一条指向 Bridge 的 `::tip`，以及一个 **Next Steps 里只放了一个链接**（configuration）。**它不是变更总表，也没有逐项索引**——想靠这一页找到所有迁移条目是找不到的，其余十页都不在它的链接里。

categories:
  - 前端开发

tags:
  - nuxt4
  - 迁移文档
  - 总览
  - Nuxt2到Nuxt3
  - 升级迁移
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

从旧项目升上来，你想先要一张"总共要改哪些地方"的清单，于是打开迁移文档的第一页。这一页比你预期的短得多——它没有清单，只有三条大方向和一个链接。

## 这一页的全部内容

页首定性句：**"Nuxt 3 is a complete rewrite of Nuxt 2, and also based on a new set of underlying technologies."**

正文第一段是范围与预期：

> There are significant changes when migrating a Nuxt 2 app to Nuxt 3, although you can expect migration to become more straightforward as we move toward a stable release.

紧跟一条 `::note`：**"This migration guide is under progress to align with the development of Nuxt 3."**

然后是"三条 significant changes"，编号原文：

1. Moving from Vue 2 to Vue 3, including defaulting to the Composition API and script setup.
2. Moving from webpack 4 and Babel to Vite or webpack 5 and esbuild.
3. Moving from a runtime Nuxt dependency to a minimal, standalone server compiled with nitropack.

最后一条 `::tip` 给了另一条路：**如果你需要留在 Nuxt 2、又想用上 Nuxt 3 的特性，去看 Bridge**（指向 `bridge/overview`）。

再往下就是这一页唯一的小节 **Next Steps**，里面**只有一个条目**：configuration 那一页的链接。

## 用它当总表会扑空的三个原因

**第一，它不索引其余页面。** 迁移文档实际有 **11 页**：configuration、auto-imports、meta、pages-and-layouts、component-options、plugins-and-middleware、runtime-config、bundling、server、module-authors、overview 本身。而这一页的 Next Steps **只列了 configuration 一条**，另外九页都不在这页里——**要逐项定位得直接看站点目录或搜索。**

**第二，它的口径是 2→3，不是 3→4。** 全页措辞都是 "migrating a Nuxt 2 app to Nuxt 3"，`::note` 也明说指南是为对齐 Nuxt 3 的开发而"仍在写"。**从 3 升到 4 的内容在另一页**（入门那侧的升级指南，带 27 个变更小节与影响级别标注），两者不是同一份文档。

**第三，"三条大改动"是概述，不是可执行项。** 它没有告诉你改哪个文件、哪个键——这三条在 4.x 文档里各自有对应的详细页：第二条的构建器部分落在配置参考的 `## builder`（三种写法，webpack／rspack 需自己装）与 `## esbuild`（那份共享默认值），第三条的服务端部分落在 `guide/concepts/server-engine`。**这三条真正的用处是让你知道去哪一页找细节。**

## 三条各自在 4.x 里的落点

| 概述里的说法 | 细节在哪 |
| --- | --- |
| Vue 2 → Vue 3，默认 Composition API 与 `script setup` | 组件选项那一页（`migration/component-options`，十个选项小节各自处理） |
| webpack 4＋Babel → Vite 或 webpack 5＋esbuild | `api/nuxt-config` 的 `## builder`／`## esbuild`／`## vite`／`## webpack`；构建工具那一页（`migration/bundling`） |
| 运行时 Nuxt 依赖 → 用 nitropack 编译的最小独立服务端 | `guide/concepts/server-engine`；配置参考的 `## nitro` 与 `## server`（服务端构建器） |

这三行只是"去哪找"的指路，具体改动仍以各详细页为准——**这一页本身没给任何步骤。**

## Bridge 那条 tip 的实际含义

`::tip` 是给"上不了 3 但想要新特性"的项目留的路：**Nuxt Bridge 是一个兼容层，让你在 Nuxt 2 里使用 Nuxt 3+ 的特性，采用 opt-in 机制**（这句在升级指南的 Nuxt 2 相关小节里也重复了一遍）。

也就是说，摆在面前的路线选择是三条，而不是两条：直接迁、留在 2 走 Bridge、或者按 2→3 的文档逐步迁。**这一页只提了后两条，第一条的详细说明在 configuration 那页里**（它给的是一步一步的编号操作）。

## 自查

- 这一页只有：一句定性、一句范围说明、**一条 "guide is under progress" 的 `::note`**、**三条大改动**、**一条 Bridge 的 `::tip`**、**Next Steps 一个链接**。
- **它不是变更总表**：迁移文档共 11 页，此页只链接其中 1 页。
- **它的口径是 Nuxt 2 → Nuxt 3**；3 → 4 的逐项变更在入门侧的升级指南那一页（27 个变更小节）。
- 三条大改动分别指向 component-options、builder/esbuild/vite/webpack 与 bundling、server-engine 三处细节。
- 留在 2 又想要新特性 → Bridge（兼容层、opt-in）。

## 下一篇

下一篇 605 读 configuration 那一页——**这才是迁移文档里真正有步骤的一页**：六个章节、九步编号操作（`defineNuxtConfig`、`router.extendRoutes`→`pages:extend`、`routeNameSplitter` 的手动复刻、`buildModules`→`modules` 等），外加三条禁用规则（不用 `require`／`module.exports`、异步配置已废弃、别在配置里直接导入 `.env`）。

## 本文依据

- https://nuxt.com/docs/4.x/migration/overview（页首定性句、"There are significant changes…" 段、`::note` 的 under progress 句、三条编号改动原文、`::tip` 的 Bridge 指向、Next Steps 仅一个链接的事实与该页全部 `##` 小节）
- https://nuxt.com/docs/4.x/getting-started/upgrade（"Nuxt Bridge is a compatibility layer that allows you to use Nuxt 3+ features in Nuxt 2 with an opt-in mechanism." 那句；该页另含 27 个 3→4 变更小节）
- https://nuxt.com/docs/4.x/migration/configuration 与迁移目录下其余九页（configuration、auto-imports、meta、pages-and-layouts、component-options、plugins-and-middleware、runtime-config、bundling、server、module-authors 加上 overview 本身共 11 页，可在这个文档站点的 sitemap 里逐条核对）
