---
url: /posts/9339b84f8c510d0f9c54f933d0e4cb97/
title: 524 - 上线前该看的四处：错误原因不外泄、chunk 失效、启动错误、服务端错误
date: 2027-04-04T11:00:00+08:00
updated: 2027-04-04T11:00:00+08:00
author: cmdragon

summary:
  摘要：上线前后错误行为不一样，四处要提前核对。第一处：**错误 `cause` 在开发期会保留并暴露给你的错误页，在生产"never included in error responses or the error page payload"**——线上拿到的错误比本地少一层。第二处：chunk 加载失败（网络问题或**新部署让旧的带 hash 的 chunk 地址失效**）有内置处理，`experimental.emitRouteChunkError` 有四种取值，默认是导航时重载。第三处：启动错误会调 `app:error` 钩子，Vue 错误有 `onErrorCaptured`／`vue:error`／`vueApp.config.errorHandler` 三层，而 **Nitro 服务端错误当前不能定义处理函数**，只能渲错误页。第四处：静态部署下 `404.html` 默认是空壳，要 `experimental.prerenderErrorPages` 才会把你的 `error.vue` 服务端渲进去。

categories:
  - 前端开发

tags:
  - nuxt4
  - 错误处理
  - 部署
  - chunk
  - 生产
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

本地报错误时你看到的堆栈信息很全，线上同样的操作只拿到一句提示。不是日志掉了——是官方在生产环境**故意不给**。

这四处差别在上线前值得逐条核对。

## 一：cause 在生产不外泄

错误处理页那句很短，但直接决定你的错误上报能拿到什么：

> In development, an error `cause` is preserved and exposed to your error page so you can trace the original error; in production, causes are never included in error responses or the error page payload.

拆两点：**开发期**保留 `cause` 并暴露给错误页，为的是让你能追到原始错误；**生产**里 `cause` 既不进错误响应、也不进错误页载荷。

所以，如果你打算在 `error.vue` 里读 `error.cause` 来显示详情或上报，**这段代码在生产永远读不到东西**，而且不会报错。要在线上定位原始错误，靠的不是载荷里的 cause，而是服务端自己的日志与上报通道（官方在这一句里没给替代方案，错误上报框架那条路见下一处）。

## 二：chunk 失效有内置处理

那节标题是 Errors with JS Chunks，第一句正好点出两类原因：

> You might encounter chunk loading errors due to a network connectivity failure or a new deployment (which invalidates your old, hashed JS chunk URLs).

括号里那句很关键：**新部署会让你旧的、带 hash 的 chunk 地址失效**。这就是"用户标签页开着不放，你发了新版本，他一点某个页面就报错"的机制原因。

内置处理一句：**"Nuxt provides built-in support for handling chunk loading errors by performing a hard reload when a chunk fails to load during route navigation."** —— 导航时加载失败会做一次硬重载。

可调项是 `experimental.emitRouteChunkError`，两处记载合起来是四种取值：

| 取值 | 官方行为描述 |
| --- | --- |
| `automatic`（默认行为） | 导航到新路由时 chunk 加载失败，就重载那个新路由；同时发出 `app:chunkError` 钩子 |
| `automatic-immediate` | **不等导航**，chunk 加载失败时立刻重载当前路由；官方说这适用于"不是由导航触发的 chunk 错误"，例如懒加载组件加载失败；同时明确它的副作用是"可能不需要那个 chunk 也被重载" |
| `false` | 完全不再钩进这类错误 |
| `manual` | 你自己处理，官方指向内置实现那份源码文件（`packages/nuxt/src/app/plugins/chunk-reload.client.ts`）取思路 |

`automatic-immediate` 那句权衡值得抄给团队：**换掉"必须靠导航才恢复"的代价，是多出一些不必要的重载。** 你的站如果有懒加载组件在首屏就加载，默认取值可能不够，这时才考虑 `automatic-immediate`。

## 三：三层能钩、一层不能

错误处理页把可钩的层列清楚了。Vue 错误那一节给的三层：

| 入口 | 官方描述 |
| --- | --- |
| `onErrorCaptured` | 用它来介入 Vue 错误 |
| `vue:error` 钩子 | 有错误冒泡到顶层时调用；那条 `::note` 补充：**它基于 `onErrorCaptured` 生命周期钩子** |
| `vueApp.config.errorHandler` | 用错误上报框架时在插件里给一个全局处理函数；官方强调 **"It will receive all Vue errors, even if they have been handled"** |

最后一行是上线时最该配的那个——**即使错误已被处理，它照样收得到**。官方给的就是这个用法场景："handle error, e.g. report to a service"。

启动错误是另一套：Nuxt 在启动应用出错时调 `app:error`，官方列了包含范围五项——跑 Nuxt 插件、处理 `app:created` 与 `app:beforeMount` 钩子、SSR 时把 Vue 应用渲成 HTML、客户端挂载（**并当场建议这一种还是用 `onErrorCaptured` 或 `vue:error` 处理**）、处理 `app:mounted` 钩子。

**只有一层你不能自定义处理**：Nitro Server Errors 那一节只有一句话——

> You cannot currently define a server-side handler for these errors, but can render an error page.

注意 "cannot currently"，这是官方自己留的余地。所以服务端这一层的错误目前**只有一个出口：错误页**。

## 四：静态部署下错误页是空壳

这一条属于部署侧与错误处理的交叉处，官方在两节都写了。渲染概念页：**"By default `404.html` is an empty shell, so your `error.vue` (and its layout and data) only appear once the client app has booted."**，要它带内容就设 `experimental.prerenderErrorPages`，Nuxt 会在构建期用一个合成的 404 错误渲 `error.vue` 写进 `404.html`；还能传 400-599 之间的状态码数组（示例 `[404, 500]`）多渲几页。

但那条随之而来的限制更要记住：同一个文件服务所有缺失路径，所以**错误页不能依赖请求**——`useRoute()`、`useRequestURL()` 以及按当前路径取的数据在预渲染的 HTML 里是构建期的值，水合时才纠正。官方给的两条应对是：请求相关的标记包进 `<ClientOnly>`，请求相关的数据用 `import.meta.prerender` 跳过。

## 自查

- 生产里 `cause` 不进错误响应也不进错误页载荷；想在 `error.vue` 读它做上报是行不通的，线上要靠服务端日志与 `vueApp.config.errorHandler`。
- chunk 失效的内置处理只在"导航时"重载；要处理懒组件那种非导航触发的失败才用 `automatic-immediate`，代价是不必要的重载；`manual` 时官方只给了内置实现源码的链接。
- 启动错误钩 `app:error`（含插件、两个早期钩子、SSR 渲染、挂载、`app:mounted`）；客户端挂载这一种官方建议改用 `onErrorCaptured`/`vue:error`。
- Nitro 服务端错误当前不能定义处理函数，只能渲错误页。
- 静态部署的 `404.html` 默认空壳，`experimental.prerenderErrorPages` 才会把 `error.vue` 渲进去；渲进去之后它不能依赖请求，靠 `<ClientOnly>` 与 `import.meta.prerender` 兜。

## 下一篇

下一篇 525 收线上排错可用的手段：官方给的只有 source map 与调试器两节，而生产环境能读到什么、`--inspect` 为什么帮不上，正是这两节的边界。
