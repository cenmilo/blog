---
url: /posts/711805160f086fd4798f30cc3d3e5c37/
title: 缓存命中时取数还跑吗：swr、isr 与 payload 文件
date: 2026-12-30T09:00:00+08:00
updated: 2026-12-30T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇把"路由级缓存"和"取数"两件事对上，因为它们的边界官方是在两页各写一半。`swr` 的定义句："Add cache headers to the server response and cache it on the server or reverse proxy for a configurable TTL (time to live). The `node-server` preset of Nitro is able to cache the full response. **When the TTL expired, the cached response will be sent while the page will be regenerated in the background.** If true is used, a `stale-while-revalidate` header is added without a MaxAge."；`isr` 只改一句作用域："The behavior is the same as `swr` except that we are able to add the response to the CDN cache on platforms that support this (currently Netlify or Vercel). If `true` is used, the content persists until the next deploy inside the CDN."**注意它俩缓存的是"整个响应"**——所以命中时页面组件的 setup 根本没跑，取数自然也没跑。官方在渲染那节还补了一条与取数直接相关的 note："Routes using `isr` or `swr` also generate `_payload.json` files alongside HTML. Client-side navigation loads these cached payloads instead of re-fetching data."。再把 204 那条合上来才完整：混合渲染命中 Nitro 缓存时**所有入站请求头被丢弃**，此时 `useRequestURL()` 的 host 读成 `localhost`，要靠 `cache.varies` 把 `host`/`x-forwarded-host` 纳进缓存键。配置示例里每行注释都写清了语义（`swr: true` 无 MaxAge、`swr: 3600` 一小时、`isr: true` 留到下次部署）。

categories:
  - 前端开发

tags:
  - nuxt4
  - swr
  - isr
  - 缓存
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先给一个能自测的现象：给某个页面开了 `swr`，然后在它的 `useAsyncData` handler 里加一行 `console.log`。**连续请求两次，日志只出现一次**——第二次是缓存直接吐出来的。这一条我按官方"缓存整个响应"的说法解释得通，但它值得你自己跑一次，因为各平台的缓存层行为不一样。

## 官方那九条属性里，两条是缓存

配置块连同每行注释我原样抄一遍（注释是官方写的，正好是语义说明）：

