---
url: /posts/86c0f22786099f101b89f9d49c952096/
title: 452 - defineCachedEventHandler 在服务端的落点
date: 2027-03-11T11:00:00+08:00
updated: 2027-03-11T11:00:00+08:00
author: cmdragon

summary:
  摘要：`defineCachedEventHandler()` 与 `defineCachedFunction()` 不在 Nuxt 给服务端的官方表面里。`nuxt/server` 提供十九项与 runtime 无关的工具（`defineEventHandler`、`getQuery`、`useRuntimeConfig`、`getRouteRules` 等），**缓存一项都没有**；官方的说法是把它们算进"不可移植"那一类——它们来自 `nitropack/runtime`，Nuxt 无法为每个构建器提供，并且直接给了一句替代建议："想让缓存在任何地方都能跑，就直接依赖一个缓存库"。同一个名字在 Nitro 2 与 Nitro 3 里要分别从 `nitropack/runtime` 和 `nitro/cache` 导入，官方明说这类代码"绑定服务端 runtime，Nuxt 5 时需要重新审视"。

categories:
  - 前端开发

tags:
  - nuxt4
  - 服务端缓存
  - Nitro
  - 可移植性
  - 性能
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`server/api/rates.get.ts` 里包一层缓存，把对第三方汇率接口的调用结果留十分钟。这个想法在 Nuxt 里能落地，但落地方式跟你想的不一样：**它不在官方给服务端的那套工具里**。

这一篇讲清这件事的位置、代价，以及三种缓存手段该怎么排。

## 先确认表面里没有它

`nuxt/server` 是 Nuxt 服务端的第二个 runtime 表面（与给浏览器那半边的 `nuxt/app`，也就是 `#app`，并列），官方的定位句是：

> `nuxt/server` exists to provide agnostic utilities for server code that aren't tied to a particular server runtime.

它列出的工具清单有十九行：`defineEventHandler`、`createError`/`isNuxtError`、`getRequestURL`/`getRequestHost`/`getRequestProtocol`、`getRequestHeader`/`getRequestHeaders`、`getRouterParam`/`getRouterParams`、`getRequestIP`、`getQuery`/`getValidatedQuery`、`readBody`/`readValidatedBody`、`getCookie`/`parseCookies`/`setCookie`/`deleteCookie`、`setResponseStatus`、`handleCors`、`sendRedirect`、`serverFetch`、会话五件套、`deriveSecret`、`getRouteRules`/`matchRouteRules`、`useRuntimeConfig`、`useServerHooks`、`useAppConfig`。

**这名单里没有缓存。** 所以要用 `defineCachedEventHandler()`，就得**越过这个表面**——而"越过表面"这件事，官方是当成一处需要留意的取舍来写的，不是当成推荐做法。

## 官方把它归进"不可移植"

服务端导入那一节有两句关键的话。第一句给来源：

> Nitro's own APIs mostly take no event at all: `defineCachedEventHandler()`, `useStorage()`, `useDatabase()` and tasks. Import them from `nitropack/runtime`.

注意前半句：**这些 API 大多根本不接 event**。这点和 `nuxt/server` 那套工具正好相反（那套全部围着 `RequestEvent` 转，304 那篇讲的 handler 入参就在这一层）。

第二句在 "What Isn't Portable" 一节里，把话说到底了：

> **Caching.** `defineCachedEventHandler()` and `defineCachedFunction()` come from `nitropack/runtime`. For caching that runs anywhere, depend on a cache library directly.

翻译：这两个函数来自 `nitropack/runtime`；**如果你的缓存要能在任何地方跑，就直接依赖一个缓存库**。这一节标题就是"哪些东西不可移植"，理由是官方那句"because Nuxt can't provide them for every builder"。

同表里被归进同一类的还有三项：Storage（`useStorage()` 及其驱动配置）、Server plugins（`defineNitroPlugin()`）、Lazy handlers（`lazyEventHandler()` 来自 `h3`）。

## 换个 Nitro 版本，导入路径就变

模块兼容那一节给的是两列对照：

| 区域 | 在 `nitro2` 文件里 | 在 `nitro3` 文件里 |
| --- | --- | --- |
| Storage：`useStorage()` 与其驱动配置 | `nitropack/runtime` | `nitro/storage` |
| **Caching：`defineCachedEventHandler()`、`defineCachedFunction()`** | **`nitropack/runtime`** | **`nitro/cache`** |
| Server plugins | `defineNitroPlugin()` from `nitropack/runtime` | `definePlugin()` from `nitro` |
| Nitro 自己的 runtime 钩子（`request`、`error`） | `useNitroApp().hooks` from `nitropack/runtime` | `useNitroHooks()` from `nitro/app` |
| Lazy handlers：`lazyEventHandler()` | `h3` | `nitro/h3` |

升级那一节再补了一句提醒，语气是"以后要回来改"：

> For a handler that needs h3 or Nitro helpers that `nuxt/server` does not provide, such as `readMultipartFormData`, `proxyRequest`, `useStorage` or `defineCachedEventHandler`, keep importing `defineEventHandler` and those helpers from `h3` (or `nitropack/runtime`) for that handler. Code that does this is tied to the server runtime and needs revisiting on Nuxt 5.

