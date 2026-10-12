---
url: /posts/d0c6c8d9436a272102778979500a6bb1/
title: useRequestURL：拿一个两端都对的 URL 对象
date: 2027-01-25T10:00:00+08:00
updated: 2027-01-25T10:00:00+08:00
author: cmdragon

summary:
  摘要：useRequestURL 返回一个标准的 URL 对象，服务端和浏览器两端都能用（不用自己拼 location 或 request.url）。它有一个和多租户/代理相关的坑：在混合渲染 + 缓存策略下，命中 Nitro 缓存返回响应时，所有入站请求头会被丢弃，于是 host 会被读成 localhost。官方给的解法是配 cache.varies，把 host、x-forwarded-host 这类头纳入缓存键。这一篇讲 URL 对象的用法、那条命中缓存丢头的事实、以及 varies 的对症写法。

categories:
  - 前端开发

tags:
  - nuxt4
  - useRequestURL
  - URL对象
  - 混合渲染
  - cache-varies
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

想要"当前这次请求的完整 URL"，SSR 应用里有个两端不一致的问题：浏览器有 `window.location`、服务端没有；服务端有请求的 url、浏览器没有。`useRequestURL` 把这层抹平——两端都返回一个 URL 对象。

## 它返回什么

官方定位：

> `useRequestURL` is a helper function that returns an [URL object](https://developer.mozilla.org/en-US/docs/Web/API/URL/URL) working on both server-side and client-side.

关键词是 **URL 对象**——它有标准属性（`pathname`、`host`、`searchParams` 等，MDN 有全表），不是裸字符串。官方示例（`app/pages/about.vue`）：

```vue [app/pages/about.vue]
<script setup lang="ts">
const url = useRequestURL()
</script>

<template>
  <p>URL is: {{ url }}</p>
  <p>Path is: {{ url.pathname }}</p>
</template>
```

开发环境下渲染结果：

```html
<p>URL is: http://localhost:3000/about</p>
<p>Path is: /about</p>
```

两端都拿到的是同一个语义的 URL 对象，你不用再写 `if (import.meta.server) ... else window.location ...` 那套分支。

## 一个和多租户/缓存相关的坑

官方在这个页面放了一条 important，是这一篇最该记住的东西：

> When utilizing Hybrid Rendering with cache strategies, all incoming request headers are dropped when handling the cached responses via the Nitro caching layer (meaning `useRequestURL` will return `localhost` for the `host`).

拆开理解：

- **混合渲染**：一个站一部分静态、一部分按需服务端渲染（300 讲的 Nitro 能力之一），配合 `swr`/`isr` 这类缓存路由规则。
- 命中 Nitro 缓存层、直接回缓存响应时——**所有入站请求头被丢弃**。
- 请求头没了，`host` 头就没了，于是 `useRequestURL()` 的 `.host` 被读成 **`localhost`**，而不是真实域名。

这在"按域名区分租户"的多租户场景里是真问题：你根据 `url.host` 判断是哪个客户，结果缓存命中时它变成 `localhost`，判断就错了。这也正是 320 一带提到的"isr/swr 路由命中缓存时 host 变 localhost"的同一个根因。

## 解法：把 host 纳入缓存键 cache.varies

官方紧跟给了对症配置：

> You can define the `cache.varies` option to specify headers that will be considered when caching and serving the responses, such as `host` and `x-forwarded-host` for multi-tenant environments.

`cache.varies` 列出的头会参与"缓存与命中判断"——意思是：只要 `varies` 里的头不同，就算不同的缓存条目。把 `host`、`x-forwarded-host` 纳进去，缓存层就会按域名分别缓存/命中，`useRequestURL().host` 也就不会再被压成 `localhost`。

这条写在路由规则或 `nitro` 配置里（`nitro` 配置节下一篇 320 讲）。要点记一句：**用了带缓存的混合渲染、又靠 host 判断——把 host/x-forwarded-host 加进 cache.varies。**

## 和相邻几个取 URL 的入口对齐

这一片里"读这次请求的地址/头"有三个组合式，别混：

- `useRequestURL()` → 一个 URL **对象**，两端可用（本篇）。
- `useRequestHeader(s)`（315）→ 读**请求头**（值），服务端有、浏览器空。
- `useRequestEvent()`（314）→ 整个底层 **event**，默认是 h3 的、读的是 `event.path`。

多数"要当前 URL / 路径 / 域名"的场景，`useRequestURL()` 是最直接、两端语义一致的那个。

## 自查

- 要当前请求的完整 URL/路径/查询，用 `useRequestURL()`（返回 URL 对象，两端一致），不必自己写 server/client 分支。
- 用了 `swr`/`isr` 等缓存混合渲染、又依赖 `host` 做多租户判断：命中缓存会丢全部入站头、`host` 变 `localhost`——配 `cache.varies: ['host', 'x-forwarded-host']`。
- 分清三个入口：URL 对象用 useRequestURL、读头用 useRequestHeaders、拿整个 event 用 useRequestEvent。

## 下一篇

下一篇 317 讲 `useResponseHeader`：在页面或中间件里给这次响应加一个响应头（v3.14+），以及它的 value 写法。

## 本文依据

- useRequestURL（`api/composables/use-request-url`）：定位句 "returns an URL object working on both server-side and client-side."；`about.vue` 示例（`useRequestURL()`、`url.pathname`）与开发环境结果（`http://localhost:3000/about`、`/about`）；important "When utilizing Hybrid Rendering with cache strategies, all incoming request headers are dropped when handling the cached responses via the Nitro caching layer (meaning useRequestURL will return localhost for the host)." 与 "define the cache.varies option to specify headers ... such as host and x-forwarded-host for multi-tenant environments."；MDN URL 实例属性链接。
