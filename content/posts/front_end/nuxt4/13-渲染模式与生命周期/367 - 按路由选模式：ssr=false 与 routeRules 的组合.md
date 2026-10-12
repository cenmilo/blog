---
url: /posts/6a63d3eceba380bfc1900cba7c88d458/
title: 按路由选模式：ssr=false 与 routeRules 的组合
date: 2027-02-11T10:00:00+08:00
updated: 2027-02-11T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：整站一刀切 SSR 或 CSR 常常不够——营销页要 SEO、后台要纯客户端。routeRules 让你按路由分组决定"这一片页走哪条渲染、用什么缓存"。官方文档列了九个可用属性，逐个讲清并给默认行为：`redirect`、`ssr`、`cors`、`headers`、`swr`、`isr`、`prerender`、`noScripts`、`appMiddleware`（注意 `proxy` 不在这份清单里）。还讲一件容易忽略的事：被 `ssr: false` 覆盖的路由，它的页面组件会被从服务端包体里排除掉（仅构建期优化），以及三条"仍保留进服务端包体"的例外。最后一条硬限制：hybrid 在 `nuxt generate` 下用不了。

categories:
  - 前端开发

tags:
  - nuxt4
  - routeRules
  - 混合渲染
  - ssr
  - 缓存策略
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

一个内容站带一个后台，是最好的例子：内容页应该静态、生成一次就够，后台要登录、更像动态应用。以前一个 Nuxt 应用只能全站选一种模式（全 universal 或全客户端）。Hybrid rendering 打破这条——它用 Route Rules 按路由分组，决定服务器对某个 URL 该怎么响应。

## 机制：它到底改了什么

官方对 hybrid 的原理说得很实：

> Using route rules you can define rules for a group of nuxt routes, change rendering mode or assign a cache strategy based on route! Nuxt server will automatically register corresponding middleware and wrap routes with cache handlers using Nitro caching layer.

也就是：你在 `routeRules` 写规则，Nuxt 服务端**自动注册对应中间件**、并用 **Nitro 缓存层**包住路由。你只声明意图，机制它自己接。

## 九个属性逐条

官方在 Route Rules 小节明确列出"你能用的属性有这些"，共九项，逐条（类型照文档）：

| 属性 | 类型 | 作用（依文档） |
| --- | --- | --- |
| `redirect` | `string` | 定义服务端重定向 |
| `ssr` | `boolean` | 关掉应用某些部分的 HTML 服务端渲染，让它 `ssr: false` 只在浏览器渲染 |
| `cors` | `boolean` | `cors: true` 自动加跨域头；可用 `headers` 覆盖来自定义输出 |
| `headers` | `object` | 给站点某些部分（比如你的资源）加特定响应头 |
| `swr` | `number \| boolean` | 给服务端响应加缓存头、缓存在服务器或反向代理，TTL 可配。Nitro 的 `node-server` preset 能缓存整个响应。TTL 过期时先发缓存、页面在后台重新生成；用 `true` 会加一个 `stale-while-revalidate` 头但不带 MaxAge |
| `isr` | `number \| boolean` | 行为同 `swr`，但能把响应加进支持该能力的平台的 **CDN 缓存**（文档说目前 Netlify 或 Vercel）。用 `true` 内容在 CDN 里保留到下次部署 |
| `prerender` | `boolean` | 构建期预渲染这些路由，把它们作为静态资源打进产物 |
| `noScripts` | `boolean` | 关掉站点某些部分的 Nuxt 脚本与 JS 资源提示渲染 |
| `appMiddleware` | `string \| string[] \| Record<string, boolean>` | 定义在 Vue 应用部分（不是 Nitro 路由）的页面路径上，哪些中间件该跑、哪些不该跑 |

几点提醒：

- 这份清单里**没有 `proxy`**。别按旧教程往 routeRules 里写 `proxy` 当作这一页记载的属性——本页九项不含它。
- `swr` 与 `isr` 的差别就一句：**isr 能进 CDN 缓存**（Netlify/Vercel），swr 停在服务器/反代那层；两者都能给 `number`（TTL 秒）或 `boolean`。
- `isr: true` / `swr: true` 与数字的区别在文档里写明了：`isr: true` 内容留到下次部署；`swr: true` 是加一个不带 MaxAge 的 `stale-while-revalidate` 头。

## 组合示例

365 见过这段，这里当作"属性怎么配在一起"的完整参照：

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

内容页预渲染、产品页按需生成后台再验证缓存 1 小时、博客页进 CDN、后台纯客户端、API 加跨域、老地址重定向——一页配齐七种策略，互不干扰。

还有个官方 note：用 `isr` 或 `swr` 的路由**会额外产出 `_payload.json`**，客户端导航读这些缓存 payload 而不是重新取数（payload extraction）。

## `ssr: false` 会瘦服务端包体

一个常被忽略的点：被 `ssr: false` 覆盖的路由只在浏览器渲染，Nuxt 就**把它的页面组件从服务端包体里排除**——前提是该路由到达路径的规则在构建期就能定死，动态路由（如 `pages/products/[id].vue` 归到 `/products/**`）也算。

但有三种情况页面**仍留在服务端包体**：

1. 更精确的规则在纯客户端区域里又开了 SSR（`'/admin/**': { ssr: false }` 配 `'/admin/report': { ssr: true }`）；
2. 页面有 alias、或用绝对路径声明的子路由落在纯客户端区域之外；
3. 页面是父壳、渲染一个仍在服务端渲染的子页面。

官方强调这只是**构建期优化**，"does not change what the server sends to the browser"——不改服务器实际发给浏览器的内容。

## 平台原生规则与一条硬限制

能落到部署平台原生规则上的，Nuxt 会尽量自动映射以获得最佳性能（文档说目前支持 Netlify 和 Vercel）。

反过来有条明确边界：

> Note that Hybrid Rendering is not available when using nuxt generate.

用 `nuxt generate`（纯静态、无服务器）时 hybrid 用不了——因为它靠服务端注册中间件和缓存层，`nuxt generate` 没有那台一直在跑的服务器。

## 自查

- routeRules 让 Nuxt 自动注册中间件 + 用 Nitro 缓存层包住路由。
- 官方属性九项：redirect/ssr/cors/headers/swr/isr/prerender/noScripts/appMiddleware；**清单里没有 proxy**。
- `swr` 缓存到服务器/反代，`isr` 多一层能进 CDN（Netlify/Vercel）；两者可给数字(TTL)或 true，语义有别。
- `ssr: false` 覆盖的路由其页面组件从服务端包体排除（构建期优化），三种情况除外。
- hybrid 依赖服务端，`nuxt generate` 下不可用。

## 下一篇

下一篇 368 起从"渲染形态"转到"生命周期"：Nuxt 把一次访问切成服务端七步、客户端五步，先看清这条主轴和 365 那张"谁在哪跑"的关系。

## 本文依据

- Hybrid Rendering 机制、Route Rules 九项属性清单与各自描述、`isr`/`swr` 的 `_payload.json` note、`ssr: false` 对服务端包体的影响与三种例外、Netlify/Vercel 原生映射、`nuxt generate` 下 hybrid 不可用的 `::important`（`guide/concepts/rendering`）。
