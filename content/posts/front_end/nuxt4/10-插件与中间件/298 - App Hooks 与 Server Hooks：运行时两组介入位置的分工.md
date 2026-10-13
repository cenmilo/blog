---
url: /posts/5f7b0ec44db58cd3036e394685cf134f/
title: 298 - App Hooks 与 Server Hooks：运行时两组介入位置的分工
date: 2027-01-19T10:00:00+08:00
updated: 2027-01-19T10:00:00+08:00
author: cmdragon

summary:
  摘要：运行时钩子其实是两组：App Hooks 在 Nuxt 应用的渲染生命周期上（app:created、page:start、vue:error 那些，插件或组件订），Server Hooks 在 Nitro 服务端上（request、render:html、render:response、error 这些，服务端插件订）。它们都叫运行时，但一个作用在客户端/应用的侧的渲染时机，一个在请求处理到响应发出的服务端侧。这一篇把两组的名字、触发时机、在哪订、和典型用途并排列出来，重点分清"要改发给浏览器的 HTML/响应"用 Server Hooks 的 render:*，"要响应一次页面导航/挂载"用 App Hooks 的 page:*/app:*。

categories:
  - 前端开发

tags:
  - nuxt4
  - App-Hooks
  - Server-Hooks
  - 运行时钩子
  - Nitro
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

294 把钩子分了构建期/运行时两类。297 讲了构建期怎么注册。这一篇把"运行时"再拆细——它其实有两组：**App Hooks** 和 **Server Hooks**。名字都带 Runtime，但作用在不同的一侧，选错就订不到。

## App Hooks (Runtime)：应用的渲染生命周期

官方定位（294 也引过）：

> App hooks can be mainly used by Nuxt Plugins to hook into rendering lifecycle but could also be used in Vue composables.

订的位置：插件里 `nuxtApp.hook(...)`，或组件里 `useRuntimeHook(...)`（283）。这一组就是你最熟的——275 那张 App Hooks (Runtime) 表：`app:created`、`app:mounted`、`vue:error`、`page:start`/`page:finish`、`page:loading:start`、`link:prefetch`、`app:data:refresh` 等。它标的是"应用被创建、被挂载、页面开始/结束加载"这些**渲染与导航时机**，很多条目还区分 Server & Client / 只 Client。

## Server Hooks (Runtime)：Nitro 服务端的请求与响应

官方定位：

> These hooks are available for server plugins to hook into Nitro's runtime behavior.

订的位置：`server/plugins/` 里的 `defineNitroPlugin`，`nitroApp.hooks.hook(...)`（287 见过 `render:html`）。这组是 Nitro 那一侧的生命周期，`api/advanced/hooks` 的 "Nitro App Hooks (Runtime, Server-Side)" 表列了这些：

| 钩子 | 参数 | 何时触发 |
| --- | --- | --- |
| `request` | `event` | 收到一个请求时 |
| `render:html` | `html, { event }` | 构造 HTML **之前** |
| `render:response` | `response, { event }` | 发送响应之前 |
| `render:island` | `islandResponse, { event, islandContext }` | 构造岛屿 HTML 之前 |
| `beforeResponse` | `event, { body }` | 发送响应之前 |
| `afterResponse` | `event, { body }` | 发送响应之后 |
| `error` | `error, { event? }` | 发生错误时 |
| `close` | - | Nitro 关闭时 |
| `dev:ssr-logs` | `{ path, logs }` | 一次请求周期结束、带服务端日志数组时 |

官方也说明这组以 Nitro 文档为准（"See Nitro for all available hooks"），表是节选。

## 两组怎么分：作用在"应用渲染"还是"HTTP 请求/响应"

把两组并排放，判据就清楚了：

| | App Hooks | Server Hooks |
| --- | --- | --- |
| 订它的位置 | 插件 `nuxtApp.hook` / 组件 `useRuntimeHook` | 服务端插件 `nitroApp.hooks.hook` |
| 管的是什么 | 应用的创建/挂载/导航这些渲染时机 | 一次 HTTP 请求进来、HTML/响应怎么构造怎么发 |
| 在哪侧 | 多数两端、部分只客户端 | 服务端 |
| 典型例子 | `app:mounted`、`page:start`、`vue:error` | `request`、`render:html`、`render:response` |

两条最实用的选择：

- **"要在发出去的 HTML/响应上做手脚"** → Server Hooks 的 `render:*`。287 那个往 `html.head.push(...)` 推 meta 的例子就是订 `render:html`（构造 HTML 之前）。
- **"要响应一次页面导航/组件挂载"** → App Hooks 的 `page:*`/`app:*`。

一个是"服务端怎么把这趟请求变成响应"，一个是"前端应用生命周期走到了哪一步"——同一个用户访问里，Server Hooks 先参与（服务端渲染阶段），App Hooks 覆盖客户端导航/挂载等。

## 别和构建期钩子又混回去

三组放一起记牢，294 那条时间线：

- **Nuxt Hooks (Build Time)**：build/dev 时跑，模块/`nuxt.config` 订（`pages:extend` 等，297）。
- **App Hooks (Runtime)**：请求/导航时跑，插件/组件订（`app:*`/`page:*`/`vue:*`）。
- **Server Hooks (Runtime)**：服务端处理请求时跑，Nitro 插件订（`request`/`render:*`/`error`）。

后两组都叫"运行时"，区别就是上面那张 App vs Server 表。

## 自查

- 要改发给浏览器的 HTML 或响应内容 → Server Hooks（`render:html` / `render:response` / `beforeResponse`），订在 `server/plugins` 的 `nitroApp.hooks.hook`。
- 要响应页面挂载/导航/水合时机 → App Hooks（`app:*`/`page:*`），订在插件 `nuxtApp.hook` 或组件 `useRuntimeHook`。
- 别把这三组（构建期 / 应用运行时 / 服务端运行时）混：先问"哪个阶段、哪一侧"。
- Server Hooks 全量以 Nitro 文档为准；App Hooks 全量以 app 源文件为准，两张表都是节选。

## 下一篇

下一篇 299 讲自定义钩子：官方 Adding Custom Hooks 那节怎么用 declare module 扩展 RuntimeNuxtHooks / NuxtHooks / NitroRuntimeHooks 给自己起的钩子补类型。

## 本文依据

- Lifecycle Hooks 指南（`guide/going-further/hooks`）：App Hooks (Runtime)（"mainly used by Nuxt Plugins to hook into rendering lifecycle but could also be used in Vue composables."、`nuxtApp.hook('page:start')` 示例）；Server Hooks (Runtime)（"available for server plugins to hook into Nitro's runtime behavior."、`defineNitroPlugin` 里 `nitroApp.hooks.hook('render:html'/'render:response', ...)` 示例）。
- Lifecycle Hooks API（`api/advanced/hooks`）：Nitro App Hooks (Runtime, Server-Side) 表（`request`/`render:html`/`render:response`/`render:island`/`beforeResponse`/`afterResponse`/`error`/`close`/`dev:ssr-logs` 的参数与描述、"See Nitro for all available hooks."）；App Hooks (Runtime) 表（见 275）。
