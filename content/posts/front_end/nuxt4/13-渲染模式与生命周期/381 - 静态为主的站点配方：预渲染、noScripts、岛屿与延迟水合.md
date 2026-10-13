---
url: /posts/25ce752c40cd9195989d540dd8632569/
title: 381 - 静态为主的站点配方：预渲染、noScripts、岛屿与延迟水合
date: 2027-02-16T09:00:00+08:00
updated: 2027-02-16T09:00:00+08:00
author: cmdragon

summary:
  摘要：一个博客或营销站 95% 是死文字和图片，Nuxt 默认却对整页做水合——那部分 JS 纯属开销，拖慢交互、抢带宽和 CPU、还压移动端 PageSpeed。官方给了一份把 JS 砍到接近零的配方，四件工具拼起来：① 内容路由构建期 `prerender`；② 用 `noScripts` 路由规则把这些页的脚本去掉（发纯 HTML+CSS）；③ 需要服务端渲染但不交互的部件用 `.server.vue` 岛屿；④ 少数要交互的路由保留脚本、再用延迟水合按可见/空闲/永不控制水合时机。配一条要注意：`noScripts` 页完全不水合，里面的交互式岛屿（`nuxt-client` 标记或可交互插槽）失效。

categories:
  - 前端开发

tags:
  - nuxt4
  - 静态站
  - noScripts
  - 岛屿
  - 性能
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

前面 367、378、379 分别讲了 routeRules、延迟水合、预渲染。381 把它们拼成一份成品配方——官方 `guide/recipes/mostly-static-sites`，目标一句话：给内容和营销站发接近零 JS，同时保留几块交互。

## 先说为什么默认配置在浪费

配方开头点破问题：

> By default, Nuxt server-renders a page and then hydrates all of it. For a page that is 95% static text and images, that hydration JavaScript is mostly overhead: it delays interactivity and competes with your content for bandwidth and CPU, which can hurt metrics like mobile PageSpeed scores.

一篇 95% 是死文字和图片的页面，把整页水合出来的 JS 基本是纯开销。官方给了四件工具来"只在需要的地方付这个钱"。

## 配方四步

1. **预渲染**内容路由（379）。
2. 用 **`noScripts`** 路由规则把这些路由的脚本去掉（353 features、367）。
3. 需要服务端渲染但不需要交互的部件，用**服务端组件（岛屿）**。
4. 保留脚本的路由上用**延迟水合**（378），让交互只在需要时才付费。

## 第一步：预渲染 + 去脚本

