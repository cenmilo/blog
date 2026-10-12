---
url: /posts/eacf310b3191e1e19306b5e344d3ca5e/
title: 类型化取数：routeTypedFetch 与 strictRouteTypes 的三档
date: 2027-01-01T10:00:00+08:00
updated: 2027-01-01T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：平时 `useFetch('/api/xxx')` 的返回类型来自服务端贡献给 `ServerRoutes` 的响应类型；这两个开关改的是**类型从哪来**和**认不出的路径要不要报错**。`routeTypedFetch` 的定位句："Type `$fetch` and `useFetch` from the routes your server builder reports, rather than from the responses nitro contributes to `ServerRoutes` through its `InternalApi` interface."，官方列了五条后果（逐条都是原话级别的信息）：**"a `Date` a handler returns is a `string` at the call site"**、"a `body` the handler validates cannot be omitted or given the wrong shape"、"a method the route does not answer is rejected"、**"`params` is no longer accepted, since it was only ever an alias for `query`"**、以及手写增强 `ServerRoutes` 的路由需要重写（用 `Endpoint` 声明的可以先重写、开关两侧都生效）。它是 opt-in，但 **`future.compatibilityVersion: 5` 会启用它**，并且官方注明**这个选项只在两种类型化并存期间存在，Nuxt 5 起生成路由集是唯一来源、选项消失**。`strictRouteTypes` 则"Reject a request to a path no server route answers. Requires `routeTypedFetch`"，**三档而非布尔**：`false`（默认）认不出的路径被接受并解析为 `unknown`、且不拒绝任何方法，理由句是 "Nuxt cannot see every way a request may be answered: nitro middleware, a `routeRules` proxy or a catch-all handler can answer a path no route describes."；`true` 只接受服务端构建器报告的路由及其可答方法；**`'isomorphic'`** 在 `true` 之上把你的页面也当作返回 `string` 的 GET 路由，于是 `$fetch('/about')` 按 Vue Router 提供的那条路由来类型化而不是被拒。

categories:
  - 前端开发

tags:
  - nuxt4
  - 类型
  - routeTypedFetch
  - strictRouteTypes
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先解释这两个开关解决的是哪类错误：**路径拼错了、方法用错了、或者你以为是 `Date` 其实是字符串**。默认情况下前两种不会报错，第三种你已经遇到过（235 那个 `new Date()` 变成 `string`）。

## routeTypedFetch：换掉类型的来源

原文定义（注意它比较的是两个来源）：

> Type `$fetch` and `useFetch` from the routes your server builder reports, rather than from the responses nitro contributes to `ServerRoutes` through its `InternalApi` interface.

```ts twoslash [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    routeTypedFetch: true,
  },
})
```

配套一句解释了为什么"更紧"："The generated route set carries the `body`, `query` and `headers` a handler validates as well as its response, typed as it arrives over the wire, so a resolving call is typed more tightly than before"——**关键是 "typed as it arrives over the wire"（按线路上到手的样子来类型化）**。

五条后果我逐条抄，因为第 4 条会直接改掉可用选项：

- a `Date` a handler returns is a `string` at the call site
- a `body` the handler validates cannot be omitted or given the wrong shape
- a method the route does not answer is rejected
- `params` is no longer accepted, since it was only ever an alias for `query`
- a route you registered by augmenting `ServerRoutes` by hand needs rewriting, as the two descriptions of a route differ；用 `Endpoint`（从 `nuxt/app` 导入）重写声明的路由，**开关两侧都生效**，所以可以在切换之前先重写

第 1 条我认为是这批里最实用的：**开了它，`Date` 就不再被"尽力转换"的假象迷惑**。这与 235 那句 "Nuxt does its best to convert the return type" 正好呼应——**类型化程度提高，代价是类型不再"替你美化"**。

第 4 条要特别注意：`params` 这个别名（220 表里还写着 "Alias for `query`."）在这个模式下**不再被接受**。也就是说，升级类型化会让一部分原本合法的调用变成错误。

## 它是 opt-in，但兼容版本会自动开

官方两句放在一起才完整：

> Because of those, it is opt-in, and `future.compatibilityVersion: 5` enables it. Setting it explicitly selects either typing whatever the compatibility version, so you can adopt it before moving to Nuxt 5, or move to Nuxt 5 with `routeTypedFetch: false` and adopt it separately.

> This option exists only while both typings are supported. From Nuxt 5 the generated route set is the only source and the option is gone.

我的读法是：**这是个有寿命的过渡开关**。现在显式设 `true` 属于提前采用；显式设 `false` 也能带着 Nuxt 5 先不动类型化，之后再单独迁移。**但"以后再迁移"要有预算，因为它最终会消失。**

## strictRouteTypes：三档，不是布尔

定义句与依赖关系：

> Reject a request to a path no server route answers. Requires `routeTypedFetch`, which types `$fetch` and `useFetch` from the routes your server will serve.

```ts twoslash [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    routeTypedFetch: true,
    strictRouteTypes: true,
  },
})
```

一句关键澄清（**响应类型由 handler 决定，跟这个开关无关；它只管"认不出的路径怎么办"**）：

