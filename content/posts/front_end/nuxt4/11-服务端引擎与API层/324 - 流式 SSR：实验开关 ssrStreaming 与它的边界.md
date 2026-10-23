---
url: /posts/a681207c18993b8ccb34d01ae4a35ac5/
title: 324 - 流式 SSR：实验开关 ssrStreaming 与它的边界
date: 2027-01-28T09:00:00+08:00
updated: 2027-01-28T09:00:00+08:00
author: cmdragon

summary:
  摘要：普通 SSR 是"服务端把整页渲染完才发 HTML"，首字节要等最慢那段取数。experimental.ssrStreaming 改成先立刻发 HTML 外壳（head、样式、预取提示、入口脚本），再用 Vue 的 renderToWebStream 把 body 内容逐步流出去，显著压低 TTFB。这一篇讲它的机制、开关写法、几个会"自动退回缓冲渲染"的情况（命中缓存/isr/swr/redirect、ssr:false、机器人 UA、预渲染、服务端重定向、shell 刷出前的致命错误），以及最重要的一条边界：HTTP 状态码和响应头必须在 shell 刷出前设好，刷出后再 setResponseStatus/useResponseHeader/useCookie 会被丢弃。

categories:
  - 前端开发

tags:
  - nuxt4
  - 流式渲染
  - ssrStreaming
  - TTFB
  - botRegex
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


普通 SSR（319 的第 7 步）是"全部渲染完才发 HTML"——首字节要等最慢的那段数据取完。流式渲染换了个顺序：先把"壳"发出去让浏览器动起来，body 内容随后一块块补。Nuxt 用实验开关 `experimental.ssrStreaming` 提供它。

## 机制

官方说明：

> Enables SSR streaming to dramatically improve Time to First Byte (TTFB). When enabled, the server sends the HTML shell (including `<head>`, styles, preload hints, and entry scripts) immediately, then streams the rendered body content progressively using Vue's `renderToWebStream`.

先解释 TTFB（Time to First Byte）：浏览器发出请求到收到第一个字节的时间。流式的做法：

- **立刻发 HTML 外壳（shell）**：含 `<head>`、样式、预取提示（preload hints）、入口脚本。
- **再逐步流式输出 body**：用 Vue 的 `renderToWebStream` 把渲染出来的内容一块块推。

