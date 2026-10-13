---
url: /posts/e6b715fb7e0d24e9545d9bdc04997251/
title: 656 - Kit 服务端处理程序两函数：route 精确匹配、middleware 与只在开发那份
date: 2027-05-18T11:00:00+08:00
updated: 2027-05-18T11:00:00+08:00
author: cmdragon

summary:
  摘要：往服务端加路由有两条路：**`addServerHandler` 加一份会进产物的 handler，`addDevServerHandler` 加只在开发期用、生产构建里被排除的那一份**。两者的字段都是五个，而 `route` 的语义是那一节最要紧的一句：**"要精确匹配的路由；以 `/**` 结尾则同时匹配它下面的路径；给空字符串时这个 handler 被注册为 middleware"**。`/**` 那条还有一句补偿：**Nitro 2 宿主的路由只匹配基路径下面的部分，所以 Nuxt 会把基路径再注册一次，让同一次注册在两边表现一致**。开发版那一栏里，**`method`、`middleware`、`route` 三项在 `nitropack` v2 宿主上都会变规则**（method 被忽略、每个 dev handler 都当 middleware 跑、不带 `/**` 的路由也匹配其下路径）。

categories:
  - 前端开发

tags:
  - nuxt4
  - 服务端路由
  - 开发期处理器
  - NuxtKit
  - 预渲染
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模块要提供一个接口，或者想在 dev 里挂一个查看器——这两个函数是入口。它们的字段看起来普通，**真正要记的是那三处"在 Nitro 2 宿主上规则会变"**。

## 开场句里点名了六个

页面先交代 Nitro 是什么：**一个用于构建极快 Web 服务器的开源 TypeScript 框架，Nuxt 用它作为服务端引擎**。然后点名可用函数：**`useNitro` 取实例（如果你的模块在没有 Nitro 时也要工作就用 `tryUseNitro`）、`addServerHandler` 加服务端处理程序、`addDevServerHandler` 加只在开发模式使用的那一份、`addNitroPlugin` 加插件扩展 Nitro 的运行时行为、`addPrerenderRoutes` 加要被预渲染的路由**。

**这句点了六个名字，而页上有九个条目**——`addServerImports`、`addServerImportsDir`、`addServerScanDir` 三个没在开场句里。这一篇讲前三个（两个 handler ＋ 预渲染），其余留到下一篇。

## `addServerHandler`：五个字段

```ts
function addServerHandler (handler: ServerHandlerInput): void
```

定性：**添加一个 Nitro 服务端处理程序；如果你想创建服务端中间件或自定义路由，就用它**。

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `handler` | `string \| { nuxt?: string, nitro2?: string, nitro3?: string }` | `true` | **事件处理程序的路径，或每个服务端 API 各一个路径**；Nuxt 注册应用的服务端能跑的那份实现 |
| `route` | `string` | `false` | **要精确匹配的路由。以 `/**` 结尾则同时匹配它下面的路径。如果给空字符串，这个 handler 被注册为 middleware** |
| `middleware` | `boolean` | `false` | 指明这是一个中间件；**中间件在每个路由上都会被调用，通常应什么都不返回以把请求传给下一个 handler** |
| `lazy` | `boolean` | `false` | 用懒加载导入这个 handler；**只在你希望按需加载时有用** |
| `method` | `string` | `false` | 路由方法匹配器；**如果 handler 的文件名里含方法名，它会作为默认值被使用** |

第一行的对象形态就是服务端兼容性那一章讲的三个 variant 键（`nuxt`/`nitro2`/`nitro3`）。`lazy` 与 `method` 两行都指向文件命名：**`runtime/robots.get` 里的 `.get` 就是"文件名里含方法名"那个默认值的来源**（示例里 handler 路径写的正是 `./runtime/robots.get`，不带扩展名）。

### `/**` 在两边怎么对齐

页面在参数表下面单独给了一句：

> A route ending in `/**` matches its base path too, so `route: '/_fonts/**'` serves `/_fonts` and everything below it. On a Nitro 2 host, whose router matches only the paths below the base, Nuxt registers the base path as a second route so that one registration behaves the same on both.

两处结论：**`/_fonts/**` 连 `/_fonts` 自己也服务**（不需要再注册一次基路径）、**Nitro 2 宿主上 Nuxt 会替你把基路径补注册一遍**，所以你写一次就够了。

完整示例是一对文件：

```ts [module.ts]
import { addServerHandler, createResolver, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  setup (options) {
    const { resolve } = createResolver(import.meta.url)

    addServerHandler({
      route: '/robots.txt',
      handler: resolve('./runtime/robots.get'),
    })
  },
})
```

```ts [runtime/robots.get.ts]
export default defineEventHandler(() => {
  return {
    body: `User-agent: *\nDisallow: /`,
  }
})
```

访问 `/robots.txt` 得到：

```txt
User-agent: *
Disallow: /
```

**返回的是 `{ body: ... }` 这个对象形状**，页面把它直接对应到那段纯文本响应。

## `addDevServerHandler`：生产构建里没有它

