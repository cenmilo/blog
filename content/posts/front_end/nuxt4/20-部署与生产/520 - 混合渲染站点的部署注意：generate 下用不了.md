---
url: /posts/faa42bafabed0c2da324d48c40022f98/
title: 520 - 混合渲染站点的部署注意：generate 下用不了
date: 2027-04-03T10:00:00+08:00
updated: 2027-04-03T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：混合渲染解决的是"一个站点里不同路由要不同处理"：内容页构建时渲好、后台只要客户端渲染、商品页缓存起来后台再刷新。实现手段是 `routeRules`，官方一共列了九个可用属性。**部署侧最关键的是那条 `::important`：用 `nuxt generate` 时混合渲染不可用**——因为这条命令产出的就是纯静态文件，没有服务器去执行这些规则。另外两条要知道：能映射时路由规则会尽量交给部署平台的原生机制（官方当前点名 Netlify 与 Vercel），以及 `ssr: false` 覆盖的路由其页面组件会从服务端 bundle 里排除，但**这只是构建期优化，不改变服务器发给浏览器的内容**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 混合渲染
  - 路由规则
  - 部署
  - 缓存
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

博客内容页希望构建时就渲好，`/admin` 那套后台只想在浏览器里跑，商品列表页希望缓存一小时。这三件事用同一份 `nuxt.config` 表达，就是混合渲染。

## 官方怎么定义它

Hybrid Rendering 那节第一句：**"Hybrid rendering allows different caching rules per route using Route Rules and decides how the server should respond to a new request on a given URL."**

第二句是它的动机，值得照读：**"Previously every route/page of a Nuxt application and server must use the same rendering mode, universal or client-side. In various cases, some pages could be generated at build time, while others should be client-side rendered."**，官方给的正是那个混合场景的例子——内容站带一个后台：**"think of a content website with an admin section. Every content page should be primarily static and generated once, but the admin section requires registration and behaves more like a dynamic application."**

执行机制一句：**"Nuxt server will automatically register corresponding middleware and wrap routes with cache handlers using Nitro caching layer."** 也就是说规则不是给你看的文档，而是被翻译成了中间件加缓存包装，**由 Nuxt 服务器执行**。这句话已经把"部署形态"定下来了：要有服务器。

## 官方那份 routeRules 示例，逐行

```ts twoslash [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    // Homepage pre-rendered at build time
    '/': { prerender: true },
    // Products page generated on demand, revalidates in background, cached until API response changes
    '/products': { swr: true },
    // Product pages generated on demand, revalidates in background, cached for 1 hour (3600 seconds)
    '/products/**': { swr: 3600 },
    // Blog posts page generated on demand, revalidates in background, cached on CDN for 1 hour (3600 seconds)
    '/blog': { isr: 3600 },
    // Blog post page generated on demand once until next deployment, cached on CDN
    '/blog/**': { isr: true },
    // Admin dashboard renders only on client-side
    '/admin/**': { ssr: false },
    // Add cors headers on API routes
    '/api/**': { cors: true },
    // Redirects legacy urls
    '/old-page': { redirect: '/new-page' },
  },
})
```

八条规则各对应一种部署行为：

| 规则 | 官方注释里的含义 | 谁来执行 |
| --- | --- | --- |
| `'/': { prerender: true }` | 首页在构建期预渲染 | 构建过程 |
| `'/products': { swr: true }` | 按需生成、后台再验证、缓存到 API 响应变化 | 服务器 |
| `'/products/**': { swr: 3600 }` | 同上，缓存一小时（3600 秒） | 服务器 |
| `'/blog': { isr: 3600 }` | 按需生成、后台再验证、缓存在 CDN 上一小时 | CDN＋服务器 |
| `'/blog/**': { isr: true }` | 按需生成一次，直到下次部署，缓存在 CDN | CDN＋服务器 |
| `'/admin/**': { ssr: false }` | 后台只在客户端渲染 | 服务器（不发 HTML） |
| `'/api/**': { cors: true }` | 给 API 路由加 cors 头 | 服务器 |
| `'/old-page': { redirect: '/new-page' }` | 重定向旧地址 | 服务器 |

九个属性的定义那份表（`redirect`、`ssr`、`cors`、`headers`、`swr`、`isr`、`prerender`、`noScripts`、`appMiddleware`）我在讲两种缓存策略那篇逐条给原文；这一篇只管部署侧的三条硬结论。

## 一、generate 下用不了

这是最容易被忽略的一条，写在 `::important` 里：

> Note that Hybrid Rendering is not available when using `nuxt generate`.

原因不必猜，两节合起来就是：**`nuxt generate` 产出的是纯静态文件，而 route rules 里的 `swr`/`isr`/`ssr:false`/`redirect` 要靠"服务器自动注册中间件与缓存包装"来执行**——没有服务器，就没有执行者。

