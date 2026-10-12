---
url: /posts/68fa07acb077d9b7aba557acf5c27bf8/
title: 渲染与首屏的十二个实验开关：ssrStreaming 排在最前
date: 2027-06-02T11:00:00+08:00
updated: 2027-06-02T11:00:00+08:00
author: cmdragon

summary:
  摘要：这一组里 `ssrStreaming` 的分量最重，因为它改的是"响应什么时候算定下来"：开启后服务器**立刻把 HTML 外壳发出去**，随后用 Vue 的 `renderToWebStream` 逐段流式渲染 body。代价随之而来——HTTP 状态与响应头在外壳冲出第一字节时就被提交，组件渲染期间调的 `setResponseStatus()`、`useResponseHeader()`、`useCookie()` 写入**都会被丢掉**。其余十一个开关的代价小得多，但各有各的边界，比如 `clientNodePlaceholder` 开了以后传给 `.client.vue` 的属性不会出现在 SSR HTML 里。

categories:
  - 前端开发

tags:
  - nuxt4
  - 实验特性
  - SSR
  - 流式渲染
  - 首屏性能
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## `ssrStreaming`：先看清它换来了什么

原文那句目标：开启 SSR 流式渲染以**显著改善 TTFB**；开启后服务器立即发出 HTML 外壳（含 `<head>`、样式、preload 提示、入口脚本），随后用 Vue 的 `renderToWebStream` 渐进式发送渲染出的 body。