对完全没客户端交互的路由，把 `prerender` 和 `noScripts` 叠在 routeRules 里：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    '/blog/**': { prerender: true, noScripts: true },
    '/about': { prerender: true, noScripts: true },
  },
})
```

这些路由构建期渲成纯 HTML，且渲出的页**省掉 Nuxt 入口脚本、import map、payload 脚本和 JS 资源提示，CSS 照常包含**——结果就是 JavaScript 成本接近零。

`noScripts` 的代价官方用 `::important` 说死：

> A `noScripts` page does not hydrate. Nothing on it is interactive: no `@click` handlers, no `<NuxtLink>` prefetching (links still work as plain `<a>` tags), no client-side navigation away from the page. Only use it on routes where that is what you want.

`noScripts` 页**完全不水合**：没有 `@click`、没有 `<NuxtLink>` 预取（链接退化成普通 `<a>`）、离开本页没有客户端导航。只在你确实要这种效果的路由上用。

## 去脚本路由的导航与预取

因为 `noScripts` 页没有客户端路由，进出它都是整页加载。官方说 Nuxt 两头都替你处理：脚本路由 `navigateTo`/`<NuxtLink>`/`router.push` 到一个 `noScripts` 路由会触发文档加载；且这些页的页面组件在规则构建期可解析时会**从客户端包体里剔除**。脚本页和无脚本页都会发一个声明式的 speculation rules 标签，让支持的浏览器在用户点之前就预取/预渲染目标——它不跑 JS、作用域限定在你的页面路由，所以像 `/logout` 这种服务端路由不会被乱抓。开实验 view transitions 还能让这种整页导航带动画而非闪白。

## 第二步：岛屿放服务端渲染的部件

有些静态块带真逻辑（渲染 markdown、代码高亮、CMS 内容），用**服务端组件**——在服务端渲染、依赖永不进客户端包：

```bash [Directory Structure]
-| app/
---| components/
-----| HighlightedMarkdown.server.vue
```

`.server.vue` 后缀的组件在页面里当普通组件用即可。这种静态岛屿在 `noScripts` 路由上工作正常，因为 HTML 渲染时就嵌进去了。

`::warning` 那条边界很关键：

> Fully interactive widgets inside islands (components marked with `nuxt-client`, or interactive island slots) do **not** work on a `noScripts` route. Island teleports are relocated into place by a small inline script before hydration; with scripts disabled, that relocation cannot run and nothing hydrates. If a page needs even one interactive widget, keep scripts on that route and use lazy hydration instead.

岛屿里**完全交互**的部件（`nuxt-client` 标记的、或可交互岛屿插槽）在 `noScripts` 路由上失效——岛屿靠水合前那段内联脚本把内容搬到位，脚本被禁就搬不了、什么也不水合。只要页面需要一个交互部件，就在该路由保留脚本、改用延迟水合。

## 第三步：其余用延迟水合

要一点交互、又不想整页水合的路由，保留脚本、用延迟水合控制何时水合：

```vue [app/pages/index.vue]
<template>
  <div>
    <HeroSection />
    <LazyTestimonials hydrate-never />
    <LazyNewsletterSignup hydrate-on-visible />
    <LazyCookieBanner hydrate-on-idle />
  </div>
</template>
```

`hydrate-never` 对内容站特别有用：HTML 服务端渲染，但永不水合，JS 永不执行（官方补一句：不过 prop 变化仍会触发水合）。这些策略的完整清单在 378。

## 成品长这样

一个典型以静态为主的站点最后配成：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    '/': { prerender: true, noScripts: true },
    '/blog/**': { prerender: true, noScripts: true },
    '/contact': { prerender: true },
  },
})
```

内容路由发纯 HTML+CSS、导航到它们是 speculation 预取的文档加载；`.server.vue` 岛屿在到处承担服务端渲染部件、把重依赖挡在客户端包外；少数交互路由保留脚本，用 `hydrate-on-visible`/`hydrate-on-idle`/`hydrate-never` 限制浏览器开局干多少活。

## 自查

- 默认整页水合对死内容页是浪费；配方用四件工具按需在个别路由付费。
- 内容路由：`prerender: true` + `noScripts: true` 发纯 HTML；`noScripts` 页完全不水合、无交互、无客户端导航、链接退化成 `<a>`。
- 服务端渲染的部件用 `.server.vue` 岛屿；但 `noScripts` 路由上岛屿里的完全交互部件失效。
- 要交互就保留脚本、用延迟水合（`hydrate-never`/`hydrate-on-visible`/`hydrate-on-idle`）控时机。

## 下一篇

下一篇 382 收 `nuxt generate` 与静态托管的适配点：`ssr: true` 的 SSG 与 `ssr: false` 的纯 SPA 两条路各自产出什么、200/404 回退页怎么在主机上配、以及预渲染下没有服务端端点这条限制。

## 本文依据

- mostly-static 问题陈述（95% 静态页的水合 JS 是开销、伤 mobile PageSpeed）、四步配方、`prerender`+`noScripts` 组合及其省掉/保留内容、`noScripts` 不水合的 `::important`、脚本/无脚本路由的整页加载与 speculation rules 预取、`.server.vue` 岛屿与"完全交互岛屿部件在 noScripts 失效"的 `::warning`、延迟水合 `hydrate-never`/`hydrate-on-visible`/`hydrate-on-idle`（prop 变化仍触发水合）、成品 routeRules 示例（`guide/recipes/mostly-static-sites`；noScripts 见 353、routeRules 见 367、延迟水合见 378）。
