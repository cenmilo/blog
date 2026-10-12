---
url: /posts/46bafd6c48f20d476b887280b9fb7a0f/
title: router：两个子项、hashMode 的两条后果与写了两遍的默认值
date: 2027-04-23T09:00:00+08:00
updated: 2027-04-23T09:00:00+08:00
author: cmdragon

summary:
  摘要：`## router` 标题下面**没有正文**，直接是 `### options`——"Additional router options passed to `vue-router`"，加一条 callout 限定 **"Only JSON serializable options should be passed by Nuxt.config"**，要更多控制就去写 `router.options.ts`。这一节自己的子项只有两个：`hashMode`（`boolean`、默认 `false`；开启后 **URL 从不发给服务器、SSR 不支持**）与 `scrollBehaviorType`（`string`、默认 `"auto"`；**可选值清单没有给**）。两处格式痕迹值得留意：**这两个子项的默认值都被写了两遍**，其中一处还从 `"auto"` 变成了 `'auto'`。能传哪些 vue-router 选项，答案在自定义路由那篇配方里，**共七项**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 路由选项
  - hash模式
  - 滚动行为
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

要给 `vue-router` 传选项——比如让链接的激活类名换个写法、或者让锚点跳转平滑滚动——你在配置参考里找 `router`。找到的这一节标题下是空的，第一个东西就是 `### options`。

## 这一节的形状：定性在子项里

`## router` **没有定性句**，直接进 `### options`，它的原文两句：

> Additional router options passed to `vue-router`. On top of the options for `vue-router`, Nuxt offers additional options to customize the router (see below).

再加一条 callout：

> **Note**: Only JSON serializable options should be passed by Nuxt config. For more control, you can use an `router.options.ts` file.

最后是 See 指向 `vue-router` 的 RouterOptions 接口文档。**`### options` 自己也没有 Type 行、没有 Default 行。**

那句 "Only JSON serializable options" 是这一节最实用的判据：`vue-router` 的构造选项里，**函数和实例类的东西不能写在这里**（比如 `history`，它是个工厂函数；`scrollBehavior` 也是函数）。要传这些就得落到文件那侧。

## 能传哪七个：清单不在这节里

配置参考没有列"哪些选项可以走 `nuxt.config`"。这份清单在自定义路由那篇配方里，**七项**：

- `linkActiveClass`
- `linkExactActiveClass`
- `end`
- `sensitive`
- `strict`
- `hashMode`
- `scrollBehaviorType`

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  router: {
    options: {},
  },
})
```

**注意这七项里只有后两项在配置参考里有条目**——`linkActiveClass`、`linkExactActiveClass`、`end`、`sensitive`、`strict` 五个在配置参考的 `## router` 下**没有对应小节**，它们能传的依据只有配方页那七行。

同页还有一条与升级档位绑着的说明：设了 `future.compatibilityVersion: 5` 时路由**默认区分大小写**（与 Nitro 一致），要把这层退回未区分，就设 `router.options.sensitive` 为 `false`。

## hashMode：两条后果写在同一句里，默认值被写了两遍

原文：

> You can enable hash history in SPA mode. In this mode, router uses a hash character (#) before the actual URL that is internally passed. When enabled, the **URL is never sent to the server** and **SSR is not supported**.

- **Type**: `boolean`
- **Default:** `false`
- 文档紧接着又写了一行 `**Default**: false`

两条后果要连在一起读：**URL 从不发给服务器**（所以服务器端拿不到路径，任何依赖路径的服务端逻辑、预渲染、按路径的路由规则都失去依据），**SSR 不支持**。配方页给的示例也印证它是配着关 SSR 用的：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  ssr: false,
  router: {
    options: {
      hashMode: true,
    },
  },
})
```

还有两个实验开关会被它连带影响，两处都写在实验特性那一页：

- `experimental.serverPathFallback`：原文一句 "**The option has no effect when `hashMode` is enabled.**"
- `experimental.early404`：自动关闭的条件之一就是这个——"The option is disabled automatically in development, **when `hashMode` is enabled**, when a root-level catch-all page … and when a custom `app/router.options` file may modify `routes`"。

## scrollBehaviorType：只有默认值，没有可选值清单

原文一句：

> Customize the scroll behavior for hash links.

- **Type**: `string`
- **Default:** `"auto"`
- 下面又写了一行 `**Default**: 'auto'`（**同一份文档里这里用单引号**）

**这一项没有列出可选值。** 唯一的第二个取值线索在配方页：把这一项设成 `smooth`，然后访问带锚点的地址（例如 `https://example.com/blog/my-article#comments`）时浏览器会平滑滚到那个锚点。加上默认的那个 `auto`，文档里**实际出现过的取值只有这两个**，其他值能不能传、传了会怎样没有记载。

