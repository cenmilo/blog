---
url: /posts/628dcc755162f13f092f5d330f3f97a4/
title: Suspense 与异步页面：谁在等谁
date: 2027-02-13T09:00:00+08:00
updated: 2027-02-13T09:00:00+08:00
author: cmdragon

summary:
  摘要：`await useFetch(...)` 到底在"等"什么？官方说得很清楚：Nuxt 底层用 Vue 的 `<Suspense>` 挡住导航，直到异步数据就绪才让视图切过去。但"等"分两端——服务端那一趟 Nuxt 本来就要等取数 resolve 才把 HTML 序列化出去，所以 `await` **根本不改变服务端渲染出的 HTML**；`await` 改变的是你自己 `<script setup>` 里后续代码的执行时机、以及客户端导航时挡不挡。不 await、`lazy`、`await` 三者各有各的表现，官方还专门警告"`await` 一个 `lazy` 调用会给你意外结果"。这一篇把"谁在等谁"讲透。

categories:
  - 前端开发

tags:
  - nuxt4
  - Suspense
  - useFetch
  - await
  - lazy
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

368 里服务端第 6 步用 `useFetch`/`useAsyncData` 取数，客户端第 4 步才挂载水合。中间那句"`await`"其实决定了页面导航时用户看到什么。很多人以为 `await` 是让服务端多等一会儿——恰恰相反。这一篇把"到底谁在等谁"说清。

## Nuxt 用 `<Suspense>` 挡导航

官方在取数那页的 Suspense 小节给的定义：

> Nuxt uses Vue's `<Suspense>` component under the hood to prevent navigation before every async data is available to the view. The data fetching composables can help you leverage this feature and use what suits best on a per-call basis.

也就是：Nuxt 底层套了 Vue 的 `<Suspense>`，在异步数据没到齐之前**不让视图切过去**。它还挂了 371 里那两个钩子——`app:suspense:resolve`、以及 `NuxtPage` 里 Suspense 的 `page:start`（pending）/ `page:finish`（resolved）。想让用户感知到"正在切页"，官方建议加 `<NuxtLoadingIndicator>` 做导航间的进度条。

## `await` 不改变服务端 HTML

这是最反直觉、也是官方写得最直白的一条（"A note on `await`"）：

> The `await` does **not** change the server-rendered HTML. During server rendering, Nuxt waits for the request to resolve before serializing the page either way (`<Suspense>`, and `onServerPrefetch` under the hood), so the fully populated result is always sent to the browser.

拆开：服务端那一趟，不管你 `await` 与否，Nuxt 都会**等取数 resolve 了再把页面序列化出去**——底层靠的就是 `<Suspense>` 和 `onServerPrefetch`。所以首屏 HTML 永远带满数据。写不写 `await`，服务端产出的 HTML 一模一样。

## 那 `await` 改变什么

官方接着说，`await` 改变的是"你自己代码的下一步"和"客户端导航的表现"：

- **带 `await`**：执行暂停直到数据就绪，之后代码可以假定 `data` 已填好。客户端导航时，**这会阻塞导航直到数据 resolve**——用户留在当前页（可选配 `<NuxtLoadingIndicator>`），然后落到一个填满的页面。这是默认行为。
- **不带 `await`**：请求在后台跑，代码立刻往下走，`data` 先是默认值、请求完成才填上。客户端导航时**立刻切页**，你得自己处理加载/错误状态（用返回的 `status`、`error` refs）。

官方补一句定性："Neither approach is universally better; the right choice depends on the experience you want for that route." 没有哪种一律更好，按这条路由你要的体验选。

## 不 await ≠ `lazy`

表面看"不 await"和 `lazy` 效果像（都不挡导航、都要自己处理 loading），官方专门点出区别：

> `lazy` is an explicit flag that defers the request until the component mounts, whereas simply not awaiting starts the request during setup. Prefer `lazy` ... when you want non-blocking behaviour, since it makes the intent explicit.

差别在**请求何时发起**：不 await 是在 setup 期间就发起、只是不等它；`lazy` 是把请求**推迟到组件挂载时**再发，是显式意图标记。要非阻塞行为，优先 `lazy`（或 `useLazyFetch`/`useLazyAsyncData`），因为它把意图写明白。

## `await` 一个 `lazy` 会翻车

`::warning` 讲了一条最容易踩的组合：

> The `await` and `lazy` are independent, and on the client awaiting a `lazy` function won't have the effect you expect. If you `await` a `lazy` call (for example `await useLazyFetch(...)` or `await useFetch(..., { lazy: true })`), it still blocks server rendering as usual, but on **client-side navigation** the `await` resolves immediately without waiting for the request. `data` will still be at its default value right after the `await`, and you must handle the loading state via `status`. If you actually want the navigation to wait for the data, drop the `lazy` option rather than relying on `await`.

结论：`await useLazyFetch(...)` 里那个 `await` 在客户端导航时**会立刻返回、不等请求**，紧跟其后 `data` 还是默认值——你以为 `await` 挡住了，其实没挡。真要导航等数据，就把 `lazy` 去掉，别指望 `await` 兜住。

## 谁在等谁（一张表）

| 阶段 | 等不等 | 靠什么 |
| --- | --- | --- |
| 服务端渲染 | 等（无论如何都等取数完再出 HTML） | `<Suspense>` + `onServerPrefetch`，与 `await` 无关 |
| 客户端导航 · 带 `await` 非 lazy | 挡导航到数据就绪 | 默认行为 |
| 客户端导航 · 不带 `await` | 不挡，立刻切，data 后填 | 自己处理 `status`/`error` |
| 客户端导航 · `lazy` | 不挡，请求推迟到挂载 | 显式非阻塞标记 |
| 客户端导航 · `await` + `lazy` | `await` 立即返回、没真等 | 易误解，要靠 `status` |

## 自查

- Nuxt 用 `<Suspense>` 挡导航直到异步数据就绪；服务端本来就等取数完才序列化 HTML。
- `await` **不改变服务端 HTML**，只改变后续代码时机与客户端导航挡不挡。
- 带 `await`（非 lazy）挡导航、落到满数据页；不 await 立刻切、自己管加载态。
- `lazy` 把请求推迟到挂载，比"不 await"意图更明确；`await` 一个 `lazy` 调用在客户端导航时会立即返回、不等请求。

## 下一篇

下一篇 373 进入水合专题：先讲清水合是什么、为什么会同时存在"服务端 HTML"和"客户端重建"两套 DOM。

## 本文依据

- 取数页 Suspense 小节（"Nuxt uses Vue's `<Suspense>` component under the hood to prevent navigation before every async data is available"）、NuxtLoadingIndicator note、A note on `await` 的 "The `await` does not change the server-rendered HTML ... (`<Suspense>`, and `onServerPrefetch` under the hood)"、with/without await 两条、"Neither approach is universally better"、lazy 与不 await 的区别句、`::warning`（await+lazy 在客户端立即返回）（`getting-started/data-fetching`）。
- 服务端第 6 步用 useFetch/useAsyncData 取数、客户端挂载水合、`app:suspense:resolve`/`page:start`/`page:finish` 位置（`guide/concepts/nuxt-lifecycle`，钩子见 371）。