> With `routeTypedFetch` enabled a response is typed by the handler that answers it whatever this option is set to. What the option decides is whether a path Nuxt does not recognise is an error or is simply typed as `unknown`.

三档原文：

| 取值 | 官方说明 |
| --- | --- |
| `false`（默认） | "a path Nuxt does not recognise is accepted and resolves to `unknown`, and no method is rejected. **Nuxt cannot see every way a request may be answered: nitro middleware, a `routeRules` proxy or a catch-all handler can answer a path no route describes.**" |
| `true` | "only the routes your server builder reports are accepted, for the methods they answer." |
| `'isomorphic'` | "as `true`, and your pages are included as `GET` routes returning `string`, so `$fetch('/about')` is typed by the route the Vue router serves rather than rejected." |

`false` 那句理由我很认同——**Nitro 中间件、routeRules 代理、catch-all 路由都能应答一个"路由表里不存在"的地址**，所以默认不敢报错。这也解释了 065 讲过的服务端动态段与 pages 的关系：catch-all 存在时，开 `true` 会把合法请求判成错误。

`'isomorphic'` 那一档解决的是另一类需求：**前端取自己页面的 HTML**（比如预渲染后抓另一个页面的内容做处理）。官方明说这样 `$fetch('/about')` 会按路由来类型化、返回 `string`。

开了严格档之后，官方给的判例（注释是原文）：

```ts
// server/api/todos.post.ts answers POST only
await $fetch('/api/todos', { method: 'POST' })
await $fetch('/api/todo', { method: 'POST' })
//           ^ no POST route matches '/api/todo'
await $fetch('/api/todos')
//           ^ no GET route matches '/api/todos'
```

**一个字母的差别（`/api/todo` vs `/api/todos`）就变成编译期错误**——这正是这类开关的价值。

## 与 createUseFetch 的 routes 是什么关系

226 讲过 `createUseFetch({ routes })` 那套第三方接口声明，两件事的边界官方写得很清楚：

- `routes` 那套**与 `routeTypedFetch` 无关**："Its responses, query parameters, and request bodies are typed from that route set, **including when `experimental.routeTypedFetch` is off**."
- `routeTypedFetch` 只管**你自己服务端**上报的路由；`routes` 是客户端自己声明的一套，官方补了隔离句："The routes a client declares are its own. They are not added to your app's route set, so plain `$fetch` and `useFetch` are unaffected."
- **要拒绝未匹配路径，需要两个开关同时开**（`routeTypedFetch: true` + `strictRouteTypes: true`），官方原话是 "Unmatched paths and methods are accepted by default and resolve to `unknown`. To reject them, enable `routeTypedFetch` and `strictRouteTypes`"。

## 迁移时我会做的三步

1. 先把手写增强 `ServerRoutes` 的地方改成 `Endpoint`/`DynamicParam` 声明（官方允许先重写、两侧都生效）。
2. 打开 `routeTypedFetch: true` 跑一次 `nuxt typecheck`（035/036 讲过这个命令）——**这一步会暴露 `params` 用法与 `Date` 假设**，这两类是原文点名的行为变化。
3. 再考虑 `strictRouteTypes`：**如果项目里有 Nitro 中间件、catch-all 服务端路由或 `routeRules` 代理，就从 `false` 起步或先不上**，因为默认档存在的理由正是"这三种情况 Nuxt 看不见"。

## 自查

1. 开了 `routeTypedFetch` 之后，`params` 会怎样？（答：不再被接受，官方说它"was only ever an alias for `query`"。）
2. `strictRouteTypes` 的默认档为什么不敢报错？（答：nitro 中间件、`routeRules` 代理、catch-all handler 都可能应答路由表里没有的路径。）
3. `'isomorphic'` 那一档多接受了什么？（答：把页面本身当作返回 `string` 的 GET 路由，于是 `$fetch('/about')` 不报错。）

## 下一篇

245 - baseURL 的三个来源：请求选项、runtimeConfig 与 app 命名空间

## 本文依据

- `guide/going-further/experimental-features`：`routeTypedFetch` 一节（定义句、配置示例、"carries the `body`, `query` and `headers` a handler validates... typed as it arrives over the wire"、五条后果原文、opt-in 与 `compatibilityVersion: 5`、显式选择的两句、"This option exists only while both typings are supported..." 的 note）；`strictRouteTypes` 一节（定义与依赖句、配置示例、"a response is typed by the handler that answers it whatever this option is set to"澄清句、三档逐项原文、`$fetch` 判例三行与两条错误注释）。
- `api/composables/create-use-fetch`：`routes` 一节（"including when `experimental.routeTypedFetch` is off"、四条限制、两个开关同开才拒绝、客户端声明隔离句、fetchdts tip）（226 已逐句引）。
- `api/composables/use-fetch`：`query`/`params` 两行互为别名的原文（220 已逐句引）。
- `getting-started/data-fetching`：Serializing Data From API Routes 段（"Nuxt does its best to convert the return type"）（235 已逐句引）。
- `api/commands/typecheck`：`nuxt typecheck` 的用法与前置（035/036 已逐句引）。
- `directory-structure/server`：服务端路由动态段与 catch-all 的记载（065 已逐句引）。
