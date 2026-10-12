---
url: /posts/c5ad66d79c5fc1edde87ceec6c080415/
title: routeRules 的服务端选项：redirect、headers、cache 与 proxy
date: 2027-01-28T10:00:00+08:00
updated: 2027-01-28T10:00:00+08:00
author: cmdragon

summary:
  摘要：routeRules 是"按路径给服务端定规则"的一张表，Nuxt 会自动为匹配的路由注册对应中间件、并用 Nitro 缓存层包住。这一篇把它的服务端侧选项逐个讲清：redirect（服务端重定向，可带 statusCode）、headers（给某段路径加响应头）、cors（自动加跨域头、可用 headers 覆盖）、cache（把整个响应缓存、命中直接回）、swr/isr（带 TTL 或留到下次部署的缓存策略，isr 能进 CDN）、ssr:false（这段只浏览器渲染）、prerender（构建期预渲染）、noScripts、appMiddleware。并给 routeRules 的三处入口（nuxt.config / defineRouteRules / extendRouteRules）和"nuxt generate 下混合渲染不可用"的限制。

categories:
  - 前端开发

tags:
  - nuxt4
  - routeRules
  - 缓存
  - 重定向
  - 混合渲染
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

routeRules 在 140–141 那两篇开过头，这一篇把它**服务端侧**的选项摊全。先记它的机制，官方在 Hybrid Rendering 那节写得很准：

> Nuxt server will automatically register corresponding middleware and wrap routes with cache handlers using Nitro caching layer.