所以选型顺序是这样：**先决定生产形态有没有进程在跑**。有服务器才谈得上混合渲染；只有文件服务，可选项就退回到构建期预渲染，加上线上流量的差别处理（`routeRules` 里的 `prerender` 仍然有效，因为那是构建期语义）。

## 二、能交给平台的会交给平台

混合渲染那节还有一句：**"Whenever possible, route rules will be automatically applied to the deployment platform's native rules for optimal performances (Netlify and Vercel are currently supported)."**

这条对部署的实际意义有两层：**在这两个平台上，规则不只在你的应用里执行，还会尽量翻译成平台自己的路由/缓存机制**（这也是 `isr` 能"缓存在 CDN"的前提）；反过来说，**官方点名的支持范围当前只有这两家**，其他平台上同样的规则由 Nitro 缓存层执行，能不能进 CDN 缓存取决于你前面有什么。

## 三、`ssr: false` 影响的是产物，不是响应

这一节很容易被读反，官方把它单独列成 Server Bundle Size with `ssr: false`。规则是：**被 `ssr: false` 覆盖的路由只在浏览器渲染，所以 Nuxt 把它的页面组件从服务端 bundle 里排除**；并且这个排除适用于"覆盖该页面所有可达路径的规则能在构建期解析出来"的情况，包括 `/products/**` 这类规则下的动态路由（官方举 `pages/products/[id].vue`）。

反过来，三种情况下页面**仍留在服务端 bundle**，官方逐条列了：

```ts
'/admin/**': { ssr: false },  // 更具体的规则在下面重新开了 SSR
'/admin/report': { ssr: true },
```

- 客户端区域内有更具体的规则重新启用 SSR（如上）
- 该页面有 alias，或用绝对路径声明的子路由落在客户端区域之外
- 该页面是父壳，而它渲染的子页面仍然是服务端渲染的

最后那句限定要一起读：**"This is a build-time optimization only; it does not change what the server sends to the browser."** 也就是说，**别指望这条能改变任何响应内容**——它省的是产物体积，不是行为差异。排查"这个页面怎么还是服务端渲的"时，上面三条就是候选原因，而不是"优化没生效"。

## 还有一条相关的边界：ESR 是部署目标

顺带把边缘渲染的位置说清楚，因为它常被当成第四种渲染模式。官方那句很直接：**"Note that ESR is more a deployment target than an actual rendering mode."**

能用到 ESR 的平台官方列了三家与各自前提：Cloudflare Pages（用 git 集成与 `nuxt build` 零配置）、Vercel Cloud（`nuxt build` 加 `NITRO_PRESET=vercel-edge`）、Netlify Edge Functions（`nuxt build` 加 `NITRO_PRESET=netlify-edge`）。末尾一句正是本篇主题的延伸：**混合渲染可以和边缘渲染一起用，靠的就是路由规则。**

## 自查

- 混合渲染＝按路由给不同规则，执行者是"Nuxt 服务器自动注册中间件并用 Nitro 缓存层包装"，所以前提是有服务器。
- **`nuxt generate` 下混合渲染不可用**（`::important` 原句），纯静态形态只剩构建期语义可用。
- 规则会尽量映射到平台原生机制，官方当前只点名 Netlify 与 Vercel。
- `ssr: false` 让页面组件从服务端 bundle 排除，但三种情况会保留（更具体规则重开 SSR、alias/绝对路径子页落在区域外、父壳仍渲服务端子页）；官方明说这只是构建期优化，不改变服务器发出的内容。
- ESR 被官方定性为"部署目标而非渲染模式"，三个平台各有预设要求，且它能与混合渲染叠加。

## 下一篇

下一篇 521 逐条拆 `swr` 与 `isr`：TTL 语义、`true` 时加的是哪个头、`node-server` 预设能缓整份响应这句话怎么用，以及 `_payload.json` 在这两种路由上是什么时候生成的。

## 本文依据

- https://nuxt.com/docs/4.x/guide/concepts/rendering（Hybrid Rendering 的定义句、"previously every route/page must use the same rendering mode"与内容站加后台的例子、服务器自动注册中间件并包装缓存的机制句、那份八条 `routeRules` 示例及其逐行注释、Route Rules 九属性清单、`isr`/`swr` 也产出 `_payload.json` 的 `::note`、Server Bundle Size with `ssr: false` 整节（排除条件、三条保留情况、"build-time optimization only"）、路由规则尽量映射到平台原生规则且当前支持 Netlify 与 Vercel、`nuxt generate` 下不可用的 `::important`、Vercel ISR 示例卡；Edge-Side Rendering 一节："more a deployment target than an actual rendering mode"、三个平台与预设、可与混合渲染叠加）
- https://nuxt.com/docs/4.x/getting-started/prerendering（`nuxt generate` 产出静态文件的机制与爬虫，用于说明为何没有执行者）