开关写法：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    ssrStreaming: true,
  },
})
```

## 爬虫会被自动排除

流式和 SEO 有冲突（爬虫常常拿不到完整 DOM），官方默认帮爬虫走完整渲染：

> Streaming is automatically disabled for bot and crawler user agents (such as Googlebot, Bingbot, etc.) to ensure search engines receive fully-rendered HTML for SEO safety. The default pattern matches indexing crawlers only; Lighthouse and other audit tools deliberately fall outside it so synthetic measurements reflect the same streamed response real users get.

可以自定义哪些 UA 被排除，用 `botRegex`：

```ts
experimental: {
  ssrStreaming: {
    botRegex: /googlebot|bingbot|my-internal-crawler/i,
  },
}
```

一条坑 note：

> Setting `botRegex` replaces the default pattern rather than adding to it, so list every crawler you want to keep out of streaming.

`botRegex` 是**整个替换**默认模式，不是追加——你一旦设它，就得把自己想排除的爬虫列全。也呼应前面那句：Lighthouse 这类审计工具**故意不在**默认 botRegex 里，好让它们量到的就是真实用户拿到的流式响应。

## 一条最该记住的边界：状态与头要在 shell 刷出前设好

官方 warning：

> Response status and headers must be set before the shell is flushed. Streaming commits the HTTP status and headers with the first byte, so anything that mutates the response after that point cannot reach the client. This is inherent to streaming, not a Nuxt-specific bug.

流式在吐出第一个字节时就把状态码和响应头"焊死"了。官方给了分界（以 shell 刷出为界）：

- **能到客户端**：Nuxt 与 Nitro **插件**里的改动（插件在渲染开始前就全跑完）。
- **会被丢弃**：在组件渲染期间调用的 `setResponseStatus()`、`useResponseHeader()`、`useCookie()` 写入、h3 的 `setHeader()`/`appendResponseHeader()`——**包括路由中间件或 `<script setup>` 里 `await` 之后的那些**，因为它们发生在 shell 已上线之后。

324 这条直接推翻了 307/315/317 的写法在流式路由上的可用性：开流式后，你要保留一个响应改动，就**挪进插件**，或者给这条路由**退出流式**：

- `routeRules: { '/path': { streaming: false } }`：按路由静态关。
- `render:route` 钩子里 `ctx.prefersStream = false`：按请求运行时关。

好消息：官方说**开发模式下流式处理器会打一条 warning 点名被丢弃的改动和路由**，"so these never fail silently"——不会让你静默踩坑。

## 会退回缓冲渲染的情况

官方列了一张"不流式、走缓冲或直接短路"的清单：

> Requests matching any of the following are not streamed; they use the buffered renderer, or short-circuit to a redirect or error response:
> - `routeRules` setting `noScripts`, `cache`, `isr`, `swr`, `redirect`, or `streaming: false` for the route
> - `ssr: false` routes (already SPA-rendered)
> - Bot/crawler user agents ... matches `botRegex`
> - Prerendered routes (`nuxt generate`)
> - Server-side `navigateTo()` redirects from plugins, middleware, or page setup
> - Fatal errors thrown during initial render (before the shell flushes)

也就是说命中缓存策略（isr/swr/cache）、noScripts、`ssr:false`、纯预渲染、机器人 UA、服务端重定向、shell 前的致命错误——这些都不走流式，退回缓冲渲染或短路。

## head 标签晚注册会变成客户端补丁

和 194、208 一路的"SEO 标签要同步"连上。官方 warning：

> Tags registered later (after an `await` in setup, or inside a nested `<Suspense>` boundary) cannot reach the already-sent `<head>`; they are delivered as inline scripts that patch the DOM on the client.

在 shell 刷出**之前**同步注册的 `useHead`/`useSeoMeta` 正常进 `<head>`；之后注册的进不了已发出的 `<head>`，只能当内联脚本在客户端补 DOM——而"不跑 JS 的爬虫、没列进 botRegex 的爬虫、link unfurler、curl"就看不到这些标签。官方点名例外：**JSON-LD、`noscript`、body 位置的标签**是"作为真实标记渲染在 `</body>` 前"、所有客户端都读得到；其余 SEO 关键标签（canonical、社交 meta）要么保持同步、要么给那条路由 `streaming: false`。

## 模块怎么参与流式

官方给了流式相关的一组钩子，模块作者用：

- **`render:route`**：每次渲染前按请求触发一次，读 `ctx.canStream` 判断这路由能不能流式，设 `ctx.prefersStream = false` 强制缓冲——"The renderer streams only when `canStream && prefersStream`"。
- **`render:html`**：shell 刷出前触发一次、第二参带 `streaming: true`；改 `htmlAttrs`/`head`/`bodyAttrs`/`bodyPrepend` 能上线，改 `body`/`bodyAppend` 被丢弃（body 即将流式）。
- **`render:html:chunk`**：每个渲染块入队前触发，改 `ctx.chunk: Uint8Array` 做字节级变换（如注 nonce），读 `ctx.index` 分清首块。
- **`render:html:close`**：body 流完、闭合标签前触发，改 `ctx.bodyAppend` 注入收尾标记。

## 自查

- 流式 = 先发 shell（head/样式/入口）再流 body，主要降 TTFB；用 `experimental.ssrStreaming` 开。
- 状态码/响应头/cookie 写入**必须在 shell 刷出前**（挪进插件），否则被丢弃；要保留就 `routeRules: { streaming: false }` 或 `render:route` 里 `prefersStream = false`。
- isr/swr/cache/noScripts/ssr:false/预渲染/机器人/服务端重定向/shell 前致命错误都会退回缓冲。
- SEO 关键标签保持同步注册；晚注册的变客户端补丁、不跑 JS 的抓取端看不到。开流式视觉问题要用 `nuxt build && nuxt preview` 验，别只看 dev。

## 下一篇

下一篇 325 讲 routeRules 的服务端选项：redirect、headers、cache、swr/isr、cors、proxy 这些规则各自干什么、在哪生效。
