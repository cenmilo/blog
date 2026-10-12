---
url: /posts/7938e6a6f4ca31d3bf5d99cf6b28f286/
title: payload 体积控制：renderJsonPayloads 与 payloadExtraction
date: 2027-03-10T10:00:00+08:00
updated: 2027-03-10T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：首屏 HTML 里那一大段 `__NUXT_DATA__` 是给客户端水合用的载荷：Nuxt 把取数结果（`useAsyncData`、`useFetch`）和应用状态（`useState`）序列化进去，好让浏览器不用再取一遍。体积上有两处控制——`experimental.payloadExtraction` 决定它留在 HTML 里还是抽成 `_payload.json`（三档：`'client'`、`true`、`false`，默认 `true`，`compatibilityVersion: 5` 下默认变 `'client'`，`ssr: false` 时被强制为 `false`），`experimental.renderJsonPayloads` 决定 JSON 载荷能不能还原复杂类型（默认开）。还有一处常被误当成体积开关：`purgeCachedData` 管的是路由跳转时清不清内存里的缓存，官方给的理由是防内存泄漏和保证数据新鲜。

categories:
  - 前端开发

tags:
  - nuxt4
  - payload
  - 水合
  - 性能
  - 缓存
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

查看源码，页面正文没几行，`<script type="application/json" id="__NUXT_DATA__">` 那一坨反倒比正文大。你想把它压下去，但要先搞清它是什么、哪几个开关各管哪一段。

## 它是什么

官方在预渲染那一页给了完整的定义：

> When Nuxt renders a page on the server, it serializes the results of your data fetching (`useAsyncData` and `useFetch`) and app state (`useState`) into a payload so the client can hydrate without re-fetching.

三件事在这句里：**来源是取数结果加应用状态**、**用途是客户端不用重取**、**它被序列化进渲染结果**。

再补一条序列化实现，因为后面要用到它：

> Payloads are serialized with devalue, so custom types (such as class instances) need payload plugins with custom reducers and revivers to survive the round trip.

## 第一处控制：留在 HTML 里还是抽成文件

`experimental.payloadExtraction` 决定这件事。官方给的是三档，两页用词一致：

| 取值 | 官方原文要点 |
| --- | --- |
| `'client'` | 首屏渲染时载荷内联在 HTML 里，客户端导航用的抽成 `_payload.json`；**首次加载不多发请求** |
| `true` | 首屏和客户端导航都用独立的 `_payload.json`；**HTML 更小、载荷文件能被 CDN 缓存，代价是首屏多发一次请求** |
| `false` | 完全不抽；载荷始终内联在 HTML 里，**不生成 `_payload.json`** |

默认值这一句要把三个条件一起记：

> The default is `true`, or `'client'` when `compatibilityVersion: 5` is set. It is forced to `false` when `ssr: false` is set.

也就是：**默认 `true`；`compatibilityVersion: 5` 下默认变 `'client'`；`ssr: false` 时被强制成 `false`**（客户端渲染没有服务端渲染结果，自然没有可抽的载荷）。

文件什么时候生成，官方分了两类：

> - Prerendered routes generate their payload file at build time.
> - Routes using ISR or SWR caching generate their payload file when the route is first rendered, even on a hybrid (non-static) site.

**预渲染的路由在构建期生成；用 ISR/SWR 缓存的路由在第一次渲染时生成**，混合渲染站点上也一样。

## 第二处控制：能不能还原复杂类型

`experimental.renderJsonPayloads` 一句定性加一句默认状态：