即 routeRules 不是"提示"，Nuxt 会真的为匹配路径**注册中间件 + 用 Nitro 缓存层包住路由**。配置形状（官方示例节选）：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    '/': { prerender: true },
    '/products': { swr: true },
    '/products/**': { swr: 3600 },
    '/blog': { isr: 3600 },
    '/blog/**': { isr: true },
    '/admin/**': { ssr: false },
    '/api/**': { cors: true },
    '/old-page': { redirect: '/new-page' },
  },
})
```

## 重定向：redirect

官方描述：

> `redirect: string` - Define server-side redirects.

写字符串或写带状态码的对象：`redirect: { to: '/new-page', statusCode: 302 }`。它是**服务端**重定向——发生在任何前端导航守卫之前（206），且会走 `Location` 头让浏览器重新请求、应用状态被重置（319 那条）。

## 响应头：headers 与 cors

> `headers: object` - Add specific headers to sections of your site - for example, your assets.
> `cors: boolean` - Automatically adds cors headers with `cors: true` - you can customize the output by overriding with `headers`.

`headers` 给某段路径挂固定响应头（比如给静态资源加缓存控制头）；`cors: true` 自动加跨域头，要精细控制就用 `headers` 覆盖。这和 315/317 那种"代码里读/设头"不同——routeRules 是"声明式按路径设头"。

## 缓存：cache、swr、isr

三个都关缓存，层级不同：

> `swr: number | boolean` - Add cache headers to the server response and cache it on the server or reverse proxy for a configurable TTL (time to live). The `node-server` preset of Nitro is able to cache the full response. When the TTL expired, the cached response will be sent while the page will be regenerated in the background. If true is used, a `stale-while-revalidate` header is added without a MaxAge.
> `isr: number | boolean` - The behavior is the same as `swr` except that we are able to add the response to the CDN cache on platforms that support this (currently Netlify or Vercel). If `true` is used, the content persists until the next deploy inside the CDN.

拆开：

- `cache`（getting-started/server 示例 `'/api/*': { cache: { maxAge: 60 * 60 } }`）：缓存整个响应、命中直接回，TTL 由 maxAge 定。experimental 那页也确认"cache route rule 覆盖路径时，GET/HEAD miss 会把 maxAge 当 Cache-Control 播报，让 CDN 吸收重复请求"。
- `swr`：stale-while-revalidate，服务端/反代缓存、过 TTL 先回旧的再后台重生成。
- `isr`：和 swr 一样，但能进 **CDN** 缓存（目前 Netlify/Vercel），`isr: true` 内容留到下次部署。

一条连带的（note）：用 isr/swr 的路由也会产出 `_payload.json`，客户端导航复用缓存的 payload 而不是重取数（200、320 那条 payload extraction）。

## 只客户端渲染：ssr:false

> `ssr: boolean` - Disables server-side rendering of the HTML for sections of your app and make them render only in the browser with `ssr: false`

官方示例给的是 admin 区 `'/admin/**': { ssr: false }`——内容页静态、后台偏动态，这种"按路径切渲染模式"正是混合渲染的用武之地。附带一条构建期优化（273 那节）：被 `ssr:false` 覆盖且构建期可确定的页面，其组件会从服务端 bundle 里排除；但更具体规则重开 SSR、有 alias/绝对路径子路由落在区域外、父壳仍服务端渲染子页——这三种页面会被留在 bundle 里。它只是构建期优化，不改变服务端发给浏览器的内容。

## 其余服务端选项

> `prerender: boolean` - Prerenders routes at build time and includes them in your build as static assets.
> `noScripts: boolean` - Disables rendering of Nuxt scripts and JS resource hints for sections of your site.
> `appMiddleware: string | string[] | Record<string, boolean>` - Allows you to define middleware that should or should not run for page paths within the Vue app part of your application (that is, not your Nitro routes)

`prerender`（323/200）、`noScripts`、`appMiddleware`。特别注意 **appMiddleware 只管 Vue 应用侧的页面路径，不管 Nitro 路由**（"not your Nitro routes"）——服务端路由要拦得用服务端中间件（291）。

## 还有 proxy

experimental 那页提到 routeRules 的 `proxy`（"routeRules proxy or a catch-all handler"）。它把某路径代理转发到别处，属于 routeRules 的一个能力，但**渲染那页的 Route Rules 列表里没有单列它**，官方对它的完整字段没有在这批文档里逐项写开，我不替它编参数表。要精确用法去查 Nitro 文档。

## 三处入口与一条限制

routeRules 能写三个地方：

1. `nuxt.config.ts` 的 `routeRules`（上面这些示例）。
2. 页面里 `defineRouteRules({...})`（需 `experimental.inlineRouteRules: true`，140 讲过；`pages/foo/[id].vue` → `/foo/*`，含正则/部分段/可重复参数的路径无法转换、会构建期告警并跳过）。
3. 模块里 `extendRouteRules`（Kit，可写 redirect{to,statusCode}、cache{maxAge}）。

一条硬限制（important）：

> Note that Hybrid Rendering is not available when using `nuxt generate`.

**`nuxt generate` 下混合渲染不可用**（200 也提过）——纯静态生成时别指望 routeRules 的缓存/SSR 切换生效。

## 自查

- 声明式按路径设行为用 routeRules：redirect 服务端重定向、headers/cors 加响应头、cache/swr/isr 缓存（isr 才进 CDN）、ssr:false 该段只客户端、prerender 构建期预渲染、appMiddleware 只管 Vue 页面路径。
- 用了 isr/swr 的路由也产 `_payload.json`，客户端导航复用缓存数据。
- 三处入口（nuxt.config / defineRouteRules / extendRouteRules）；`nuxt generate` 下混合渲染不生效。
- proxy 是 routeRules 的一项，但本批文档没逐项写它的字段，别照旧教程脑补参数。

## 下一篇

下一篇 326 把服务端中间件落到实操：用 `server/middleware` 做鉴权（抛 401）与日志（记 URL），并守住"中间件不该返回值"这条。

## 本文依据

- Rendering Hybrid Rendering 节（`guide/concepts/rendering`）：机制句"automatically register corresponding middleware and wrap routes with cache handlers using Nitro caching layer"；示例 routeRules；Route Rules 属性清单（redirect/ssr/cors/headers/swr/isr/prerender/noScripts/appMiddleware 的官方描述逐条）；swr/isr 的 TTL、full response、stale-while-revalidate、CDN、true 语义；`ssr:false` Server Bundle Size 三条保留条件；note isr/swr 也产 `_payload.json`；important "Hybrid Rendering is not available when using nuxt generate."
- Server 页（`getting-started/server`）：routeRules 示例 `'/api/*': { cache: { maxAge: 60 * 60 } }`、`redirect: { to, statusCode: 302 }`。
- Experimental Features（`guide/going-further/experimental-features`）：cache route rule 在 GET/HEAD miss 播报 maxAge 为 Cache-Control；routeRules proxy 提及。
- defineRouteRules / extendRouteRules（见 140）。
