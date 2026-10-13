---
url: /posts/93b91eb37624180a757067d518c59a90/
title: 387 - 渲染模式与取数：什么数据一定进 payload
date: 2027-02-18T09:00:00+08:00
updated: 2027-02-18T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：SSR 下浏览器能复用服务端取到的数据，靠的是一个叫 payload 的东西。官方口径很明确：**`useAsyncData` 和 `useFetch` 的取数结果、以及 `useState` 的应用状态**会被序列化进 payload，水合时客户端拿它、不再重取。这解释了为什么 setup 里裸用 `$fetch` 会二次请求（372/373 那个 hydration 隐患）——`$fetch` 没有这套转发。渲染模式会改变这条链：`experimental.payloadExtraction` 决定 payload 是内联进 HTML 还是抽成 `_payload.json`，默认 `true`（cv5 下 `'client'`），而 `ssr: false` 会把它强制成 `false`。`pick`/`transform` 只裁剪进 payload 的字段，不减少最初取回的数据。

categories:
  - 前端开发

tags:
  - nuxt4
  - payload
  - useFetch
  - useState
  - payloadExtraction
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

372 说 `await useFetch` 不改服务端 HTML，因为 Nuxt 会等取数完再序列化页面。373 说服务端取的数据要转发给客户端避免重复请求。这两件事的连接点就是 payload。这一篇把"什么数据进 payload、渲染模式怎么影响它"讲清。

## payload 是什么

取数那一节给的定义：

> The payload is a JavaScript object accessible through `useNuxtApp().payload`. It is used on the client to avoid refetching the same data when the code is executed in the browser during hydration.

一个挂在 `useNuxtApp().payload` 上的 JS 对象，客户端水合时用它避免重复取数。

## 什么一定进 payload

预渲染那一节的 Payload Extraction 开头一句最准确，它点名了三样：

> When Nuxt renders a page on the server, it serializes the results of your data fetching ([`useAsyncData`] and [`useFetch`]) and app state ([`useState`]) into a payload so the client can hydrate without re-fetching.

**进 payload 的是**：
1. `useAsyncData` 的取数结果
2. `useFetch` 的取数结果
3. `useState` 的应用状态

performance 那一节从另一头印证 `useFetch`/`useAsyncData` 的存在意义：

> To avoid fetching same data twice (once on the server and once on client) Nuxt provides `useFetch` and `useAsyncData`. They ensure that if an API call is made on the server, the data is forwarded to the client in the payload instead of being fetched again.

## 反过来：裸 `$fetch` 不进这套

对照就是 372/373 讲的二次请求成因：在 setup 里用 `$fetch` 取数，服务端取一次、水合时客户端又取一次——因为 `$fetch` "has no such mechanism"（取数页原话），它不往 payload 写、也就不被客户端复用。所以首屏要靠服务端的数据，必须是 `useFetch`/`useAsyncData`/`useState`，不能是裸 `$fetch`。`$fetch` 的位置是"纯浏览器交互（事件驱动）时"用。

## 序列化两分：别当成一回事

数据进 payload 的序列化，官方分成两拨（385 前也拆过，这里从"进 payload"角度串起来）：

- **走 Nuxt payload 的**（`useAsyncData`/`useLazyAsyncData`、`useState`）用 **devalue** 序列化，能带 regex、Date、Map/Set、`ref`/`reactive`/`shallowRef` 等（取数页 "Serializing Data From Server to Client"）。
- **从你的服务端路由 `$fetch`/`useFetch` 拿到的数据**用 **`JSON.stringify`**，只限 JS 基本类型（"Serializing Data From API Routes"），且官方明说 devalue 那套 "**does not apply**" 到这里。

一句"数据进 payload"底下其实是两条不同的序列化通道。

## 渲染模式怎么改这条链

`experimental.payloadExtraction` 决定 payload 放哪，三档（预渲染页 Payload Extraction 节）：

- `'client'`：初始渲染内联进 HTML、客户端导航时抽成 `_payload.json`，首屏不多一次请求。
- `true`：初始渲染和客户端导航都抽成 `_payload.json`（HTML 更小、可被 CDN 缓存，代价首屏多一次请求）。
- `false`：关闭抽取，payload 永远内联、不产 `_payload.json`。

默认 `true`，`compatibilityVersion: 5` 下默认 `'client'`，**`ssr: false` 时强制为 `false`**。也就是说纯客户端那条路根本没有"服务端 payload 转发"可言——没有服务端渲染就没东西可转。这和 382 的"`ssr: false` 是空壳 `#__nuxt`"是同一个后果的两面。

## `pick` 与 `transform` 只裁 payload

想少进点数据，`pick`（只挑要的字段）、`transform`（改形状）能减小 payload，但官方 `::note` 说清边界：

> Both `pick` and `transform` don't prevent the unwanted data from being fetched initially. But they will prevent unwanted data from being added to the payload transferred from server to client.

两者**不阻止最初把数据取回来**，只阻止不要的数据被加进"服务端→客户端"的 payload。

## 自查

- 一定进 payload 的：`useAsyncData`/`useFetch` 的结果 + `useState` 的状态；裸 `$fetch` 不进，这是二次请求的根因。
- 序列化两分：Nuxt payload 用 devalue（可带 Date/Map/ref 等）、从 API 路由 `$fetch`/`useFetch` 来的用 JSON.stringify（仅基本类型），devalue 那套不适用于后者。
- `experimental.payloadExtraction`：`'client'`/`true`/`false`，默认 `true`（cv5 `'client'`），`ssr: false` 强制 `false`。
- `pick`/`transform` 只裁进 payload 的字段，不减最初取回的数据。

## 下一篇

下一篇 388 从 SEO 角度做取舍：SSR 让爬虫拿到全文、预渲染更早可索引、`ssr: false` 丢 SEO；而 head/meta 得由 Unhead 渲进 HTML 才生效。

## 本文依据

- payload 定义（`useNuxtApp().payload`）、"serializes the results of your data fetching (useAsyncData and useFetch) and app state (useState) into a payload"、`useFetch`/`useAsyncData` 防二次请求（performance "forwarded to the client in the payload"）、`$fetch` has no such mechanism（`getting-started/data-fetching`、`guide/best-practices/performance`）；devalue vs JSON.stringify 两分、`payloadExtraction` 三档与默认、`ssr:false` 强制 false、`pick`/`transform` 边界 `::note`（`getting-started/data-fetching`、`getting-started/prerendering`）。
