---
url: /posts/e5c8a0d3634dc0c19f33f2e2650abc0a/
title: 280 - ssrContext：只在服务端有的那份上下文
date: 2027-01-13T10:00:00+08:00
updated: 2027-01-13T10:00:00+08:00
author: cmdragon

summary:
  摘要：nuxtApp.ssrContext 是服务端渲染过程中生成的那份上下文，只在服务端存在、浏览器里拿不到。它对外开放三个属性：url（当前请求的 URL）、event（h3 的请求事件，能读请求和响应）、payload（那份服务端到客户端的数据）。这一篇讲清"它只在一侧有"这条边界、三个属性各自能干什么，并给一条最实用的判据——想知道自己此刻在服务端还是客户端，除 import.meta.server 外，ssrContext 在不在也是同一个信号。别在客户端代码里访问它。

categories:
  - 前端开发

tags:
  - nuxt4
  - ssrContext
  - nuxtApp
  - h3
  - 服务端
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`nuxtApp.ssrContext` 是 Nuxt 应用实例上一个"只在服务端有"的成员。理解它，先理解它为什么只在一侧存在——因为它本身就是"服务端渲染这次请求"的上下文，浏览器里根本没有"服务端渲染"这回事，自然也没有它。

## 定义与三个属性

use-nuxt-app 页对它的说明：

> `ssrContext` is generated during server-side rendering and it is only available on the server side.
> Nuxt exposes the following properties through `ssrContext`:
> - `url` (string) - Current request url.
> - `event` (h3 request event) - Access the request & response of the current route.
> - `payload` (object) - NuxtApp payload object.

逐个展开：

**`url`（字符串）**：当前这次请求的 URL。服务端渲染时你手上那个请求的路径，就存在这儿。

**`event`（h3 请求事件）**：这个最有用。h3 是 Nuxt 服务端底下那个 HTTP 框架，`event` 就是它表示"一次请求+响应"的对象。官方描述 "Access the request & response of the current route."——通过它能读入站请求（头、cookie、方法等）、也能影响响应。这一篇不展开 h3 细节，但你要知道：**从 `ssrContext.event` 拿到的，就是服务端路由 handler 里那个同款的 `event`**（210-249 讲 `useRequestHeaders`/`useRequestFetch` 转发请求头时用的就是它背后的那份信息）。

**`payload`（对象）**：上一节（279）那份 `payload` 的又一路入口。官方在 279 里也点过这句——"When accessing the same `payload.data` from `ssrcontext`, you can access the same value on the server side as well." 也就是 `ssrContext.payload` 和服务端那份 `nuxtApp.payload` 是同一批数据，服务端渲染阶段可以从这里读要写进 HTML 的内容。

## 一条边界：客户端拿不到

全篇最该记住的一句话就是那句 "it is only available on the server side."。它带来的直接约束：

- 不要在客户端会执行到的代码里读 `ssrContext.url` / `ssrContext.event`——浏览器侧没有它，读出来是 `undefined`，再往下 `.url` 就报错。
- 想跨"两端都能跑"的插件/组件安全使用，先判侧：

```ts
const nuxtApp = useNuxtApp()
if (import.meta.server) {
  // 这里才碰 nuxtApp.ssrContext
  const url = nuxtApp.ssrContext?.url
}
```

`import.meta.server` / `import.meta.client` 是 Nuxt 给的"我在哪一侧"的标准判据（贯穿全系列，SSR 那一路反复用）。`ssrContext` 存不存在，本质是同一个事实的另一种表现——服务端渲染时才生成它。

这里也顺带说明一个常见疑问：**插件默认两端都跑**（272、273 都提过）。所以一个不加分辨就访问 `ssrContext` 的插件，到了客户端那一趟就会空。要只在服务端碰它，要么包 `if (import.meta.server)`，要么把整个插件写成 `.server.ts` 后缀（只在服务端加载）。

## 用在哪：读请求、改响应

`ssrContext.event` 是"服务端渲染这次请求"的把手，典型用途：

- 在插件/服务端逻辑里读某个入站信息，而又不方便用专门的组合式（`useRequestHeaders`、`useRequestURL` 这些是更常用的包装，见 210-249 那几篇；`ssrContext.event` 是更底层的直接入口）。
- 需要拿 `event` 去调只对 h3 event 工作的服务端工具时，从这儿取。

要强调的是：多数"读请求头/URL"的需求，官方更推荐用专门的组合式（`useRequestURL()` 返回 URL 对象、`useRequestHeaders([...])` 转发头），而不是手动扒 `ssrContext.event`。`ssrContext` 的价值在于它是那层**更底层的、把整份请求/响应交给你**的入口，在需要写响应或访问 event 全貌时才有必要下沉到它。

## ssrContext 和 payload 的关系

两个都在讲"服务端那份数据"，别混：

- `nuxtApp.payload`：这份"服务端→客户端"的数据本身（279）。两端都有它的痕迹（客户端水合时读的是它接过来的结果）。
- `nuxtApp.ssrContext`：只在服务端有的**渲染上下文**，其中 `ssrContext.payload` 指向的正是上面那份 payload（服务端视角）。

所以 `ssrContext` 是"服务端这次渲染的总把手"，payload 是"它要往下传的数据"。`url`、`event` 这两个键是 `ssrContext` 独有、payload 没有的——它们只在"我正在服务端处理一次具体请求"时才有意义。

## 自查

- `ssrContext` 只在服务端有；客户端会跑到的代码里读它要 `undefined`，先 `import.meta.server` 判侧或把插件写成 `.server.ts`。
- 三个属性：`url`（当前请求 URL）、`event`（h3 请求事件，读请求/写响应的底层入口）、`payload`（服务端视角的那份 payload）。
- 只是想读请求头/URL，优先 `useRequestHeaders`/`useRequestURL`；需要 event 全貌或写响应，才下沉到 `ssrContext.event`。
- 别把它和 payload（279）搞混：ssrContext 是渲染上下文，payload 是它携带的数据。

## 下一篇

下一篇 281 讲 `reloadNuxtApp`：主动整页硬重载，以及它默认 10 秒的 ttl 防重载循环、path/force/persistState 三个选项，和 263 的 restoreState 怎么配合。

## 本文依据

- useNuxtApp（`api/composables/use-nuxt-app`）`ssrContext` 一节："ssrContext is generated during server-side rendering and it is only available on the server side." 及 `url`/`event`/`payload` 三属性描述（"Current request url."、"Access the request & response of the current route."、"NuxtApp payload object."）；`payload` 一节里 "When accessing the same `payload.data` from `ssrcontext`, you can access the same value on the server side as well."
