---
url: /posts/fa7bba73e51dd2b67623f64265fb4042/
title: 451 - 服务端缓存：routeRules 的 swr 与 isr 策略
date: 2027-03-11T10:00:00+08:00
updated: 2027-03-11T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：`routeRules` 按路径组给路由挂规则：改渲染模式或指定缓存策略，Nuxt 服务端会"自动注册对应中间件，并用 Nitro 缓存层把路由包上缓存处理器"。两个缓存属性只差一层落点：`swr` 把响应缓存在服务端或反向代理，TTL 到了先把缓存发出去、同时在后台重生成，`swr: true` 只加 `stale-while-revalidate` 头而不带 max-age；`isr` 行为与 `swr` 相同，多的是能写进支持它的 CDN（官方目前点名 Netlify 与 Vercel），`isr: true` 的内容一直留到下一次部署。三条边界要记住：官方把 `routeRules` 标为实验特性、`nuxt generate` 下混合渲染不可用、带 `swr`/`isr` 的路由会同时生成 `_payload.json`。

categories:
  - 前端开发

tags:
  - nuxt4
  - routeRules
  - SWR
  - ISR
  - 缓存
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

博客文章页每次访问都在服务端重新渲染一遍，同一天同一篇文章渲染三十万次；而后台管理页反倒跟着全站设置走了服务端渲染。

这两件事都指向同一个缺口：**渲染与缓存策略需要按路径给**。官方给的入口是 `routeRules`。

## 它是什么，官方怎么定性

配置页那一条写得简短：

> Global route options applied to matching server routes.
> **Experimental**: This is an feature and API may change in the future.

后面一句把细节推给了 Nitro 的文档。这一句里的"实验特性、API 将来可能变"要当真——它是配置项级别的声明，不是渲染模式那一页的措辞。

概念页给的定位是两句：

> Using route rules you can define rules for a group of nuxt routes, change rendering mode or assign a cache strategy based on route!
>
> Nuxt server will automatically register corresponding middleware and wrap routes with cache handlers using Nitro caching layer.

第二句是这一篇的关键：**你只写规则，中间件与缓存处理器由服务端自动装上**，缓存层归 Nitro。

顺手记一处容易写错的位置：`nitro` 节下面也有一个 `routeRules` 键，但配置页里它**只有类型 `object`、一个字说明都没有**；上面那两句定性都在顶层 `routeRules` 名下。

## 九个属性全表

概念页 "Route Rules" 小节的原话是"The different properties you can use are the following"，一共九条，逐条落地：

| 属性与类型 | 官方说明 |
| --- | --- |
| `redirect: string` | Define server-side redirects. |
| `ssr: boolean` | 用 `ssr: false` 关掉这些区块的服务端渲染，只在浏览器渲染 |
| `cors: boolean` | `cors: true` 自动加 CORS 头；要定制就用 `headers` 覆盖 |
| `headers: object` | 给站点某些区块加指定响应头，例如静态资源 |
| `swr: number \| boolean` | 给服务端响应加缓存头，并缓存在服务端或反向代理，TTL（time to live）可配。Nitro 的 `node-server` 预设能缓存整个响应。TTL 到期后，缓存的响应会被发出，同时页面在后台重生成。用 `true` 时添加 `stale-while-revalidate` 头、不带 MaxAge |
| `isr: number \| boolean` | 行为与 `swr` 相同，差别是能把响应加进 CDN 缓存——在支持它的平台上（目前 Netlify 或 Vercel）。用 `true` 时内容在 CDN 里一直保留到下一次部署 |
| `prerender: boolean` | 在构建期预渲染这些路由，并把它们作为静态资产包含进构建产物 |
| `noScripts: boolean` | 关闭站点某些区块的 Nuxt 脚本与 JS 资源提示渲染 |
| `appMiddleware: string \| string[] \| Record<string, boolean>` | 定义哪些中间件应该、或不应该在**你的 Vue 应用部分**的页面路径上运行（不含 Nitro 路由） |

`swr` 那一格拆成四个可验证的事实：**缓存位置**（服务端或反向代理）、**TTL 可配**、**过期时的顺序**（先发缓存、后台重生成）、**`true` 的形态**（只加 `stale-while-revalidate`，不带 max-age）。

`isr` 与它唯一的差别写在两处：**能不能进 CDN**，以及**支持平台目前只有 Netlify 与 Vercel**。

## 官方示例里每条注释都是语义