> Allows rendering of JSON payloads with support for revivifying complex types.
> This flag is enabled by default, but you can disable this feature:

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    renderJsonPayloads: false,
  },
})
```

这一项管的是"复杂类型能不能活过来"，不是"载荷多大"。**它默认开着，官方没写关掉之后是什么形态、也没写什么场景该关**。上面那句 devalue 的说明与这一项的关系，文档也没把两者串起来讲——所以别拿它当压缩手段。

真正跟体积有关的第三个成员在 449 那篇（`stripNeverHydratedData`）：它给永不水合组件里的取数调用默认加上 `serialize: false`，把数据从 `__NUXT_DATA__` 里删掉。单个调用想缩体积，官方给的是 `useAsyncData` 的 `serialize: false` 与 `pick`（243 那篇讲过）。

## 常被误当成体积开关的那个

`experimental.purgeCachedData` 的定性是：

> Whether to clean up Nuxt static and asyncData caches on route navigation.

下一句说明了它清的是哪些、为什么清：

> Nuxt will automatically purge cached data from `useAsyncData` and `nuxtApp.static.data`. This helps prevent memory leaks and ensures fresh data is loaded when needed, but it is possible to disable it.

三个要点：**发生在路由跳转时**、**对象是 `useAsyncData` 与 `nuxtApp.static.data` 的缓存**、**理由是内存泄漏与数据新鲜**。默认开着，设成 `false` 是"关掉清理"。

所以它影响的是**运行时的内存占用和取数是否重跑**，不是发给浏览器的字节数。把它加进"payload 瘦身"清单是个常见误会。

顺带说清 `useNuxtData` 在这一层的角色：它读的就是这些缓存里的当前值，官方要求"ensure that the data-fetching composable has been called with an explicitly provided key"，返回的 `data` 是**会在缓存变化时自动更新的 ref**——这是 254 那篇讲的乐观更新与级联更新的基础，和体积无关。

## 三档怎么选

官方在这一页给了三条实际后果，正好是决策依据：

> - On a fully static site, client-side navigation reuses the data captured at build time, so data can be stale until the next rebuild.
> - For ISR/SWR routes, CDNs can cache payload files alongside the HTML, improving client-side navigation performance for cached routes. Dynamic routes like `pages/[...slug].vue` can opt in with glob patterns such as `'/**': { isr: true }`.
> - Payloads are serialized with devalue, so custom types ... need payload plugins ...

| 你的情况 | 官方依据支持的做法 |
| --- | --- |
| 全站预渲染、在意首屏 HTML 大小 | 默认 `true`：HTML 更小、载荷文件可被 CDN 缓存，代价是首屏多一次请求 |
| 全站预渲染、不想多发那一次请求 | `'client'`：首屏内联，导航时才取文件 |
| 数据会过时但重建不频繁 | 记住第一条后果：静态站的客户端导航复用的是**构建期**的数据 |
| 动态路由想让 CDN 连载荷一起缓存 | 用 `routeRules` 的 `isr`/`swr`（glob 写法），配套才有 payload 文件 |
| 载荷里有 class 实例之类 | 三条都要看：devalue 不是全还原，需要 payload plugin |

## 自查

- 载荷来源是取数结果加 `useState`，作用是让客户端不重取；序列化用 devalue，自定义类型要 payload plugin。
- `payloadExtraction` 三档默认 `true`，cv5 下默认 `'client'`，`ssr: false` 强制 `false`；文件生成时机是构建期（预渲染）或首次渲染（ISR/SWR）。
- `renderJsonPayloads` 默认开、管的是复杂类型还原，**不是体积开关**，关掉的后果文档未写。
- `purgeCachedData` 清的是内存里的 `useAsyncData`/`nuxtApp.static.data` 缓存，理由是内存与新鲜度。
- 真要缩体积，走 `serialize: false`、`pick`，以及 449 的 `stripNeverHydratedData`。

## 下一篇

下一篇 449 讲 `stripNeverHydratedData`：它给用 `hydrate-never` 懒水合的组件里的取数调用默认加 `serialize: false`，因为那些组件在客户端永远不水合、数据本来就不需要；官方同时提醒同一个 key 被会水合的组件共用时，`serialize` 必须保持一致，开发模式下会警告不一致。

## 本文依据

- 载荷定义句（serializes data fetching results and app state so the client can hydrate without re-fetching）、`payloadExtraction` 三档原文与默认值句（含 cv5 与 `ssr: false` 强制）、两条生成时机（构建期 / ISR-SWR 首次渲染）、客户端导航复用 `_payload.json` 的说明、三条实际后果（静态站数据陈旧、CDN 可缓存载荷文件与 `'/**': { isr: true }` glob、devalue 与 payload plugin）（`getting-started/prerendering`）
- `## payloadExtraction` 一节三档措辞与默认值、指向预渲染页的 read-more；`## renderJsonPayloads` 一节定义原文与 "enabled by default" 措辞与示例；`## purgeCachedData` 一节定性、"purge cached data from `useAsyncData` and `nuxtApp.static.data`"与内存泄漏/新鲜数据理由句、默认开启与示例、指向 PR #31379 的 read-more；`stripNeverHydratedData` 一节对 `serialize: false` 的引用（`guide/going-further/experimental-features`）
- `useNuxtData` 需显式 key、返回自动更新的 ref、用于复用已取数据与乐观更新（`api/composables/use-nuxt-data`）