Nuxt 默认的滚动行为在无障碍那一页有交代：新路由滚到顶部、用户后退时恢复上次位置、滚到 hash 目标。那一页接着说，**要平滑滚动或不同的偏移**，就配 `scrollBehaviorType` 或在 `router.options.ts` 里写自己的 `scrollBehavior`，并提醒平滑滚动应当尊重用户的 `prefers-reduced-motion` 设置。

这里要抠一下字眼：**"偏移量"在配置参考这一节没有对应键**——那件事只能靠 `scrollBehavior` 函数，也就是文件那侧，`scrollBehaviorType` 管的是 hash 链接的滚动类型。

## 文件那一侧：放哪儿、能改什么

callout 说的那份文件在别处给了位置和形状。

位置：升级页写着 `dir.app` 这个目录用来找 `router.options.ts` 和 `spa-loading-template.html`，默认是 `<srcDir>/`；实验特性页提到这个文件时用的路径写法是 `app/router.options`。

形状（配方页 "Custom History (advanced)" 那段）：可以覆盖 history 模式——传一个函数，接收 base URL、返回 history 模式；**返回 `null` 或 `undefined` 时 Nuxt 回落到默认 history**：

```ts [router.options.ts]
import type { RouterConfig } from '@nuxt/schema'
import { createMemoryHistory } from 'vue-router'

export default {
  // https://router.vuejs.org/api/interfaces/routeroptions
  history: base => import.meta.client ? createMemoryHistory(base) : null, /* default */
} satisfies RouterConfig
```

那个 `satisfies RouterConfig` 用的是 `@nuxt/schema` 导出的类型——**这就是"要传函数就走文件"这条路的类型出口**。另外从 `early404` 那段还能读到一个附带判据：如果这份默认导出**可能会改 `routes`**，`early404` 会被自动关掉；而"一个不含 `routes` 键的普通对象"仍然算兼容。

## 自查

- `## router` 无定性句；`### options` 也没有 Type/Default，只有两句说明＋一条 callout＋一个 See。
- 走配置文件的前提是**可 JSON 序列化**；配方页列了**七项**可传，其中五个（`linkActiveClass`、`linkExactActiveClass`、`end`、`sensitive`、`strict`）在配置参考里**没有小节**。
- `hashMode`：默认 `false`，开启后 URL 不发给服务器、不支持 SSR；并让 `serverPathFallback` 失效、让 `early404` 自动关闭。
- `scrollBehaviorType`：默认 `"auto"`（**这一项的默认值在文档里出现两次，第二次是单引号**），可选值未列，文档只另给过 `smooth` 一个实例。
- 需要 `history`、`scrollBehavior` 这类函数：写到 `router.options.ts`（位置在 `dir.app`，默认 `<srcDir>/`），类型用 `@nuxt/schema` 的 `RouterConfig`。
- `future.compatibilityVersion: 5` 让路由大小写敏感，退回方法是 `router.options.sensitive: false`。

## 下一篇

下一篇 580 看 `## components`：它有类型行、有一份两元素的默认 `dirs` 表（**其中一项带 `global: true`**），但**一个 `###` 子项都没有**——`dirs` 里能写哪些字段、`pathPrefix` 和 `global` 怎么设，全在组件目录那一页。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## router` 无正文的事实；`### options` 两句与 `::callout` 原文、See RouterOptions；`#### hashMode` 全段与两行默认值；`#### scrollBehaviorType` 一句、`string` 与两行默认值）
- https://nuxt.com/docs/4.x/guide/recipes/custom-routing（七项可序列化选项清单与 `router: { options: {} }` 示例、`sensitive` 与 compatibilityVersion 的 `::note`、Hash Mode (SPA) 段含 `ssr: false` 示例、Scroll Behavior 段与 `smooth` 示例、Custom History 段与 `RouterConfig` 示例及"返回 null/undefined 回落默认"）
- https://nuxt.com/docs/4.x/guide/best-practices/accessibility（默认滚动三行为、`scrollBehaviorType` 或自写 `scrollBehavior`、`prefers-reduced-motion` 提醒）
- https://nuxt.com/docs/4.x/guide/going-further/experimental-features（`serverPathFallback` 那句 "no effect when `hashMode` is enabled"、`early404` 自动关闭的四个条件与 `app/router.options` 的路径写法）
- https://nuxt.com/docs/4.x/getting-started/upgrade（`dir.app` 用于查找 `router.options.ts` 与 `spa-loading-template.html`、默认 `<srcDir>/`）