**这类代码绑定在服务端 runtime 上，Nuxt 5 时要重新审视。**

## 能用在哪种文件里

"能不能用"之外还有"在哪能用"。官方那条限制是对 `nuxt/server` 写的，但同一节把服务端代码的三种位置讲清了，套在缓存函数上一样的成立：

- **被服务端构建器打包的代码**：`server/` 下的路由与中间件等，这是这些 runtime API 的正常落点。
- **不能出现在应用侧**：官方原话是"Importing `nuxt/server` from a Vue component, a plugin or the `shared/` directory fails the build"，报错会把你指回 `#app`、`#imports`、`$fetch` 和 `useFetch`。同一节还补了一句：它的类型在这些上下文里**照样能解析**——这正是 `$fetch` 知道你的服务端路由返回什么类型的来源。**类型能过不等于运行得到**，这一句对缓存函数同样要当心。
- **Nuxt 进程里的代码没有这些**：`nuxt.config.ts`、模块的 `setup`、传给 `addDevServerHandler` 的函数都跑在 Nuxt 进程里，官方写"`useRuntimeConfig()` and `useAppConfig()`, for example, return empty objects. Read `nuxt.options` instead."

## 三种缓存手段怎么排

| 手段 | 写在哪 | 粒度 | 官方态度 |
| --- | --- | --- | --- |
| `routeRules` 的 `swr`/`isr`/`cache` | `nuxt.config`（或页面里的 `defineRouteRules`） | **整条路由的响应** | 声明式；服务端自动装中间件与缓存处理器；规则会尽量映射到平台原生能力（451） |
| `defineCachedEventHandler()` / `defineCachedFunction()` | 服务端代码里包住某个 handler 或函数 | **单个 handler / 单个函数** | 归入"不可移植"，来源随 Nitro 版本变，Nuxt 5 需重看 |
| 直接依赖一个缓存库 | 你自己的服务端代码 | 你定 | 官方给的替代建议原句："For caching that runs anywhere, depend on a cache library directly." |

排序的依据不是"哪个更强"，而是**你要不要把可移植性押上去**：路由级的问题用 `routeRules` 解决，配置一行、随平台走；只有当"缓存范围比路由更细"（同一个页面里某个昂贵函数、某个只被内部调用的 handler）才需要后两种，而后者是你自己承担的跨版本债务。

至于这两个函数各自的参数（TTL、变量名、存储、swr 行为等），**Nuxt 这一侧没有任何用法示例，也没有参数表**——文档未记载，实现细节归 Nitro 文档（452 的官方指向是 `nitro.build/config`）。

## 自查

- `nuxt/server` 的十九项工具里没有缓存；用它就必须越过官方表面，从 `nitropack/runtime`（Nitro 2）或 `nitro/cache`（Nitro 3）导入。
- 官方把缓存列进"不可移植"，并直接建议要通用就用缓存库。
- 这类代码绑定服务端 runtime，升级 Nuxt 5 时官方明说需要重新审视。
- 落点只能在被服务端构建器打包的代码里；应用侧文件 import 会构建失败，Nuxt 进程里的代码拿不到这些（类型仍解析得开是陷阱）。
- 按粒度选：路由级用 `routeRules`，函数/handler 级才用这两个，要通用就上缓存库。

## 下一篇

下一篇 453 回到用户能看见的两个指标：图片与字体。官方在这一节把 LCP 挂在未优化的图片上、把 CLS 挂在字体的度量回退上，并给了图片按重要性分两组加载的写法；第三方脚本那一节则点名它同时拖累 INP 与 LCP。

## 本文依据

- `nuxt/server` 的定位句（提供与 runtime 无关的 agnostic utilities、是第二个 runtime 表面）、十九行工具与类型清单、"These names are not auto-imported from `nuxt/server`"（服务端自动导入的同名者是 h3 的 helper）（`guide/going-further/server-imports`）
- Nitro 自有 API 那句（`defineCachedEventHandler()`、`useStorage()`、`useDatabase()` and tasks 从 `nitropack/runtime` 导入、大多不接 event）；"What Isn't Portable"一节的 Caching 整条与"depend on a cache library directly"、Storage/Server plugins/Lazy handlers 三条与"because Nuxt can't provide them for every builder"；"Server Code Only"一节的构建失败句、类型仍可解析、Nuxt 进程里返回空对象与改读 `nuxt.options`（`guide/going-further/server-imports`）
- 兼容表的 Caching 行（`nitropack/runtime` 对 `nitro/cache`）与相邻四行（`guide/modules/server-compatibility`）
- 升级指南中 "Helpers Outside `nuxt/server`" 一句（点名 `defineCachedEventHandler`，并写明"tied to the server runtime and needs revisiting on Nuxt 5"）（`getting-started/upgrade`）
- `routeRules` 与服务端缓存处理器的声明式路径（`guide/concepts/rendering`、`api/nuxt-config`）