开法有两种，布尔或对象：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    ssrStreaming: {
      botRegex: /googlebot|bingbot|my-internal-crawler/i,
    },
  },
})
```

`botRegex` 那条有两个容易读错的点：默认模式**只匹配索引类爬虫**，而 **Lighthouse 等审计工具是故意落在模式之外的**，这样合成测量看到的才是真实用户拿到的流式响应；另外**设置 `botRegex` 是替换默认模式，不是追加**，所以想排除哪个爬虫就得自己列全。

按路由关掉用 `routeRules`：`'/no-stream/**': { streaming: false }`。

## 七类请求会自动退回非流式

文档用 warning 框列全了：流式会在外壳冲出时提交状态与头，**这与那些需要在渲染之后再改响应的特性不兼容**。命中下列任一的请求不会流式，它们走缓冲渲染，或直接短路成重定向/错误响应：

- `routeRules` 给该路由设了 `noScripts`、`cache`、`isr`、`swr`、`redirect` 或 `streaming: false`
- `ssr: false` 的路由（本来就按 SPA 渲染）
- 爬虫类 user agent，即 `user-agent` 匹配 `botRegex` 的请求
- 预渲染的路由（`nuxt generate`）
- 来自插件、中间件或页面 setup 的服务端 `navigateTo()` 重定向
- 初次渲染（外壳冲出之前）抛出的致命错误

## 边界是"外壳冲出"那一刻，模块作者要按这条改代码

原文把边界切成两半：

- **能到达客户端**：Nuxt 与 Nitro 插件里的改动，它们在渲染开始前就跑完了。
- **会被丢弃**：组件渲染期间（包括路由中间件或 `<script setup>` 里 `await` 之后）调用的 `setResponseStatus()`、`useResponseHeader()`、`useCookie()` 写入，以及 h3 的 `setHeader()`/`appendResponseHeader()`。

两条出口也都给了：静态的 `routeRules: { '/path': { streaming: false } }`，以及运行时的 `render:route` 钩子里设 `ctx.prefersStream = false`（文档举的场景是按 cookie、认证状态或 A/B 分桶决定）。**开发模式下流式处理器会打一条警告，点名被丢弃的改动和路由**，所以这类问题在 dev 里不会静默。

流式期间出错的处理也写了：HTTP 状态已提交之后才发生的错误，会把 `payload.error` 设好，**收尾标签仍然作为一个格式完整的文档发出**，让客户端在水合时读到错误并渲染错误页；外壳冲出之前的错误则落回缓冲的错误渲染器，带正确的状态码。

## 后注册的 head 标签变成客户端补丁

这一条对做 SEO 的项目最要紧。原文：渲染器会等第一次渲染过去之后再冲外壳，所以在 `<script setup>` 里（任何 `await` 之前）、插件里、中间件里**同步注册**的 `useHead`/`useSeoMeta` 照常渲染进外壳的 `<head>`；**更晚注册的标签**（`await` 之后，或嵌套 `<Suspense>` 边界里）已经进不了发出去的 `<head>`，它们作为内联脚本在客户端修补 DOM。

谁会看不到这些标签：跑 JS 的客户端会应用补丁；匹配 `botRegex` 的请求根本不流式，拿到完整缓冲的 `<head>`；**其余的（你没列进 `botRegex` 的爬虫、链接预览抓取器、`curl`）完全看不到这些标签**。例外是 JSON-LD、`noscript` 与 body 位置的标签——它们在 `</body>` 之前作为真实标记渲染，任何客户端都能读到。文档的结论句是：**canonical 链接、社交 meta 这类 SEO 关键标签保持同步注册，或者用 `routeRules: { '/path': { streaming: false } }` 把这条路由退出流式。**

## 样式与水的三处边界

- **路由样式是流式的，JS 提示只到入口**。外壳先冲，所以它的 `<head>` 只带入口 chunk 的样式与提示；页面与布局模块注册之后，它们的 CSS 紧接外壳流出（开了 `inlineStyles` 就内联成 `<style>`，否则是 stylesheet 链接）。**路由专属的 JS chunk 不会从外壳预加载**，浏览器要解析完入口脚本才发现它们。流式对每条路由都改善 TTFB，LCP 收益最大的是"JS 能与入口 chunk 重叠"的路由。
- **开发模式会有 SFC 样式的 FOUC**：dev 下 Vite 把 `<style>` 块当 JS 模块注入，外壳里没有对应 `<link>`，浏览器在注入模块运行之前就开始绘制流来的 DOM。官方的 workaround 是把首屏关键样式放进全局 CSS（`css: ['~/assets/main.css']`），它会以内联 `<link rel="stylesheet">` 进外壳；**生产构建把所有样式抽成真 CSS 文件（或用 `features.inlineStyles` 内联），所以这一条只影响 `nuxt dev`**，验证流式视觉要对 `nuxt build && nuxt preview` 做。
- **生产下嵌套异步组件也会 FOUC**：渲染器只能内联"那一刻模块已注册"的组件样式——页面、布局，以及**直接放在 `<Suspense>` 边界里的异步组件**（Vue 在渲染开始时会立即实例化它们）。渲染在另一个异步组件内部的组件要等父级 resolve，它的 `<style>` 错过外壳后那个样式块，被放进收尾 HTML，于是短暂无样式。四条规避：首屏关键样式走全局 CSS、改用工具类（Tailwind/UnoCSS 的样式在入口样式表里）、把拥有首屏样式的异步组件直接挂在 `<Suspense>` 下、或给该路由关流式。
- **组件岛屿与流式兼容**：岛屿槽内容与 `nuxt-client` 组件本来靠渲染后的一次缝合，流过去之后无法再缝；渲染器改为在文档末尾发出一个惰性的 `<template>`，用水合前运行的内联脚本把它挪到位。**例外是同时用 `features.noScripts` 与岛屿的应用**，因为挪位脚本跑不了，会退回缓冲渲染器。

## 模块参与流式的四个钩子

这一节标题是 Module hooks，四个钩子的语义值得逐条记：

| 钩子 | 时机 | 能做什么 |
| --- | --- | --- |
| `render:route` | 每个请求一次，渲染开始**之前**（流式与否都会跑） | 读 `ctx.canStream` 判断这条请求能否流式；设 `ctx.prefersStream = false` 强制缓冲。只有 `canStream && prefersStream` 才流式 |
| `render:html` | 外壳冲出**之前**一次，第二个参数带 `streaming: true` | 改 `htmlAttrs`、`head`、`bodyAttrs`、`bodyPrepend` 能上到线上；**改 `body`/`bodyAppend` 会被丢弃**（因为 body 即将流式，dev 模式会警告）。只改 head 字段的模块（CSP 注入、OG 标签、分析 meta）无需改动即可兼容 |
| `render:html:chunk` | 渲染器每产出一个 chunk、入队之前 | 改 `ctx.chunk`（`Uint8Array`）做字节级变换（例如注入 nonce）；读 `ctx.index` 区分首块与后续块 |
| `render:html:close` | body 流结束后、收尾标签之前 | 改 `ctx.bodyAppend`（`string[]`）注入末尾标记（结束时机的分析标签、服务端渲染的调试组件等） |

CSP 那条单独有一处 note：流式渲染器会产出若干绕过 unhead 的内联脚本与样式（引导队列、IIFE、suspense 的 head push、岛屿 teleport 的挪位脚本、路由 `<style>` 块）。**只要渲染出的 head 脚本上带了 `nonce`，渲染器会自动把这些都盖上同一个 nonce**，所以模块只需要把 nonce 放到 head 脚本上，字节级需求才用 `render:html:chunk`。

## 其余十一个开关，按代价排一遍

| 开关 | 默认 | 语义与注意点 |
| --- | --- | --- |
| `navigationRepaint` | 开 | 导航前等一个动画帧，让浏览器有机会重绘、确认用户交互；**在预渲染路由上导航时能降低 INP** |
| `headNext` | 开 | head 优化：加 capo.js 插件以更高效地渲染标签，并用 hash hydration 插件减少初始水合 |
| `spaLoadingTemplateLocation` | 未声明 | 客户端页面（`ssr: false`）渲染 `~/spa-loading-template.html` 的位置：`within` 放进 `<div id="__nuxt">` 内，`body` 放在根节点旁边；**原文说这样能避免水合时的白闪** |
| `clientNodePlaceholder` | `compatibilityVersion` ≥5 时开 | 用注释节点代替空 `<div>` 作客户端专用组件的占位，修的是占位 div 与真实根同名时 scoped 样式失效的问题；**代价是传给 `.client.vue` 的 `class`/`style` 等属性不会出现在 SSR HTML**，要防布局位移就用带 `#fallback` 槽的 `<ClientOnly>` |
| `serverPathFallback` | 开 | 匹配不到页面路由的客户端导航按整文档加载，让服务端能回 `public/` 里的文件、服务端路由或 404；当前文档已经就是该路径时不会重载；**`hashMode` 开启时无效** |
| `prefetchPreloadTags` | 关 | 目标路由启用了 payload 提取时，把目标 `<link rel="preload">` 转发进当前文档；转发后**从 `preload` 降级为 `prefetch`，`as="image"` 保留 `preload` 但去掉 `fetchpriority`**；默认关的理由写得很实：与 `prefetchOn: 'visibility'`（NuxtLink 默认）叠加会一次触发大量跨路由预取；另外 Chromium/Safari 会为每条未访问路由上的图片提示打 `was preloaded... not used` 警告，且转发请求**不带 `Purpose: prefetch` 头，会算进图片 CDN 的配额与限流** |
| `emitRouteChunkError` | `automatic` | chunk 加载失败时发 `app:chunkError`；`automatic` 是导航到新路由时重载新路由，`automatic-immediate` 是失败当刻就重载当前路由（适合非导航触发的懒组件失败，副作用是可能不需要那个 chunk 也重载了），`false` 关闭自动处理，`manual` 手动接管 |
| `restoreState` | 未声明 | 从 `sessionStorage` 恢复应用状态（chunk 错误或手动 `reloadNuxtApp()` 之后）；为避免水合错误**只在 Vue 应用挂载后应用，所以首屏可能有闪动**；那条重要提示要求慎重，并建议给 `useState` 显式 key，因为自动生成的 key **可能跨构建不一致** |
| `clientFallback` | 未声明 | 打开 `<NuxtClientFallback>` 组件，用于 SSR 出错时在客户端渲染内容 |
| `crossOriginPrefetch` | 未声明 | 用 Speculation Rules API 打开跨源预取 |
| `viewTransition` | 未声明 | 与客户端路由集成 View Transition API；除了 `true` 还能传对象 `{ enabled: true, types: ['slide'] }`，按导航类型给不同 CSS 动画 |

这一组里**唯一把"会被丢弃"写成文档级风险的只有 `ssrStreaming`**；其余的代价都局部：一个属性不上 HTML、一个降级、一个闪动。

下一篇看数据与缓存那一组九个：`payloadExtraction` 的三种取值、`granularCachedData` 与 `purgeCachedData` 各自管什么、以及 `sharedPrerenderData` 为什么对预渲染站点影响大。

## 本文依据

- https://nuxt.com/docs/4.x/guide/going-further/experimental-features
