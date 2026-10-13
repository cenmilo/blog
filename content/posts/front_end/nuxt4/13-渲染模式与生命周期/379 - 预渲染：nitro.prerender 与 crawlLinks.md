---
url: /posts/eec5435cc04f5b7d7a2a170f52484d7d/
title: 379 - 预渲染：nitro.prerender 与 crawlLinks
date: 2027-02-15T10:00:00+08:00
updated: 2027-02-15T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：预渲染就是在构建期把页面拼成静态 HTML，请求时直接发成品、不再现算。`nuxt generate` 就是"带 `nitro.static:true` 的 `nuxt build`"（等价 `nuxt build --prerender`），靠 Nitro 爬虫从根页 `/` 出发，顺着页面里的 `<a href>` 一层层抓、每抓一页存下 HTML 和 `_payload.json`。指定要渲/要跳过用顶层 `prerender.routes`、`prerender.ignore`；爬虫发现不了的（`/sitemap.xml`、`/robots.txt`）靠 `crawlLinks` 或直接列进 `routes`；并发、`failOnError` 这类 Nitro 专属项放 `nitro.prerender`。routeRules 的 `prerender: true/false` 也能按路由控制。

categories:
  - 前端开发

tags:
  - nuxt4
  - 预渲染
  - generate
  - crawlLinks
  - 静态站
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

367 见过 routeRules 里的 `prerender: true`，但整站预渲染、以及"哪些页会被渲出来"这件事，官方集中在 `getting-started/prerendering` 讲。这一篇先把预渲染是什么、用什么命令、怎么指定范围说清，路由从哪来与自动发现的边界留到 380。

## 预渲染是什么

官方定义：

> Nuxt allows for select pages from your application to be rendered at build time. Nuxt will serve the prebuilt pages when requested instead of generating them on the fly.

构建期把选定的页面渲好、请求时直接发成品。和 SSR 的区别就是"现算 vs 预先算好"。

## `nuxt generate` 到底干了啥

Crawl-based 那节把命令讲透：

> Use the `nuxt generate` command to build and pre-render your application using the Nitro crawler. This command is similar to `nuxt build` with the `nitro.static` option set to `true`, or running `nuxt build --prerender`.

三句话等价：**`nuxt generate` ≈ `nuxt build` + `nitro.static: true` ≈ `nuxt build --prerender`**。它建站、起一个 nuxt 实例，然后**默认预渲染根页 `/`，加上它链接到的任何页面、那些页面又链接到的页面，依此类推**。

产物在 `.output/public`，可丢任何静态托管，本地用 `npx serve .output/public` 预览。静态与预渲染构建**还会产出 `200.html` 和 `404.html` 两个 SPA 回退页**（382 细讲）。

## 手动指定要渲 / 要跳过：Selective Pre-rendering

爬虫顺链接抓不一定合你意，官方给了 `prerender.routes` 与 `prerender.ignore`：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  prerender: {
    routes: ['/user/1', '/user/2'],
    ignore: ['/dynamic'],
  },
})
```

`routes` 手动要渲、`ignore` 手动不渲。

## `crawlLinks`：抓那些没被链接指向的页

有些页没有任何链接指向它（`/sitemap.xml`、`/robots.txt`），爬虫发现不了。用 `crawlLinks: true` 让爬虫开抓，再把这类路由补进 `routes`：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  prerender: {
    crawlLinks: true,
    routes: ['/sitemap.xml', '/robots.txt'],
  },
})
```

## `prerender` vs `nitro.prerender`：两个层次

这一条容易混，官方在 Selective 末尾专门分开：

> The top-level `prerender` option applies whichever `server.builder` you use. Builder-specific options, such as Nitro's `concurrency` or `failOnError`, can be set in `nitro.prerender`.

- **顶层 `prerender`**：换哪个 `server.builder` 都生效（`routes`、`ignore`、`crawlLinks` 放这）。
- **`nitro.prerender`**：Nitro 这个构建器专属的选项放这，文档举例 `concurrency`（并发数）、`failOnError`（某路由渲失败要不要整体报错）。

要精确控制 Nitro 的并发、失败行为，就去 `nitro.prerender` 配；跨构建器通用的路径控制留在顶层 `prerender`。

## 用 routeRules 按路由控制

最后一个入口是 routeRules（367 见过）：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    '/rss.xml': { prerender: true },
    '/this-DOES-NOT-get-prerendered': { prerender: false },
    '/blog/**': { prerender: true },
  },
})
```

`prerender: true` 配进去、`prerender: false` 显式跳过；`/blog/**` 下只要被别的页链接到就渲。还有个页面内速写 `defineRouteRules({ prerender: true })`——但它是实验特性，要开 `experimental.inlineRouteRules`。

## 自查

- 预渲染＝构建期拼好 HTML、请求时发成品；`nuxt generate` ≈ `nuxt build` 带 `nitro.static:true` ≈ `nuxt build --prerender`。
- 默认从 `/` 起顺 `<a href>` 爬，产物 `.output/public`，另产 `200.html`/`404.html`。
- `prerender.routes`/`ignore` 手动要渲/跳过；`crawlLinks: true` 处理没链接指向的页（sitemap/robots）。
- 顶层 `prerender` 跨构建器生效；`nitro.prerender` 放 Nitro 专属的 `concurrency`/`failOnError`。
- routeRules 的 `prerender: true/false` 按路由控制。

## 下一篇

下一篇 380 讲预渲染路由到底从哪来、以及自动发现的硬边界：没被任何可发现页链接指向的页面没法被自动预渲染，得靠 `prerenderRoutes()`、`prerender:routes` 钩子等补上。

## 本文依据

- 预渲染定义、`nuxt generate` ≈ `nitro.static`/`nuxt build --prerender`、默认从根页顺链接爬、`.output/public` 与 `npx serve`、200/404 回退、Selective 的 `prerender.routes`/`ignore`、`crawlLinks` 补 sitemap/robots、顶层 `prerender` 与 `nitro.prerender`（concurrency/failOnError）分工、routeRules `prerender:true/false`、`defineRouteRules`+`experimental.inlineRouteRules`（`getting-started/prerendering`；routeRules 见 367）。
