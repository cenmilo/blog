---
url: /posts/c1336773232bdb7cd9ecd309aa2424d1/
title: 服务端类迁移：一句没有运行时 Nuxt 依赖与三步清理
date: 2027-05-04T10:00:00+08:00
updated: 2027-05-04T10:00:00+08:00
author: cmdragon

summary:
  摘要：`migration/server` 是全篇最短的迁移页之一：一句定性加**三步清理**。定性句把代价说清了——**"In a built Nuxt 3 application, there is no runtime Nuxt dependency. That means your site will be highly performant, and ultra-slim. But it also means you can no longer hook into runtime Nuxt server hooks."** 三步是：**删掉 `nuxt.config` 里的 `render` 键**；**`~~/server/api` 与 `~~/server/middleware` 里的文件会自动注册，所以把它们从 `serverMiddleware` 数组里删掉**；**`serverMiddleware` 里剩下的项要改成直接指向文件或 npm 包，而不是内联函数**。三条 read-more 分别指向服务端引擎概念页、`server/` 目录页，以及运行时服务端钩子那一节。

categories:
  - 前端开发

tags:
  - nuxt4
  - 服务端迁移
  - serverMiddleware
  - render 键
  - 升级迁移
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

旧项目里那套 `render: { route: { ... } }` 配置和 `serverMiddleware: [function (req, res) {...}]`，升上来之后没有任何反应——这一页就是解释为什么，并给三步清理。

## 一句定性：少了一个运行时依赖，也少了运行时钩子

> In a built Nuxt 3 application, there is no runtime Nuxt dependency. That means your site will be highly performant, and ultra-slim. But it also means you can no longer hook into runtime Nuxt server hooks.

这句话是三件事：**产物里没有运行时 Nuxt 依赖**（这与 overview 那页列的第三条大改动是同一件事——从"运行时 Nuxt 依赖"变成"用 nitropack 编译出的最小独立服务端"）；**因此更小更快**；**代价是运行时 Nuxt 服务端钩子没了**。

第三条要单独记住：它意味着旧代码里那些"在运行时插一脚"的做法（Nuxt 2 的服务端钩子）不是换个写法的问题，而是**这一层能力在新架构里不由 Nuxt 提供**。想确认现在还有什么钩子可用，得去看钩子那一页的运行时服务端钩子一节（这一页最后一条 read-more 就是它）。

## 三步清理

页面上只有一个 `## Steps`，三步：

1. **Remove the `render` key in your `nuxt.config`.**
2. Any files in `~~/server/api` and `~~/server/middleware` will be automatically registered; **you can remove them from your `serverMiddleware` array**.
3. Update any other items in your `serverMiddleware` array to **point to files or npm packages directly, rather than using inline functions**.

三步的性质各不相同，分开看：

| 步骤 | 动作 | 为什么 |
| --- | --- | --- |
| 1 | 删 `render` 键 | 这一页没有解释旧 `render` 的每个子键去了哪里，只是要求删 |
| 2 | 从 `serverMiddleware` 数组里**删掉目录里已有的那些** | 因为 `~~/server/api` 与 `~~/server/middleware` 的文件**本来就会自动注册**，列进去是重复 |
| 3 | 剩下的项改成**文件或 npm 包** | 内联函数这一种形态不再被接受 |

第 2 步是"去冗余"而不是"改行为"——这与插件那侧的规矩完全同构（`app/plugins/` 目录里的插件也不必再列进配置）。第 3 步才是真改动：**旧项目里最常见的写法恰恰是内联函数**，比如直接在 `nuxt.config` 里写 `serverMiddleware: [(req, res, next) => { ... }]`，这种必须搬成一个文件再引用路径。

第 1 步要提醒一句：这一页**没有说 `render` 下的配置等价物在哪**。旧版用它改渲染路径、静态资源前缀等，新版这些各有去处（例如公开路径前缀走 `app.buildAssetsDir` 与 `app.cdnURL`），但那是另一节的内容，**不能算这一页给的依据**。

## 三条 read-more 各自指向哪

这一页挂了三个链接，顺序是：

- **`guide/concepts/server-engine`**：服务端引擎那一页，解释"没有运行时 Nuxt 依赖"到底指什么。
- **`directory-structure/server`**：`server/` 目录页，`server/api`、`server/middleware`、`server/routes` 的自动注册规则与文件形状都在那里。
- **`guide/going-further/hooks#server-hooks-runtime`**：钩子页里的运行时服务端钩子一节——**上面那句"不能再挂运行时钩子"要配合这一节看**，能挂的是 Nitro 侧那批。

## 自查

- 定性句三件事：无运行时 Nuxt 依赖、更小更快、**运行时 Nuxt 服务端钩子没了**。
- 三步：删 `render` 键；从 `serverMiddleware` 里删掉目录内已有的项；剩下的项改成文件或 npm 包引用。
- 第 2 步是去冗余（目录自动注册），第 3 步才是形态限制（不接受内联函数）。
- 旧 `render` 子键的新去处**这一页没有给**，要另查 `app` 一节与目录页。
- 三个链接分别对应：概念页、`server/` 目录页、运行时服务端钩子一节。

## 下一篇

下一篇 614 读模块作者那一页：它先说明**Bridge 有基于 `@nuxt/kit` 自动包装的向后兼容层，但有时必须走 Bridge 才能跨版本兼容**，并指出**最佳迁移路径其实是按模块开发指南重写**；然后是七个子节——插件不完全向后兼容、Vue 侧要靠 vue-demi 才同时支持 2 和 3、`this.*` 容器访问要换成 kit 工具、先用 `@nuxt/bridge` 试、CommonJS 转 ESM、**没有 `export default` 的注入插件要补一行**、避免运行时模块（这一条里 `buildModules` 的链接锚点其实指错了），最后是可选的 TypeScript。

## 本文依据

- https://nuxt.com/docs/4.x/migration/server（定性段全文（无运行时 Nuxt 依赖／highly performant and ultra-slim／no longer hook into runtime Nuxt server hooks）、`## Steps` 三步原文（删 `render` 键、`~~/server/api` 与 `~~/server/middleware` 自动注册因而可从 `serverMiddleware` 数组删除、其余项改成文件或 npm 包而非内联函数）、三条 read-more 的指向）
- https://nuxt.com/docs/4.x/migration/overview（三条大改动中"从运行时 Nuxt 依赖改为 nitropack 编译的最小独立服务端"一句，用于印证本页定性的来处）