```ts twoslash [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    // Homepage pre-rendered at build time
    '/': { prerender: true },
    // Products page generated on demand, revalidates in background, cached until API response changes
    '/products': { swr: true },
    // Products page generated on demand, revalidates in background, cached for 1 hour (3600 seconds)
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

两条完整定义（照抄，粗体是我加的标记）：

- `swr: number | boolean` —— "Add cache headers to the server response and cache it on the server or reverse proxy for a configurable TTL (time to live). The `node-server` preset of Nitro is able to **cache the full response**. When the TTL expired, **the cached response will be sent while the page will be regenerated in the background**. If true is used, a `stale-while-revalidate` header is added without a MaxAge."
- `isr: number | boolean` —— "The behavior is the same as `swr` except that we are able to add the response to **the CDN cache** on platforms that support this (currently Netlify or Vercel). If `true` is used, **the content persists until the next deploy** inside the CDN."

三个我要划出来的点：

1. **"cache the full response"**：缓存对象是整个响应（HTML + 随它带出的 payload），不是接口结果。所以命中时**页面组件的 setup 不执行，取数不跑**。
2. **TTL 到期时先发旧内容**，新页面在后台生成——用户看到的那一屏是过期的（这是设计如此，不是 bug）。
3. `isr: true` 的有效期单位是**一次部署**，而 `swr: true` 是"没有 MaxAge 的 stale-while-revalidate"。这两者都不是"永久缓存"，含义不同。

另外这一节的机制句（140 也引过）值得再来一遍：**Nuxt 会自动注册对应中间件并用 Nitro 缓存层包裹路由**——`routeRules` 里那些属性的执行者是 Nitro，不是 Vue 侧。

## 与取数直接相关的那条 note

> Routes using `isr` or `swr` also generate `_payload.json` files alongside HTML. Client-side navigation loads these cached payloads instead of re-fetching data.

这条把两件事连上了：**缓存路由的数据不只缓在 HTML 里，还缓在一个 payload 文件里**，而客户端导航读的是那个文件（235 讲的 `payloadExtraction` 正是这条通道）。

所以一条 `swr`/`isr` 路由上有两层"不重新取数"：

| 场合 | 为什么不取数 |
| --- | --- |
| 服务端命中缓存 | 整个响应被复用，setup 没跑 |
| 客户端导航到它 | 直接读 `_payload.json`，里面已经有首屏那次取数的结果 |

代价也一样两层：**数据可能是过期**（服务端 TTL 与构建期快照两种来源），200 那条 "data can be stale until the next rebuild" 讲的是静态站的同一类后果。

## 最需要小心的那一条：请求头会被丢

204 引过的那条 important 在这里变成必修项：

> When utilizing Hybrid Rendering with cache strategies, all incoming request headers are dropped when handling the cached responses via the Nitro caching layer (meaning `useRequestURL` will return `localhost` for the `host`).
>
> You can define the `cache.varies` option to specify headers that will be considered when caching and serving the responses, such as `host` and `x-forwarded-host` for multi-tenant environments.

我把它翻译成三条可执行结论：

1. **命中缓存时，任何"从请求头读出来的东西"都不可用**——包括 `useRequestURL().host`、以及依赖 `useRequestHeaders` 的鉴权转发（223 讲它在浏览器返回空对象，这里是另一种"读不到"）。
2. 多域名/多租户下必须把区分依据放进 `cache.varies`，否则**不同域会共用同一份缓存**。
3. 页内如果依赖 cookie 决定内容，就不要给这条路由开缓存——`swr` 与"按用户身份渲染"是互斥的。

顺带一提，`api/kit` 那节里那个把 `/robots.txt` 注册成服务端 handler 的例子（205）也说明：**服务端 handler 是普通 Nitro 路由，不受页面级 `swr`/`isr` 约束**，但如果你给它单独配了规则，同样按上面的行为算。

## 我该配哪一档

按文档给的语义，我的判断顺序：

1. 内容与用户无关、更新不频繁 ⇒ `prerender`（构建期一次性，最快）。
2. 内容会更新、需要自动过期 ⇒ `swr: 秒数`。
3. 部署平台是 Netlify/Vercel 且能接受"更新等到下次部署" ⇒ `isr: true`。
4. 内容按用户不同 ⇒ **不要开这三档**，老老实实每次渲染 + 用 `useAsyncData` 自己控制重取（240）。

最后一条我想强调：**开了缓存就等于把"取数的新鲜度"交给了 HTTP 缓存层**，`getCachedData`/`refresh`（236/240）那些手段全在应用层，够不到它。

## 自查

1. `swr` 与 `isr` 缓存的对象是什么？命中时页面 setup 跑吗？（答：整个响应；不跑，所以取数也不跑。）
2. TTL 刚到期时用户看到什么？（答：官方说先发缓存的那份，页面在后台重新生成——那一屏是过期的。）
3. 混合渲染命中缓存时，从请求头读值会出什么问题，怎么缓解？（答：所有入站头被丢弃、`useRequestURL` 的 host 变成 `localhost`；把 `host`/`x-forwarded-host` 放进 `cache.varies`。）

## 下一篇

238 - 配方：给外部接口做一个带鉴权的取数函数

## 本文依据

- `guide/concepts/rendering`：`routeRules` 配置块与九行官方注释；Route Rules 九条属性定义原文（尤其 `swr`、`isr` 两句）；"Routes using `isr` or `swr` also generate `_payload.json` files..." 那条 note；`ssr: false` 与服务端包体积一段（更具体规则重新开启 SSR 等例外）。
- `api/composables/use-request-url`：headers dropped ⇒ host 为 `localhost`、`cache.varies` 与 `host`/`x-forwarded-host`（204 已逐句引）。
- `getting-started/prerendering`：Payload Extraction 一节（`isr`/`swr` 首次渲染时生成 payload、CDN 可缓存、`'/**': { isr: true }` 的写法、stale 那句）与 `payloadExtraction` 三取值（200 已逐句引）。
- `api/nuxt-config`、`api/kit/pages`：routeRules 的机制句与 `extendRouteRules`（140/206 已逐句引）。
- `api/composables/use-request-headers`、`api/utils/call-once`：读头与"服务端那一趟"的边界（223/230 已逐句引）。