定性：**添加一个只在开发模式使用的 Nitro 服务端处理程序；这个 handler 会被排除在生产构建之外**。

```ts
function addDevServerHandler (handler: DevServerHandlerInput): void
```

字段也是五个，但**每个都多了一句"在某宿主上不一样"**：

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `handler` | `EventHandler \| { nuxt?: (event: RequestEvent) => unknown, nitro2?: EventHandler, nitro3?: EventHandler }` | `true` | 事件处理程序，或每个服务端 API 各一个；**注意 `nuxt` 那一支接收的是 `RequestEvent` 并返回 `unknown`** |
| `route` | `string` | `false` | 要匹配的路由；以 `/**` 结尾则同时匹配其下路径；**在 `nitropack` v2 宿主上，不带 `/**` 的路由也会匹配其下的路径**；空字符串则注册为中间件 |
| `method` | `string` | `false` | 要匹配的 HTTP 方法（例如 `'POST'`）；**在 `nitropack` v2 宿主上，handler 对每种方法都会运行** |
| `middleware` | `boolean` | `false` | 在路由 handler 之前运行；返回空以把请求交给下一个；**在 `nitropack` v2 宿主上，每个 dev handler 都作为中间件运行** |
| `meta` | `object` | `false` | 路由元数据（例如 OpenAPI 操作信息）；**Nitro v3 会把它挂到不是中间件的那个 handler 上** |

**这一张表里三行都在说同一件事：dev handler 在 `nitropack` v2 宿主上的语义更松**——路由变成前缀匹配、方法过滤失效、一律当中间件跑。**只在一个宿主上验过的 dev handler 不能当作两边都对**。

最小用法是给一个纯 Web API 的响应：

```ts
addDevServerHandler({
  handler: {
    nuxt: () => new Response(`Response generated at ${new Date().toISOString()}`),
  },
  route: '/_handler',
})
```

**`handler` 直接写成 `{ nuxt: ... }` 这种 variant 对象**，函数返回一个 `Response`。

页面给的实用例子是**Tailwind 配置查看器**，它同时示范了三件事：**用 `joinURL(nuxt.options.app?.baseURL, '/_tailwind')` 把 base 拼进路由**、**用动态 import 加载第三方包并 `@ts-expect-error`**、**把这个包的 `asMiddleware()` 结果直接当 handler 传**（这里传的是普通函数，不是 variant 对象）：

```ts
import { joinURL } from 'ufo'
import { addDevServerHandler, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  async setup (options, nuxt) {
    const route = joinURL(nuxt.options.app?.baseURL, '/_tailwind')

    // @ts-expect-error - tailwind-config-viewer does not have correct types
    const createServer = await import('tailwind-config-viewer/server/index.js').then(r => r.default || r) as any
    const viewerDevMiddleware = createServer({ tailwindConfigProvider: () => options, routerPrefix: route }).asMiddleware()

    addDevServerHandler({ route, handler: viewerDevMiddleware })
  },
})
```

**`route` 里要拼 `app.baseURL`**这一点很实在：应用不在根路径部署时，写死的 `/_tailwind` 会 404。

## `addPrerenderRoutes`：把路由交给预渲染

```ts
function addPrerenderRoutes (routes: string | string[]): void
```

只有一个参数：**要预渲染的一个路由或路由数组**。

示例是一个 sitemap 模块的形状——**先用自己的 `defaults` 决定要不要预渲染**：

```ts
import { addPrerenderRoutes, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  meta: {
    name: 'nuxt-sitemap',
    configKey: 'sitemap',
  },
  defaults: {
    sitemapUrl: '/sitemap.xml',
    prerender: true,
  },
  setup (options) {
    if (options.prerender) {
      addPrerenderRoutes(options.sitemapUrl)
    }
  },
})
```

**这一节没写它的时机要求**，也就是说没有说在哪个钩子之前调用才有效——这与 `useNitro` 那节明写的"只能在 `ready` 钩子之后调用"形成对照，所以**不要把两者的时机假设混用**。

## 两者怎么选

| 需求 | 用哪个 | 会进产物吗 |
| --- | --- | --- |
| 模块对外提供一个 API 或一条固定路由 | `addServerHandler` | **会** |
| 只服务开发期的查看器、调试端点 | `addDevServerHandler` | **不会，被排除在生产构建外** |
| 服务端中间件 | `addServerHandler` ＋ `middleware: true`（或把 `route` 写成空字符串） | 会 |
| 让某个路径被预渲染 | `addPrerenderRoutes` | 不涉及 handler |

文档未记载的还有：`addServerHandler` 的同名 `route` 重复注册会怎样、`lazy` 与 `meta` 是否适用于生产 handler（`meta` 只在 dev 那张表里出现）、以及 dev handler 在 `server.builder` 不是 Nitro 时是否仍然注册。

下一篇讲 Nitro 其余六个函数：两个取实例的、一个插件的，和三个扫描目录的。

## 下一篇

下一篇 657：Kit Nitro 其余六函数：拿实例的两个时机、服务端插件与三个扫描目录

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/nitro
