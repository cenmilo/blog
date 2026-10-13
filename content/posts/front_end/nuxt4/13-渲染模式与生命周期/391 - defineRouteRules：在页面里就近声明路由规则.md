---
url: /posts/0e94165ab7d85e361a5655760c64b13d/
title: 391 - defineRouteRules：在页面里就近声明路由规则
date: 2027-02-19T10:00:00+08:00
updated: 2027-02-19T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：routeRules 平时写在 `nuxt.config`（367/379），但"这一页要怎么渲染"其实和页面本身最相关。`defineRouteRules` 让你在页面文件里就近声明路由规则，编译时被翻译成对应的 `routeRules` 条目——页面里 `defineRouteRules({ prerender: true })` 等价于在 config 写 `routeRules: { '/': { prerender: true } }`。它是实验特性，要先开 `experimental.inlineRouteRules`。映射规则：`foo/bar.vue` → `/foo/bar`；`foo/[id].vue` → `/foo/*`；有限可选段（`/:locale(en|fr)/about`）→ 每个备选一条规则。但带正则参数（`/:id(\d+)`）、部分段（`/prefix-:id`）、可重复段（`/:slug+`）这类没法转成 route rule 模式的，规则不生效、构建期会警告，那种得回 `nuxt.config` 的 `routeRules` 显式写。

categories:
  - 前端开发

tags:
  - nuxt4
  - defineRouteRules
  - routeRules
  - 实验特性
  - 预渲染
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

367/379 的 routeRules 都写在 `nuxt.config` 里集中管。但"这页要不要预渲染、要不要关 SSR"这种决策，逻辑上属于那一个页面。`defineRouteRules` 就是把这个声明挪到页面文件里、就近写。

## 它是什么

官方定义：

> Define route rules for hybrid rendering at the page level.

在页面里就近声明路由规则。用法：

```vue [app/pages/index.vue]
<script setup lang="ts">
defineRouteRules({
  prerender: true,
})
</script>

<template>
  <h1>Hello world!</h1>
</template>
```

官方给出它会被翻译成什么：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    '/': { prerender: true },
  },
})
```

也就是说 `defineRouteRules` 里的对象，被原样并进这一页对应路由的 `routeRules`。里面能用的键就是 367 那套 routeRules 属性（`prerender`、`ssr`、`swr`、`isr`、`headers` 等）。

## 先用开关打开它

它是实验特性，官方 `::read-more` 要求先开：

> This feature is experimental and in order to use it you must enable the `experimental.inlineRouteRules` option in your `nuxt.config`.

没开 `experimental.inlineRouteRules`，页面里的 `defineRouteRules` 不起作用。

一个效果说明：官方 `::note` 讲——跑 `nuxt build` 时，这样标了 `prerender: true` 的首页会在 `.output/public/index.html` 预渲染好、静态服务。

## 页路径怎么映射成规则

"Notes" 给了三条映射：

| 页面文件 | 规则作用到 |
| --- | --- |
| `~/pages/foo/bar.vue` | `/foo/bar` 的请求 |
| `~/pages/foo/[id].vue` | `/foo/*` 的请求 |
| 有限可选段的页（如自定义 `path` `/:locale(en|fr)/about`） | 每个备选各生成一条（`/en/about` 和 `/fr/about`） |

动态 `[id]` 会被摊成 `/foo/*` 这种通配——所以规则是按"整片路径"生效，不是按某个具体 id。

## 哪些路径转不了会静默失效（有警告）

这是最该记住的边界。官方列了三类"没法转成等价 route rule 模式"的：

- 带正则的参数 `/:id(\d+)`
- 部分段 `/prefix-:id`
- 可重复段 `/:slug+`

遇到这些，这一页的规则**不会被应用**，并且 Nuxt 在构建期**发出警告**。官方给的兜底：这种情况就回到 `nuxt.config` 里的 `nitro.routeRules` 显式写。

另外，如果你用了自定义 `path` 或 `alias`（`definePageMeta` 里设的），要更精确控制时也应直接在 `nuxt.config` 写 `routeRules`——就近声明覆盖不到这些特殊形态。

## 自查

- `defineRouteRules({...})` 在页面里就近声明路由规则，编译时并进该页路由的 `routeRules`，可用键就是 367 那套 routeRules 属性。
- 实验特性：必先开 `experimental.inlineRouteRules`。
- 映射：`bar.vue`→`/foo/bar`、`[id].vue`→`/foo/*`、有限可选段各生成一条。
- 带正则参数/部分段/可重复段的路径转不了→规则不生效+构建警告，那种回 `nuxt.config` 的 `routeRules`/`nitro.routeRules` 显式写；自定义 `path`/`alias` 也建议直接写 config。

## 下一篇

下一篇 392 把预渲染的"发现"再抠细一层：爬虫具体顺着 HTML 里哪些 `<a href>` 去发现路由、`prerenderRoutes()` 与两个钩子怎么把发现不了的页补进去。

## 本文依据

- `defineRouteRules` 定义（"Define route rules for hybrid rendering at the page level"）、`experimental.inlineRouteRules` 的 `::read-more`、`{ prerender: true }`→routeRules `'/'` 翻译示例、`::note`（`nuxt build` 预渲染首页到 `.output/public/index.html`）、三条路径映射（`/foo/bar`、`/foo/*`、有限可选段逐条）、带正则/部分段/可重复段不应用+构建警告、自定义 `path`/`alias` 建议直接写 config（`api/utils/define-route-rules`；routeRules 属性与预渲染见 367/379/380）。
