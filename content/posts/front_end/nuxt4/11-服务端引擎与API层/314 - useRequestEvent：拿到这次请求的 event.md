---
url: /posts/33ae0039be41eec7e5e745c7460de473/
title: 314 - useRequestEvent：拿到这次请求的 event
date: 2027-01-24T11:00:00+08:00
updated: 2027-01-24T11:00:00+08:00
author: cmdragon

summary:
  摘要：useRequestEvent 是在应用侧（setup、插件、中间件）拿到"这次入站请求的底层 event"的入口。它只能在 Nuxt 上下文里用，且在浏览器里返回 undefined——所以取用要先判空。返回类型由配置的服务端构建器决定：默认 @nuxt/nitro-server 下是 h3 v1 的 H3Event（所以官方示例读的是 event.path 而不是 event.url）。这一篇讲它的定位、浏览器返回 undefined 这条硬事实、以及为跨构建器可移植而存在的那份 RequestEvent 通用形状（req/url/res/context 四属性）。

categories:
  - 前端开发

tags:
  - nuxt4
  - useRequestEvent
  - 请求事件
  - h3
  - SSR
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

311、313 讲的都是"服务端代码里的 import 面"。这一篇回到**应用侧**：在一个组件/插件/中间件里，想读"当前这次请求"的底层信息，用 `useRequestEvent`。

## 它是什么、在哪能用

官方定位：

> Access the incoming request event with the useRequestEvent composable.

限定：

> Within the [Nuxt context] you can use `useRequestEvent` to access the incoming request.

和 `useRuntimeConfig`、`useAppConfig` 一样绑在 Nuxt 上下文四类位置上（295）。用法：

```ts
// 取底层请求 event
const event = useRequestEvent()

// 读取站请求的 path
const path = event?.path

// 读一个请求头
const userAgent = event?.headers.get('user-agent')
```

注意示例里全程用 `event?.` 可选链——因为下一条：它可能没有值。

## 关键硬事实：浏览器里返回 undefined

官方 tip 一句话：

> In the browser, `useRequestEvent` will have no effect.

（原文："In the browser, `useRequestEvent` will return `undefined`."）

原因不复杂：入站请求的 event 是**服务端处理这次 SSR 请求时**才有的对象；到了浏览器，已经没有"服务端正在处理我的这次请求"这回事了，`useRequestEvent()` 自然拿不到，返回 `undefined`。所以在两端都会跑的代码里，取到 event 后要判空：`if (event) { ... }`（307 的 `setResponseStatus` 用的正是同一个 `useRequestEvent()`、同一个判空套路）。

## 返回类型看服务端构建器

官方签名：

```ts
function useRequestEvent (nuxtApp?: NuxtApp): NuxtRequestEvent | undefined
```

`NuxtRequestEvent` 的形状由你配的构建器决定，官方说得很具体：

> `NuxtRequestEvent` is the event in the shape your configured [server builder] provides. With the default `@nuxt/nitro-server`, that is h3 v1's `H3Event`, which is why the example above reads `event.path` rather than `event.url`.

这条解释了一个细节：默认构建器下 event 是 h3 的 `H3Event`，所以读路径是 `event.path`，**不是 `event.url`**。你要是按 h3 之外的直觉写 `event.url`，在默认构建器下就对不上。

## 为可移植存在的 RequestEvent

当没有服务端构建器贡献具体 event 类型时，它退到一份 Web 标准的通用 `RequestEvent`——官方列了四个属性：

| 属性 | 类型 | 说明 |
| --- | --- | --- |
| `req` | `Request` | 入站请求，含 `headers`、`method` 和 body |
| `url` | `URL` | 解析后的请求 URL |
| `res` | `{ status?, statusText?, headers }` | 要发出去的状态与响应头 |
| `context` | `RequestEventContext` | 每请求状态，Nuxt 自己的态在 `context.nuxt` 下 |

官方给这条的目的很明确——"Code that should work whichever server builder is configured (a module's server handler, for example) should only read these properties."。也就是**要跨构建器可移植（比如模块的服务端 handler），就只读 `req`/`url`/`res`/`context` 这四个**，别依赖 h3 专有的形状。这跟 313 讲的 `nuxt/server` 可移植是同一动机。

顺带把两个概念连上：280 讲过 `nuxtApp.ssrContext` 也带一个 `event`（只在服务端）；`useRequestEvent()` 是应用侧拿同一个"这次请求的 h3 event"的组合式入口。一个挂在 nuxtApp 上、一个用组合式取，指向的是同一趟服务端请求。

## 自查

- 在 setup/插件/中间件里要读这次入站请求 → `useRequestEvent()`，但它在浏览器返回 `undefined`，取用先判空（`event?.path`）。
- 默认 `@nuxt/nitro-server` 下 event 是 h3 的 `H3Event`，读路径是 `event.path` 不是 `event.url`。
- 要跨构建器可移植的代码，只读 `req`/`url`/`res`/`context` 四个通用属性。
- 它只能在 Nuxt 上下文里调（295 四类位置）。

## 下一篇

下一篇 315 讲读请求头的两个组合式：`useRequestHeader` 读单个头、`useRequestHeaders` 读全部或指定几个，两者在浏览器分别返回 undefined / 空对象。

## 本文依据

- useRequestEvent（`api/composables/use-request-event`）：定位句；"Within the Nuxt context you can use useRequestEvent to access the incoming request."；示例（`useRequestEvent()`、`event?.path`、`event?.headers.get('user-agent')`）；tip "In the browser, useRequestEvent will return undefined."；签名 `useRequestEvent(nuxtApp?): NuxtRequestEvent | undefined`；"NuxtRequestEvent is the event in the shape your configured server builder provides. With the default @nuxt/nitro-server, that is h3 v1's H3Event, which is why the example above reads event.path rather than event.url."；RequestEvent 四属性表（req/url/res/context）与"only read these properties"（可移植）。
