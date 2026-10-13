---
url: /posts/b6a05c358fadbe128458d51216f08642/
title: 402 - routeTypedFetch：内部接口路径的类型化
date: 2027-02-23T09:00:00+08:00
updated: 2027-02-23T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：`$fetch('/api/todos')` 的返回类型从哪来，Nuxt 换过一套来源，`experimental.routeTypedFetch` 就是切换它的开关。官方定义："Type `$fetch` and `useFetch` from the routes your server builder reports, rather than from the responses nitro contributes to `ServerRoutes` through its `InternalApi` interface."——改为按**服务端构建器报告的路由集**定型，那张表还带着 handler 校验过的 `body`、`query`、`headers`，并按数据过线后的样子定型。五条后果逐条：handler 返回的 `Date` 在调用处是 `string`；被校验过的 `body` 不能省略也不能给错形状；路由不该答的方法被拒绝；**`params` 不再接受**（它本来就只是 `query` 的别名）；手写增强 `ServerRoutes` 注册的路由需要改写（两种路由描述方式不同），而用 `nuxt/app` 的 `Endpoint` 按新形状声明的路由在开关开关两侧都生效。它默认不开、`future.compatibilityVersion: 5` 会启用；且官方明说**这个选项只存在于两套定型并存的期间，Nuxt 5 起选项消失**。

categories:
  - 前端开发

tags:
  - nuxt4
  - routeTypedFetch
  - 类型化取数
  - useFetch
  - 升级
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

401 的 `strictRouteTypes` 要先开 `routeTypedFetch` 才能用，这个前置开关本身就是 395-414 里改动最大的一条——它换掉的是"接口类型从哪来"的地基。

## 换的是类型来源

官方定义：

> Type `$fetch` and `useFetch` from the routes your server builder reports, rather than from the responses nitro contributes to `ServerRoutes` through its `InternalApi` interface.

两套来源的差别：

- **旧**：类型来自 Nitro 通过 `InternalApi` 接口贡献给 `ServerRoutes` 的**响应**。
- **新（开了这个开关）**：类型来自**你的服务端构建器报告的那份路由集**。

而且新来源带的不只是返回值。官方补了一句：

> The generated route set carries the `body`, `query` and `headers` a handler validates as well as its response, typed as it arrives over the wire, so a resolving call is typed more tightly than before

生成的那张路由表装着 handler **校验过的** `body`、`query`、`headers`，以及它的响应——并且是"按数据过线后的样子"定型（所以才会出现下面第一条那个反直觉结果）。

## 五条后果

逐条按原文落地：

| 后果 | 原文要点 |
| --- | --- |
| 返回 `Date` 变 `string` | "a `Date` a handler returns is a `string` at the call site" |
| 被校验的 `body` 不能省、不能错形状 | "a `body` the handler validates cannot be omitted or given the wrong shape" |
| 不该答的方法被拒绝 | "a method the route does not answer is rejected" |
| **`params` 不再接受** | "`params` is no longer accepted, since it was only ever an alias for `query`" |
| 手写 `ServerRoutes` 要改写 | 见下 |

第一条最容易让人困惑：`server/api/foo.ts` 里 `return new Date()`，调用处 `useFetch('/api/foo')` 的 `data` 类型是 `string`——因为它"按过线时的样子"定型，`Date` 走 JSON 过去就成了字符串（这条和 235 讲的 API 路由用 `JSON.stringify` 序列化、只限基本类型是同一个根子）。

第四条最容易被 CI 抓到：`useFetch` 的选项表里 `params` 与 `query` 原本是互为别名的（220 那张 23 行表里就有），开了 `routeTypedFetch` 后 **`params` 直接不被接受**，官方的理由是它本来就只有 `query` 的别名作用。手上还有 `params` 写法的，迁移时统一换成 `query`。

## 手写注册的路由要改写

第五条后果影响的是模块作者或手工增强过类型的人：

> a route you registered by augmenting `ServerRoutes` by hand needs rewriting, as the two descriptions of a route differ

因为两套"描述一条路由"的方式不一样，你手写的增强得按新形状重写，官方在升级指南里对应有 "Typed fetch rebuilt on generated route types" 一节。

但它同时给了一个可以提前动手的口子：

> A route declared with `Endpoint` (imported from `nuxt/app`) in the rewritten shape is typed with `routeTypedFetch` both on and off, so you can rewrite it ahead of the switch

也就是用 `nuxt/app` 导出的 **`Endpoint`** 按新形状声明的路由，**在开关开与关两种状态下都生效**——所以可以现在就改，不用等切换那天。

## 默认状态与 `compatibilityVersion: 5`

正因为上面这几条会改动现有代码，官方写明了它是 opt-in：

> Because of those, it is opt-in, and `future.compatibilityVersion: 5` enables it. Setting it explicitly selects either typing whatever the compatibility version, so you can adopt it before moving to Nuxt 5, or move to Nuxt 5 with `routeTypedFetch: false` and adopt it separately.

三个可执行结论：

1. 它默认不开；设 `future.compatibilityVersion: 5` 会被一并启用（352 那条 `future` 节）。
2. 想提前采用：显式 `routeTypedFetch: true`，不用等版本切换。
3. 想先升 Nuxt 5、暂时不碰类型化：`routeTypedFetch: false`，两件事分开做。

## 这是一条有寿命的选项

官方 `::note` 把它的时间性说死了：

> This option exists only while both typings are supported. From Nuxt 5 the generated route set is the only source and the option is gone.

**Nuxt 5 起生成的路由集是唯一来源，这个选项消失。** 所以别把它当成长期配置项规划——它是两套定型并存期间的过渡开关。401 的 `strictRouteTypes` 依赖它，也跟着受影响。

## 自查

- `routeTypedFetch` 把 `$fetch`/`useFetch` 的类型来源从"Nitro 经 `InternalApi` 贡献给 `ServerRoutes` 的响应"换成"服务端构建器报告的路由集"，该表带 handler 校验过的 `body`/`query`/`headers`。
- 五条后果：`Date`→`string`、被校验的 `body` 不可省/不可错形状、不该答的方法被拒、**`params` 不再接受**（原本只是 `query` 别名）、手写增强 `ServerRoutes` 需改写。
- 用 `nuxt/app` 的 `Endpoint` 按新形状声明的路由在开关两侧都生效，可提前迁移。
- 默认 opt-in；`future.compatibilityVersion: 5` 会启用；也可显式 `true` 提前采用或 `false` 分开升级。
- 选项有寿命：Nuxt 5 起生成的路由集是唯一来源，选项消失（`strictRouteTypes` 依赖它）。

## 下一篇

下一篇 403 处理 monorepo：`workspaceDir` 与 `typescript.includeWorkspace`/`hoist` 怎么配合，让父工作区里的类型可见。

## 本文依据

- `## routeTypedFetch`：定义句（"Type `$fetch` and `useFetch` from the routes your server builder reports, rather than from the responses nitro contributes to `ServerRoutes` through its `InternalApi` interface"）、"carries the `body`, `query` and `headers` a handler validates as well as its response, typed as it arrives over the wire"、五条后果原文、`Endpoint`（imported from `nuxt/app`）在开关两侧都生效、opt-in 与 `future.compatibilityVersion: 5` 启用及两种采用节奏、`::note`（"From Nuxt 5 the generated route set is the only source and the option is gone"）（`guide/going-further/experimental-features`）；`params` 与 `query` 原为互为别名的选项表见 220（`api/composables/use-fetch`）；API 路由用 `JSON.stringify` 只限基本类型见 235。
