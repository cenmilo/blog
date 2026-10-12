---
url: /posts/7c9103d5fdaa08bdb06eaedd5f24207f/
title: event.context 与 params：动态文件名怎么变成参数
date: 2027-01-22T09:00:00+08:00
updated: 2027-01-22T09:00:00+08:00
author: cmdragon

summary:
  摘要：服务端 handler 里有两个常碰的对象：event.context.params（动态文件名匹配到的段）和 event.context（整趟请求的上下文，可读写、用来在中间件和 handler 间传数据）。这一篇讲动态路由在服务端怎么取值——单段 [name] 用 getRouterParam(event,'name')、catch-all [...] 用 event.context.params._（无名时）或 event.context.params.slug（[...slug] 时）、整条匹配路径在 event.context.path；再讲中间件往 event.context 挂东西（如 event.context.auth）给下游 handler 读，这是服务端"跨处理步骤传数据"的标准做法。

categories:
  - 前端开发

tags:
  - nuxt4
  - event-context
  - 动态路由
  - params
  - h3
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上一篇读的是"客户端传来的体/查询串"。这一篇读另一类——**URL 路径里的那段动态部分**，以及服务端各处怎么通过 `event.context` 互相传数据。

## event.context.params：动态段落在这

`server/` 里文件名带方括号的动态段，匹配到的值进 `event.context.params`。官方 server 那页 "Route Parameters" 给的是单段 `[name]`，推荐用 `getRouterParam` 取：

```ts [server/api/hello/[name].ts]
export default defineEventHandler((event) => {
  const name = getRouterParam(event, 'name')
  return `Hello, ${name}!`
})
```

> You can now universally call this API on `/api/hello/nuxt` and get `Hello, nuxt!`.

访问 `/api/hello/nuxt`，`name` 就是 `nuxt`。官方 tip 也建议用 `getValidatedRouterParams` 配合 schema 做运行时和类型双层校验。

## catch-all：整路径与片段各在哪

`[...]` 这种兜底匹配，官方 "Catch-all Route" 给的位置和单段不同，要分两处看：

```ts [server/api/foo/[...].ts]
export default defineEventHandler((event) => {
  // event.context.path 取整条路径：'/api/foo/bar/baz'
  // event.context.params._ 取匹配到的片段：'bar/baz'
  return `Default foo handler`
})
```

给 catch-all 起名字用 `[...slug].ts`，片段就在 `event.context.params.slug`：

```ts [server/api/foo/[...slug].ts]
export default defineEventHandler((event) => {
  // event.context.params.slug to get the route segment: 'bar/baz'
  return `Default foo handler`
})
```

三个位置记牢，服务端和页面路由不一样、最容易记串：

- **整条匹配路径** → `event.context.path`
- **无名 catch-all 的片段** → `event.context.params._`（那个 `_` 是默认 key）
- **命名 catch-all `[...slug]` 的片段** → `event.context.params.slug`

对比页面侧：pages 里 `[...slug]` 的 `params.slug` 是**数组**，而服务端这里给的是字符串片段（`'bar/baz'`）。别把两边的形状当成一样。

## event.context 不只是装 params：跨步骤传数据

`event.context` 是整趟请求的上下文对象，`params` 只是它的一个字段。你还能往里挂自己的数据——这正是**服务端中间件**（291）往下游传东西的标准做法。官方 auth 中间件示例：

```ts [server/middleware/auth.ts]
export default defineEventHandler((event) => {
  event.context.auth = { user: 123 }
})
```

中间件"不该返回任何东西、只能检视/扩展 event.context/抛错"（291 那条），它的"扩展"就是往 `event.context.xxx` 上挂。因为服务端中间件跑在任何具体路由之前（291），后面的 handler 就能读到这个 `event.context.auth`。这是服务端各处理环节之间传数据的正规通道——不是靠全局变量，是靠这趟请求自带的 `event.context`（每请求一份，天然隔离）。

## 一张"服务端能读哪些请求数据"的总览

把 304、305、306 串起来，handler 里能拿到的入站信息分四类：

| 读什么 | 用什么 |
| --- | --- |
| 请求体 | `readBody` / `readValidatedBody` |
| 查询串 | `getQuery` / `getValidatedQuery` |
| 动态路由段 | `getRouterParam` / `event.context.params` |
| 上游中间件挂的数据 | `event.context.xxx`（如 `event.context.auth`） |

外加整条路径 `event.context.path`。

## 自查

- 单段 `[name]` 用 `getRouterParam(event,'name')`；catch-all 片段看 `event.context.params._`（无名）或 `.slug`（命名），整路径看 `event.context.path`。
- 服务端 catch-all 的片段是字符串（`'bar/baz'`），和页面 `[...slug]` 的数组形状不同，别记混。
- 中间件给下游传数据 → 挂 `event.context.xxx`，这是每请求隔离的正规通道，不是全局变量。
- 要运行时/类型都安全地校验参数 → `getValidatedRouterParams` 配 schema。

## 下一篇

下一篇 307 讲 `setResponseStatus`：服务端设状态码（和可选 statusText）的工具，以及它那句"在浏览器里没有效果"的限定。

## 本文依据

- server 目录（`directory-structure/server`）：Route Parameters 一节（`[name].ts` + `getRouterParam(event,'name')`、`/api/hello/nuxt`→`Hello, nuxt!`、tip `getValidatedRouterParams`）；Catch-all Route 一节（`[...].ts`→`event.context.path`/`event.context.params._`、`[...slug].ts`→`event.context.params.slug`）；Server Middleware 一节（`auth.ts` 的 `event.context.auth = { user: 123 }`，见 291）。
- 与页面侧 `[...slug]` 数组形状的区别（见 111/P05 页面路由记录）。
