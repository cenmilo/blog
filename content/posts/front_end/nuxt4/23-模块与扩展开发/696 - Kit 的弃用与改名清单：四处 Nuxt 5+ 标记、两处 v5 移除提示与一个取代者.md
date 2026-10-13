---
url: /posts/5fc541006ddd1dec97297deef20bd552/
title: 696 - Kit 的弃用与改名清单：四处 Nuxt 5+ 标记、两处 v5 移除提示与一个取代者
date: 2027-06-01T09:00:00+08:00
updated: 2027-06-01T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：把 Kit 二十页里所有写着"已弃用""将被移除""以前叫某个名字"的地方收集到一起，一共这些：`extendViteConfig` 整条被 `::warning` 标为已弃用；`addVitePlugin` 那条警告说 Nuxt 5+ 里用 `server: false`/`client: false` 注册的插件**不会调用它的 `config` 与 `configResolved` 钩子**；`addVitePlugin` 与 `extendViteConfig` 两张选项表里 `server`/`client` 四行都标了 **Deprecated in Nuxt 5+**；`installModule` 被标弃用并指向 `moduleDependencies`；`isNuxt2`、`isNuxt3` 挂着"可能在 @nuxt/kit v5 移除"；`addNitroPlugin` 在 v4.6 之前叫 `addServerPlugin`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 模块开发
  - nuxt-kit
  - 弃用
  - 升级准备
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 先给结论：这些标记集中在两节

翻遍 `api/kit/*` 二十页，带弃用或移除字样的只有四个页面：**builder**（四处）、**compatibility**（两处）、**modules**（一处，而且是"将被移除"的硬话）、**nitro**（一处改名史）。剩下的十六页没有任何弃用标记。

## builder 页：Vite 侧被拆掉的是"按环境挑配置"这条路

**第一处，整条弃用。**`extendViteConfig` 那一节开头挂着 `::warning`：

> 这个钩子现在已经弃用，我们推荐改用一个 Vite 插件、在它的 `config` 钩子里做，或者——如果是环境相关的配置——用 `applyToEnvironment` 钩子。

同一节后面又补了一句方向相同的话：**在 Nuxt 5+ 里做环境相关配置，请改用 `addVitePlugin()`**，并给了带 `applyToEnvironment` 的插件写法。

**第二处，选项表里的两行。**`extendViteConfig` 的 options 有五行（`dev`、`build`、`server`、`client`、`prepend`），其中 `server` 与 `client` 两行各带一句：

> **Deprecated in Nuxt 5+.** 改用 `addVitePlugin()` 搭配 `applyToEnvironment()`。

`dev`、`build`、`prepend` 三行**没有**被标弃用。`prepend` 那行的语义也还在：设为 `true` 时用 `unshift()` 而不是 `push()` 插到数组前面。

**第三处，`addVitePlugin` 自己的警告。**这一节开头的 `::warning` 讲的是行为变化，不是函数消失：

> 在 Nuxt 5+，用 `server: false` 或 `client: false` 选项注册的插件，它们的 `config` 与 `configResolved` 钩子**将不会被调用**。请改用 `applyToEnvironment()` 方法来做环境相关的配置。

**第四处，同一页另一张表。**`addVitePlugin` 的 options 表里 `server`、`client` 两行同样标了 **Deprecated in Nuxt 5+**，只是替代写法只写了 `applyToEnvironment()`。

把四处合起来读，方向只有一个：**环境维度的分流从"注册时的布尔开关"搬到"插件内部的 `applyToEnvironment()`"**。而在 `addVitePlugin` 那一节的示例之后还有一条同族警告，讲的是钩子顺序：**`config` 钩子比 `applyToEnvironment` 更早运行，改的是全局配置；要做环境相关的改动，用 `configEnvironment`。**

## webpack 侧为什么没有这些标记

