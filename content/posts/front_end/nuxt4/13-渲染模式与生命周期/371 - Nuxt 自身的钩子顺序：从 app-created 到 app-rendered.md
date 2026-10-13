---
url: /posts/f79b2328be55b168c03e14cdbea72c0f/
title: 371 - Nuxt 自身的钩子顺序：从 app-created 到 app-rendered
date: 2027-02-12T11:00:00+08:00
updated: 2027-02-12T11:00:00+08:00
author: cmdragon

summary:
  摘要：除了 Vue 那套 `onMounted`（369 说服务端不跑），Nuxt 自己还有一批运行时钩子，沿着服务端七步、客户端五步依次触发：`app:created`、`app:redirected`、`app:rendered`、`render:html`、`app:beforeMount`、`app:mounted`、`app:suspense:resolve`、`page:start`/`page:finish` 等。这些叫 **App Hooks (Runtime)**，要用 `nuxtApp.hook(...)` 订阅，不是 Vue 生命周期钩子。关键差别是每个钩子标注的 Environment 不同：有的 Server & Client 各触发一次，有的只 Server，有的只 Client。这一篇把一次初始访问里它们出现的先后串起来，并和 294 那张全量钩子表对接。

categories:
  - 前端开发

tags:
  - nuxt4
  - 钩子顺序
  - app-created
  - app-rendered
  - 生命周期
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

368-370 讲的都是"Vue 生命周期在 Nuxt 里的行为"。但 Nuxt 还有**另一套它自己的钩子**——`app:created`、`app:rendered`、`page:finish` 这些。它们跟着 368 那两半按顺序触发，是你在请求流程里插手的正规入口。这一篇把顺序摆出来。

## 先分清两类钩子

- **Vue 生命周期钩子**：`onMounted` 等，369 已讲——服务端不跑。
- **Nuxt App Hooks (Runtime)**：`app:*`、`vue:*`、`page:*` 这批，靠 `nuxtApp.hook(name, cb)` 订阅（注册入口见 294/297），不是 Vue 的那套。294 有全量清单，这里只按**一次初始访问的出现顺序**串相关的这些。

区分它们很重要：`app:beforeMount` 和 `onBeforeMount` 名字像，但前者是 Nuxt 钩子、客户端挂载整个 app 前触发一次，后者是某个组件的 Vue 钩子、服务端不跑。

## 服务端这一半的顺序

跟 368 服务端七步对齐，Nuxt 钩子在这里依次冒头：

1. **`app:created`**（参数 `vueApp`）：Vue/Nuxt 实例创建好后调用——在应用插件之后。Environment 标的是 **Server & Client**，所以服务端和客户端各触发一次。
2. 中间是 validate、路由中间件。若服务端发生重定向，**`app:redirected`**（Server）在 SSR 重定向前调用。
3. **`app:rendered`**（参数 `renderContext`，Server）：SSR 渲染完成时调用。
4. **`render:html`**：这是 **Nitro 的服务端钩子**（Nitro App Hooks, Server-Side），在"构造 HTML 之前"调用，可注入脚本、改 meta；再往外还有 `render:response`（发送响应前）。368 那步说的"Nuxt 合并 HTML 发出前调 `render:html`"就是它。

## 客户端这一半的顺序

跟 368 客户端五步对齐：

1. **`app:created`**（再次）：客户端也建一次实例，所以这钩子客户端又触发一次。
2. **`app:beforeMount`**（参数 `vueApp`，Client）：挂载 app 前调用，"called only on client side"。
3. 挂 `app.mount('#__nuxt')` + 水合。
4. **`app:mounted`**（参数 `vueApp`，Client）：Vue 应用在浏览器初始化并挂载后调用。之后才轮到真正的 Vue 生命周期（368 第 5 步）。
5. **`app:suspense:resolve`**（参数 `appComponent`，Client）：Suspense resolved 事件时（和 372 的 Suspense 直接相关）。

页面级的还有 **`page:start`** / **`page:finish`**（Client，都在 `NuxtPage` 里的 Suspense pending/resolved 事件上触发），以及 `page:loading:start` → `page:loading:end`（后者在 `page:finish` 之后）。

## 一张顺序表

| 时机 | 钩子 | Environment | 参数 | 何时 |
| --- | --- | --- | --- | --- |
| 实例建好 | `app:created` | Server & Client | `vueApp` | 应用插件后 |
| 服务端重定向前 | `app:redirected` | Server | - | SSR 重定向前 |
| SSR 渲染完 | `app:rendered` | Server | `renderContext` | 渲染完成 |
| 构造 HTML 前 | `render:html` | Server（Nitro） | `html, { event }` | 发出前可改 HTML |
| 挂载前 | `app:beforeMount` | Client | `vueApp` | 仅客户端 |
| 挂载后 | `app:mounted` | Client | `vueApp` | Vue 应用挂到浏览器 |
| Suspense resolve | `app:suspense:resolve` | Client | `appComponent` | 水合后 |
| 页面 pending | `page:start` | Client | `pageComponent?` | NuxtPage 内 Suspense |
| 页面 resolved | `page:finish` | Client | `pageComponent?` | NuxtPage 内 Suspense |

错误类另有 **`app:error`**（Server & Client，致命错误时）、`app:error:cleared`、以及 `vue:error`（Vue 错误冒泡到根组件）、`vue:setup`（Nuxt root 的 setup 初始化时，回调必须同步）。

## Environment 那一列为什么重要

它就是 369 那条规律的钩子版：**只标 Client 的，服务端根本不触发**（`app:beforeMount`、`app:mounted`、`app:suspense:resolve`、`page:*`）；**只标 Server 的，客户端不触发**（`app:redirected`、`app:rendered`）；标 **Server & Client** 的，一次初始访问里两端各来一遍（`app:created`）。所以想"每次请求在服务端干点事"要挑 Server 或 Server & Client 的，别订一个只 Client 的。

## 自查

- Nuxt App Hooks (Runtime) 不是 Vue 生命周期，用 `nuxtApp.hook` 订（294/297）。
- 服务端顺序：`app:created` →（`app:redirected`）→ `app:rendered` → Nitro `render:html`。
- 客户端顺序：`app:created` → `app:beforeMount` → 挂载/水合 → `app:mounted` → `app:suspense:resolve` → `page:start`/`page:finish`。
- Environment 列决定它在不在服务端触发：Client-only 的钩子服务端永不跑。

## 下一篇

下一篇 372 讲取数和这条生命周期怎么咬合：Nuxt 用 Vue 的 `<Suspense>` 挡住导航直到异步数据就绪，以及 `await` 到底改变了什么、没改变什么。

## 本文依据

- 生命周期各步点名的钩子位置 `app:created`/`app:rendered`/`render:html`/`app:beforeMount`/`app:mounted`（`guide/concepts/nuxt-lifecycle`）。
- App Hooks (Runtime) 表内各钩子的 Arguments/Environment/Description（`app:created` `vueApp` Server & Client "Called when initial vueApp instance is created"；`app:rendered` `renderContext` Server；`app:beforeMount`/`app:mounted` Client；`app:suspense:resolve`、`page:start`、`page:finish`、`app:redirected`、`app:error`、`vue:setup`、`vue:error`）与 Nitro App Hooks (Server-Side) 的 `render:html`（`api/advanced/hooks`）。全量钩子清单见 294。