```ts [nuxt.config.ts]
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

注释里出现了三组反复使用的措辞，可以直接当判据：**generated on demand**（按需生成，不是构建期）、**revalidates in background**（过期不阻塞，后台补）、**cached until API response changes / for 1 hour / once until next deployment**（三种失效条件）。

最后一条 `'/old-page': { redirect: '/new-page' }` 和 `redirect` 属性那句"Define server-side redirects."放在一起看：这是**服务端**重定向，不是客户端导航拦截。

## 四条边界

**一、`nuxt generate` 用不了混合渲染。** 这条是 `::important` 级的：

> Note that Hybrid Rendering is not available when using [`nuxt generate`].

038 那篇讲过 `nuxt generate` 的真实身份就是 `prerender: true` 的 build——产物里没有服务端，自然没有能装缓存处理器的地方。

**二、规则会尽量映射到平台原生能力。**

> Whenever possible, route rules will be automatically applied to the deployment platform's native rules for optimal performances (Netlify and Vercel are currently supported).

**三、`swr`/`isr` 的路由会一起产出 `_payload.json`。** 一条 `::note`：

> Routes using `isr` or `swr` also generate `_payload.json` files alongside HTML. Client-side navigation loads these cached payloads instead of re-fetching data.

这和 448 讲的 `payloadExtraction` 是连着的：载荷文件能被 CDN 和 HTML 一起缓存，客户端导航直接复用。

**四、缓存类规则与流式互斥。** 450 那条退回清单里第一条就是"`routeRules` setting `noScripts`, `cache`, `isr`, `swr`, `redirect`, or `streaming: false`"——**给路由挂 `swr`/`isr` 等于把它请出流式渲染**。

这里还牵出一个上面九条里没有的名字：`cache`。`early404` 那节写着"A `cache` route rule covering the path advertises its `maxAge` as `Cache-Control` on GET/HEAD misses too"，流式退回清单也列了它，但**概念页那九条属性里没有 `cache`**，它的完整形态要看 Nitro 文档——这一处属于 Nuxt 文档与 Nitro 文档的边界，Nuxt 侧未列全。

## 顺带一条 ssr:false 的构建副作用

这一节容易被忽略，因为它讲的是产物而不是响应：

> A route covered by `ssr: false` is only ever rendered in the browser, so Nuxt excludes its page component from the server bundle.

条件是"覆盖该页面所有可达路径的规则都能在构建期确定"，官方举例连 `pages/products/[id].vue` 配 `/products/**` 这种动态路由也算。三种情况会把这个页面**留在服务端包里**：

- 更具体的规则在下面重新开了 SSR（`'/admin/**': { ssr: false }` 配 `'/admin/report': { ssr: true }`）
- 页面有 alias，或用绝对路径声明的子路由落在纯客户端区域之外
- 页面是父壳，而它渲染的子页面仍然走服务端渲染

官方给这句收尾："This is a build-time optimization only; it does not change what the server sends to the browser."——**它瘦的是服务端包，不是发给浏览器的东西**。

391 那篇讲过 `defineRouteRules`：规则也可以写在页面文件里就近声明，与配在 `nuxt.config` 里是同一套属性。

## 自查

- 九个属性只有三个是"缓存/渲染时机"类：`swr`、`isr`、`prerender`；`ssr: false` 关的是服务端 HTML，`redirect` 是服务端重定向。
- `swr` 与 `isr` 的行为官方写成"same as"，差别只在能否进 CDN（目前 Netlify/Vercel）与 `true` 的失效边界（不带 max-age vs 留到下次部署）。
- TTL 到期是"先发缓存、后台重生成"，不是"先卡住等重生成"。
- `nuxt generate` 下这套不可用；带 `swr`/`isr` 的路由会产出可被 CDN 缓存的 `_payload.json`。
- 挂了 `swr`/`isr`/`cache` 的路由不参与流式渲染；`cache` 属性在 Nuxt 概念页的九条里没有，要看 Nitro 文档。

## 下一篇

下一篇 452 讲比路由规则更细的一层缓存：`defineCachedEventHandler()` 与 `defineCachedFunction()`。官方把它们归在"不可移植"那一类——它们来自 `nitropack/runtime`，而不是 Nuxt 的服务端表面，"想让缓存在任何地方都能跑，就直接依赖一个缓存库"。

## 本文依据

- 配置页 `## routeRules`：定义句"Global route options applied to matching server routes"、`::important` 性质的实验特性声明与指向 Nitro 配置文档、`## nitro` 下 `routeRules` 只有类型 `object` 无说明（`api/nuxt-config`）
- 概念页 Hybrid Rendering 一节：整段引入（内容与后台两种页面的对照）、"define rules for a group of nuxt routes, change rendering mode or assign a cache strategy"、"automatically register corresponding middleware and wrap routes with cache handlers using Nitro caching layer"、八条示例及其全部注释原文、九个属性的类型与说明逐条、`::note` 关于 `isr`/`swr` 生成 `_payload.json`、Server Bundle Size with `ssr: false` 一节（排除逻辑、三种保留情形、build-time optimization only）、平台原生规则映射句、`::important` 关于 `nuxt generate` 不可用、Vercel ISR 示例卡片（`guide/concepts/rendering`）
- `cache` 路由规则的两处出现：`early404` 的 `maxAge`/`Cache-Control` 句与流式退回清单（`guide/going-further/experimental-features`）
- `noScripts` 属性指向 features 页的 read-more、`appMiddleware` 只作用 Vue 应用部分（`guide/concepts/rendering`、`guide/going-further/features`）