`extendWebpackConfig` 与 `addWebpackPlugin` 两节的选项表**都没有 Deprecated 字样**（`extendWebpackConfig` 的表同样是 `dev`/`build`/`server`/`client`/`prepend` 五行）。文档给出的理由是机制性的：Nuxt 5 改用共享配置，环境分流由 Vite 侧的 `applyToEnvironment()` 承担，而 webpack 路径没有对应说法。**这一节没有解释 webpack 侧在 Nuxt 5+ 会怎样**，要迁就不了只能自己验证。

另外，跨构建器的路子还在：`addBuildPlugin` 的描述是"`addVitePlugin` 与 `addWebpackPlugin` 的构建器无关版本，如果两者存在就都加上"。

## modules 页：`installModule` 是最重的一处

`installModule` 那一节开头是一个 `::callout{type="warning"}`，原文：

> **已弃用：** 改用 `defineNuxtModule` 里的 `moduleDependencies` 选项。`installModule` 函数将会被移除（或者可能变成非阻塞）。

"will be removed (or may become **non-blocking**)"这句里的两个可能性文档自己没定，但它已经把迁移目标说清了：依赖另一个模块时用 `moduleDependencies` 声明，可以带 `version` 约束与 `overrides`/`defaults` 两类配置合并（那是依赖声明那一节的内容，另一篇逐项拆过）。

## compatibility 页：两处 removal 提示，且其中一处对不上

`isNuxt2` 和 `isNuxt3` 两节各挂着一条 note，**句子完全相同**：

> 改用 `isNuxtMajorVersion(2, nuxt)`。这可能在 @nuxt/kit v5 或未来某个大版本里被移除。

对 `isNuxt2` 来说这句读得通。对 `isNuxt3` 来说不通：那一节讲的是"当前版本是不是 3.x"，替代写法却给了查 2。**文档没有说明这是笔误还是别的意图**，安全做法是这两个都别用，直接按目标主版本调 `isNuxtMajorVersion(major, nuxt?)`。

## nitro 页：一处改名，时间在 v4.6

`addNitroPlugin` 那一节的 note 只有一句：**这在 Nuxt v4.6 之前叫 `addServerPlugin`。**

改名同时带来一个搜索陷阱：搜 `addServerPlugin` 找不到这一节，而旧文章、旧仓库里的代码用的正是这个名字。同一节还有一条与兼容有关的要求（必须在插件文件里显式从 `nitropack/runtime` 导入 `defineNitroPlugin`），那在讲服务端时机的那篇已经展开。

## 一个跨页的口径冲突，值得再记一次

`extendViteConfig` 在这里被整条标为已弃用；但讲构建器那一节把 `extendViteConfig`／`extendWebpackConfig` 列为"模块影响打包的正常通道"，**没有提任何弃用**。同一个函数在两个位置的口径不同，本系列按各页原文分别记录、不替它们做合并推断——这意味着你读构建器那一章时看到的"推荐做法"，在 Kit 页已经变成"已弃用"。

## 迁移时的动手顺序

- 先扫 `extendViteConfig`／`extendWebpackConfig` 的调用：如果传了 `server`/`client`，就是四处标记里最容易被漏的那两行。
- 把"只在客户端生效""只在服务端生效"的插件改成 `addVitePlugin()` 传函数、内部写 `applyToEnvironment(environment => environment.name === 'client')` 这类判断。
- 环境相关的配置改动从 `config` 钩子挪到 `configEnvironment`。
- 用 `installModule` 的地方换成 `moduleDependencies`，并检查 `@nuxt/kit` 是否在 `dependencies`（对象形态的 `handler` 也依赖这一点）。
- 把 `isNuxt2`／`isNuxt3` 换成 `isNuxtMajorVersion`。
- 老代码里的 `addServerPlugin` 改成 `addNitroPlugin`，注意它需要 v4.6 起的 Kit。

下一篇讲图层与模块怎么选：可复用的东西放在哪一层，官方的两节各给了什么依据。

## 下一篇

下一篇 697：图层还是模块：官方两节给出的分工与四条判断

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/builder
- https://nuxt.com/docs/4.x/api/kit/modules
- https://nuxt.com/docs/4.x/api/kit/compatibility
- https://nuxt.com/docs/4.x/api/kit/nitro
