---
url: /posts/124addee2f72b4d48ee63716feffa8f6/
title: 460 - 错误有四个来源：Vue 渲染、启动、Nitro 服务端、JS chunk
date: 2027-03-14T10:00:00+08:00
updated: 2027-03-14T10:00:00+08:00
author: cmdragon

summary:
  摘要：官方开篇把 Nuxt 里避不掉的运行时错误列成**四个来源**，因为它们发生在不同上下文：Vue 渲染生命周期内的错误（SSR 与 CSR 都有）、启动阶段的错误（SSR 加 CSR）、Nitro 服务端生命周期内的错误（`server/` 目录）、以及下载 JS chunk 失败。四类各自的接手点也不同：第一类用 `onErrorCaptured`、`vue:error` 钩子或 `vueApp.config.errorHandler`，第二类是 `app:error` 钩子，第三类官方明写**目前不能定义服务端处理器、只能渲染错误页**，第四类由框架内置硬重载、可用 `experimental.emitRouteChunkError` 改成 `false` 或 `manual`。呈现方式也是一条规则：未处理的致命错误会渲染 JSON 响应或全屏错误页。

categories:
  - 前端开发

tags:
  - nuxt4
  - 错误处理
  - 钩子
  - 水合
  - chunk
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

同一个"报错了"，在 Nuxt 里可能是四件不同的事。你把它们用同一个 `try/catch` 兜，结果有的能兜住、有的根本到不了你的代码。

官方这一节第一句就给了解释的前提：

> Nuxt is a full-stack framework, which means there are several sources of unpreventable user runtime errors that can happen in different contexts:

**unpreventable**（无法预防）、**different contexts**（发生在不同上下文）——所以入口是分开的四个。

## 四类原文

官方列的四条是这样的：

- Errors during the Vue rendering lifecycle (SSR & CSR)
- Server and client startup errors (SSR + CSR)
- Errors during Nitro server lifecycle (`server/` directory)
- Errors downloading JS chunks

括号里那三个标注是这一篇的全部要点：第一类**两端都会发生**，第二类是**启动阶段的两端**，第三类只属于 `server/` 目录，第四类是**下载**——不是执行。

顺带一条官方 `::tip` 的术语：**SSR** 是 Server-Side Rendering，**CSR** 是 Client-Side Rendering。

## 第一类：Vue 渲染生命周期

官方给的接手点有三个，层次不同：

| 接手点 | 官方说明 | 特点 |
| --- | --- | --- |
| `onErrorCaptured` | You can hook into Vue errors using `onErrorCaptured`. | Vue 的组合式 API 生命周期钩子，在组件里用 |
| `vue:error` | Nuxt provides a `vue:error` hook that will be called if any errors propagate up to the top level. | 只在错误**冒泡到顶层**时调用；`::note` 写明它**基于 `onErrorCaptured`** |
| `vueApp.config.errorHandler` | If you are using an error reporting framework, you can provide a global handler through `vueApp.config.errorHandler`. **It will receive all Vue errors, even if they are handled.** | 面向报错上报服务；关键在括号那句——**已被处理的错误也会进来** |

官方给的插件写法把后两条放在同一个文件里：

```ts [plugins/error-handler.ts]
export default defineNuxtPlugin((nuxtApp) => {
  nuxtApp.vueApp.config.errorHandler = (error, instance, info) => {
    // handle error, e.g. report to a service
  }

  // Also possible
  nuxtApp.hook('vue:error', (error, instance, info) => {
    // handle error, e.g. report to a service
  })
})
```

三个都收到同样的 `(error, instance, info)` 三参形态。选哪个取决于你要不要"已经被 `onErrorCaptured` 处理掉的错误"——要上报全部，就用 `errorHandler`。

## 第二类：启动错误

一句定义加一份清单：

> Nuxt will call the `app:error` hook if there are any errors in starting your Nuxt application.

官方接着列了"This includes"的**五项**：

1. running Nuxt plugins
2. processing `app:created` and `app:beforeMount` hooks
3. rendering your Vue app to HTML (during SSR)
4. mounting the app (on client-side)，但官方紧跟一句"though you should handle this case with `onErrorCaptured` or with `vue:error` hook"
5. processing the `app:mounted` hook

