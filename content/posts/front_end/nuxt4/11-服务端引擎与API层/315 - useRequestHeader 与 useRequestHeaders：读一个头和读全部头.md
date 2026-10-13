---
url: /posts/8328b8acbd98500584c175efb7feefed/
title: 315 - useRequestHeader 与 useRequestHeaders：读一个头和读全部头
date: 2027-01-25T09:00:00+08:00
updated: 2027-01-25T09:00:00+08:00
author: cmdragon

summary:
  摘要：读入站请求头有两个组合式，差别在"读一个"还是"读多个"。useRequestHeader('authorization') 取单个头的值，浏览器里返回 undefined；useRequestHeaders() 取全部头、useRequestHeaders(['cookie']) 只取指定几个，浏览器里返回空对象。两者都只在服务端那一趟有真值，典型用途是把首屏请求的 authorization/cookie 头转发给后续内部请求。这一篇把两者的取值形状、浏览器侧的不同返回，和官方那两个转发示例落地。

categories:
  - 前端开发

tags:
  - nuxt4
  - 请求头
  - useRequestHeader
  - useRequestHeaders
  - SSR
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上一篇（314）拿到整个 event 后，`event?.headers.get('user-agent')` 也能读头。但那太重了——多数时候你只想读某几个请求头，官方给了两个轻组合式。

## 读一个头：useRequestHeader

官方定位与用法：

> Use useRequestHeader to access a certain incoming request header.

```ts
// 取 authorization 请求头
const authorization = useRequestHeader('authorization')
```

传入一个头名，返回那个头的值。可用位置："within your pages, components, and plugins"。

浏览器侧的行为（tip）：

> In the browser, `useRequestHeader` will return `undefined`.

官方给的判据示例——用 authorization 头做鉴权跳转（放在路由中间件里）：

```ts [app/middleware/authorized-only.ts]
export default defineNuxtRouteMiddleware((to, from) => {
  if (!useRequestHeader('authorization')) {
    return navigateTo('/not-authorized')
  }
})
```

这里没头就返回 falsy、跳去 `/not-authorized`。注意这是**客户端路由中间件**（守体验）；真要护数据仍得服务端再挡一道（292）。

## 读全部或指定几个：useRequestHeaders

官方定位：

> Use useRequestHeaders to access the incoming request headers.

两种取法：

```ts
// 取所有请求头
const headers = useRequestHeaders()

// 只取 cookie 头
const { cookie } = useRequestHeaders(['cookie'])
```

区别：不带参数拿全部头（一个对象），带一个字符串数组只拿指定那几个（还能解构）。

浏览器侧的行为和上面不同——它返回的是**空对象**而不是 undefined（tip）：

> In the browser, `useRequestHeaders` will return an empty object.

所以两个组合式在浏览器侧的"没值"表现不一样：`useRequestHeader`（单数）→ `undefined`；`useRequestHeaders`（复数）→ `{}`。判空方式因此也不同（前者 `if (val)`、后者 `Object.keys(headers).length` 或直接看某个键）。

## 典型用途：把首屏头转发给内部请求

官方给 useRequestHeaders 的例子，是把初始请求的 authorization 头带到 SSR 期间的内部取数请求上：

```vue [app/pages/some-page.vue]
<script setup lang="ts">
const { data } = await useFetch('/api/confidential', {
  headers: useRequestHeaders(['authorization']),
})
</script>
```

这解决的是 210-249 讲过的一个具体问题（211、222）：SSR 阶段服务端用 `$fetch` 打自家接口时，**不会自动带上浏览器这次请求的头**（安全考虑）。所以要在服务端内部请求里保住登录态，就手动把 `authorization` 头塞进去——`headers: useRequestHeaders(['authorization'])` 正是干这个。

这条和下一篇（318）的 `useRequestFetch` 是同一需求的两种解法：`useRequestHeaders([...])` 是"手动挑几个头转发"，`useRequestFetch()` 是"整体自动转发上下文和头"。要精细控制用前者、要省事用后者。

## 两者对照

| | useRequestHeader | useRequestHeaders |
| --- | --- | --- |
| 读什么 | 单个头 | 全部头，或指定几个 |
| 传参 | 头名 | 可选的头名数组 |
| 返回 | 该头的值（字符串） | 一个对象 |
| 浏览器里 | `undefined` | 空对象 `{}` |

共同点：都只在服务端那一趟拿到真值、都可用于 pages/components/plugins。

## 自查

- 读一个头 → `useRequestHeader('name')`（浏览器 `undefined`）；读多个 → `useRequestHeaders(['a','b'])` 或不传参拿全部（浏览器 `{}`）。
- SSR 阶段要把登录态带进内部取数 → `headers: useRequestHeaders(['authorization'])`（211 那个"服务端 $fetch 不带 cookie"问题的解法之一）。
- 精细转发某几个头用这俩；要整体自动转发上下文用 `useRequestFetch`（318）。
- 客户端路由中间件里判头只护体验，数据保护要在服务端（292）。

## 下一篇

下一篇 316 讲 `useRequestURL`：返回一个两端都能用的 URL 对象，以及那个"混合渲染命中缓存时 host 变成 localhost"的坑与 `cache.varies` 解法。

## 本文依据

- useRequestHeader（`api/composables/use-request-header`）：定位句；`useRequestHeader('authorization')`；tip "In the browser, useRequestHeader will return undefined."；`authorized-only.ts` 中间件示例（无头 navigateTo('/not-authorized')）。
- useRequestHeaders（`api/composables/use-request-headers`）：定位句；`useRequestHeaders()` 取全部、`const { cookie } = useRequestHeaders(['cookie'])` 取指定；tip "In the browser, useRequestHeaders will return an empty object."；`some-page.vue` 示例（`useFetch('/api/confidential',{ headers: useRequestHeaders(['authorization']) })`）。
- useRequestFetch（`api/composables/use-request-fetch`）：SSR 手动转发头的另一解（见 318）。
