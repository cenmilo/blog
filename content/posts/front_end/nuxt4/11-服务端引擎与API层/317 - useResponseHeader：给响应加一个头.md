---
url: /posts/07a62c736f8a46e753fedeba20943047/
title: 317 - useResponseHeader：给响应加一个头
date: 2027-01-25T11:00:00+08:00
updated: 2027-01-25T11:00:00+08:00
author: cmdragon

summary:
  摘要：前面几篇读请求（头、URL、event），这一篇写响应的一个头。useResponseHeader('X-My-Header') 返回一个可写的 ref，赋 .value 就把这个响应头加到当前页面这次响应上——这是"按页设响应头"的官方路子；想全站每页都加同一个头，就在路由中间件里设。它是 v3.14+ 才有的组合式，可用在 pages、components、plugins。这一篇讲它的 value 赋值写法、两种典型落点（单页 vs 中间件全局），以及它和 setResponseStatus（307，设的是状态码）的分工。

categories:
  - 前端开发

tags:
  - nuxt4
  - useResponseHeader
  - 响应头
  - 中间件
  - SSR
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

314–316 都在"读"这次请求。这一篇反过来，给这次**响应**加一个头。工具是 `useResponseHeader`。

## 它是什么、怎么用

官方定位：

> Use useResponseHeader to set a server response header.

版本门槛（important）：

> This composable is available in Nuxt v3.14+.

可用位置："within your pages, components, and plugins"。基本写法是"拿到一个可写 ref、给它赋 `.value`"：

```ts
// 设一个自定义响应头
const header = useResponseHeader('X-My-Header')
header.value = 'my-value'
```

`useResponseHeader('名字')` 返回的是**代表那个响应头的 ref**，你对 `.value` 赋值，就等于把这行头加进这次响应。传的是头名、赋的是头值。

## 落点一：按页设响应头

官方第一个例子就是"某个页面要带一个特定响应头"：

```vue [app/pages/test.vue]
<script setup>
// pages/test.vue
const header = useResponseHeader('X-My-Header')
header.value = 'my-value'
</script>

<template>
  <h1>Test page with custom header</h1>
  <p>The response from the server for this "/test" page will have a custom "X-My-Header" header.</p>
</template>
```

这是 useResponseHeader 的主场景：**每个页面设不同的响应头**（比如某个页面要单独设缓存策略头、限流头之类），在页面的 setup 里写一次就行。

## 落点二：中间件里设全站响应头

官方第二个例子把它放进路由中间件——"for example in Nuxt middleware to set a response header for all pages"：

```ts [app/middleware/my-header-middleware.ts]
export default defineNuxtRouteMiddleware((to, from) => {
  const header = useResponseHeader('X-My-Always-Header')
  header.value = `I'm Always here!`
})
```

因为中间件每次导航都跑（290、292），在里面设的头就等于"每页响应都带上这个头"。想全站统一一个响应头，用这一种；想按页不同，用第一种。

（注：中间件里能用的组合式有限制——`useRoute()` 不能用（290），但 `useResponseHeader` 这类 SSR 侧设响应的工具官方示例直接放在中间件里用了。）

## 和 setResponseStatus 的分工

这一片里有三个"设响应"的东西，划清别混：

| 要设什么 | 用什么 |
| --- | --- |
| 响应头 | `useResponseHeader(name)` + 赋 `.value`（本篇） |
| 状态码 + 可选 statusText | `setResponseStatus(event, code, statusText?)`（307） |
| 抛出一个带状态的错误 | `createError({ status })`（308） |

都是"影响这次响应"，但作用在不同字段：头归 useResponseHeader，状态码归 setResponseStatus，错误归 createError。而且它们同样依赖服务端那一趟（纯客户端设响应头/状态码没意义，和 307"浏览器里没有效果"同理——响应头也是服务端发响应时写进头的）。

## 自查

- 按页设响应头 → 页面 setup 里 `useResponseHeader('name').value = 'val'`；全站设 → 路由中间件里这么设。
- 它是 ref 语义：传头名、赋 `.value` 设头值。
- v3.14+ 才有，用在 pages/components/plugins。
- 分清楚：设头 useResponseHeader、设状态码 setResponseStatus、抛错误 createError，三个各管响应的一处字段。

## 下一篇

下一篇 318 讲 `useRequestFetch`：SSR 取数时把这次请求的上下文和头自动转发过去，和手动 useRequestHeaders 转发的对比。

## 本文依据

- useResponseHeader（`api/composables/use-response-header`）：定位句；important "This composable is available in Nuxt v3.14+."；"set any server response header within your pages, components, and plugins."；`useResponseHeader('X-My-Header')` + `header.value = 'my-value'` 基本用法；`test.vue` 按页示例；`my-header-middleware.ts` 中间件示例（"for example in Nuxt middleware to set a response header for all pages"）。