第 4 项是唯一带建议的：**挂载阶段的错误建议走第一类那两个入口**，不是 `app:error`。这和 019 那篇的生命周期顺序能对上——服务端到"渲成 HTML"，客户端到"挂载"与"mounted"。

## 第三类：Nitro 服务端生命周期

这一类官方只有一段，而且是一段限制：

> You cannot currently define a server-side handler for these errors, but can render an error page, see the Render an Error Page section.

**目前不能为这类错误定义服务端处理器**，能做的只有渲染一个错误页。换句话说，`server/` 目录里出的错，不要指望有一个"Nuxt 侧的服务端错误钩子"可以挂。

同一页下面还有一句对端上的错误处理有直接影响的定性：

> When Nuxt encounters a fatal error (any unhandled error on the server, or an error created with `fatal: true` on the client) it will either render a JSON response (if requested with `Accept: application/json` header) or trigger a full-screen error page.

**fatal** 的定义在这里被写死：服务端任何未处理的错误，或客户端带 `fatal: true` 创建的错误。响应形态按 `Accept` 头分岔。官方还补了一句版本差异：**设了 `future.compatibilityVersion: 5` 时，页面错误总是触发错误页，而服务端路由或服务端中间件里抛出的错误返回 JSON 响应。**

## 第四类：下载 JS chunk

成因官方给了两种：网络中断，以及**新部署让旧的带 hash 的 chunk URL 失效**。

默认行为是内置的：

> Nuxt provides built-in support for handling chunk loading errors by performing a hard reload when a chunk fails to load during route navigation.

改法是把 `experimental.emitRouteChunkError` 设成 `false`（完全不接这类错误）或 `manual`（自己处理）；官方还直接给了自动实现的文件链接供参考。

这一类要单独理解：**439 讲过切得越细、取 chunk 的机会越多**，所以代码分割策略变了，这类错误的频率也会变；部署新版的瞬间，旧页面上的用户点进一条还没下载的路由就会撞上它。

## 哪一类去哪接

| 类别 | 官方给的接手点 | 这一篇之后 |
| --- | --- | --- |
| Vue 渲染生命周期 | `onErrorCaptured`、`vue:error`、`vueApp.config.errorHandler` | 局部兜住用 `<NuxtErrorBoundary>` |
| 启动（含 SSR 渲染 HTML、插件、三个 app 钩子） | `app:error` | 客户端挂载那项建议改用第一类入口 |
| Nitro 服务端生命周期 | **无服务端处理器可用**，只能渲染错误页 | `error.vue`（464） |
| JS chunk 下载失败 | 内置硬重载 / `experimental.emitRouteChunkError` | 469 的错误码那批 |

呈现层（全屏错误页 vs JSON）与局部层（`<NuxtErrorBoundary>` 的 `#error` 插槽，只在客户端阻止冒泡）是两套互不替代的东西，095 那篇讲过后者。

## 自查

- 四类错误来自四个上下文，不是一个总入口；括号里的 SSR/CSR 标注决定了代码能跑在哪。
- 想要"连已被处理的 Vue 错误也上报"，只有 `vueApp.config.errorHandler` 满足，`vue:error` 只收冒泡到顶层的。
- `app:error` 覆盖五项，其中挂载那项官方建议改用 `onErrorCaptured` 或 `vue:error`。
- Nitro 服务端生命周期错误目前**没有**服务端处理器可定义，只能落到错误页。
- fatal＝服务端未处理错误 或 客户端 `fatal: true`；响应按 `Accept: application/json` 分岔，cv5 下页面错误必出错误页、服务端路由错误返回 JSON。
- chunk 错误的两条成因里包含"新部署使旧 hash URL 失效"，默认硬重载，可关可自管。

## 下一篇

下一篇 461 讲读当前错误的那个组合式函数：`useError()` 不收任何参数，返回一个 `Ref`，**在客户端和服务端都可用**，错误状态变化时它会自动更新；它给出的 `NuxtError` 接口是六个字段——和 `error.vue` 那个错误对象的字段清单不是同一份写法。
