---
url: /posts/b0ab2157d20012471ed053a6b2f1ff0b/
title: Vue 生命周期在 Nuxt 里被切成两段
date: 2027-02-11T11:00:00+08:00
updated: 2027-02-11T11:00:00+08:00
author: cmdragon

summary:
  摘要：把 `onMounted` 或一段 `setInterval` 放进 `<script setup>`，本地好好的、线上首屏却不对——因为 Nuxt 会在两个完全不同的环境里各跑一遍你的组件。官方 `nuxt-lifecycle` 把一次访问拆成两半：**Server Lifecycle**（服务端，每次初始请求走七步）和 **Client Lifecycle**（浏览器，无论你选哪种渲染模式都走五步）。关键分歧是：服务端那一趟不执行任何 Vue 生命周期钩子，完整的 Vue 生命周期只在客户端那一半跑。这一篇先把七步和五步的全貌摆出来，后面几篇逐段拆。

categories:
  - 前端开发

tags:
  - nuxt4
  - 生命周期
  - SSR
  - 水合
  - 钩子
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

刚把 Vue 迁移过来的人最容易在这里翻车：`onMounted` 里 `document.getElementById(...)`、或 setup 顶层直接 `setInterval`，本地看着没事，服务端渲染首屏时行为不对甚至报错。原因是 Nuxt 把你的组件**在两个环境各跑一遍**，而这两遍的性质完全不同。官方 `guide/concepts/nuxt-lifecycle` 那一页就是把这件事讲透：

> The goal of this chapter is to provide a high-level overview of the different parts of the framework, their execution order, and how they work together.

## 一次访问分成两半

Nuxt 把生命周期切成两段：

- **Server Lifecycle**：服务端，"the following steps are executed for **every initial request** to your application"——每个初始请求走一遍，共七步。
- **Client Lifecycle**：浏览器，"This part of the lifecycle is fully executed in the browser, **no matter which Nuxt mode you've chosen**"——无论渲染模式选哪种，都在浏览器走满，共五步。

记住这条主轴：**Vue 组件自己的生命周期（onMounted 那套）只属于客户端那一半。** 服务端那一趟根本不走 Vue 生命周期钩子。

## 服务端七步

1. **Server plugins**（跑一次）：Nuxt 由 Nitro 驱动，Nitro 启动时执行 `server/plugins` 下的插件——捕获应用级错误、注册关闭钩子、注册请求生命周期钩子。官方 callout：服务端启动只跑一次；serverless 环境每次请求都重新 boot、插件也跟着跑，但**不被 await**。
2. **Server middleware**：`server/middleware/` 每个请求都执行，用于鉴权、日志、请求改写。`::warning`：**从中间件 return 一个值会终止请求、把返回值当响应发出**，一般应避免。
3. **App plugins**：先创建 Vue 和 Nuxt 实例，再执行 Nuxt 的应用插件（内置的 Vue Router、unhead，以及 `app/plugins/` 下无后缀和 `.server` 后缀的）。这步之后 Nuxt 调 `app:created` 钩子。
4. **Route validation**：初始化插件后、执行中间件前，若 `definePageMeta` 里定义了 `validate` 就调它（同步/异步都行），常用于校验动态路由参数。
5. **App middleware**：路由中间件（全局/命名/匿名三种）。全局中间件在初始加载时服务端和客户端都跑、之后每次客户端导航前再跑；命名与匿名只在页面指定的路由跑。服务端上的重定向会发 `Location:` 头、浏览器再发一次新请求，**除 cookie 外所有应用状态都会被重置**。
6. **Page and components**：渲染页面与组件、用 `useFetch`/`useAsyncData` 取数。**就在这一步，官方点破那条关键限制**——因为服务端没有动态更新、没有 DOM 操作，`onBeforeMount`、`onMounted` 及后续 Vue 钩子在 SSR 期间**不会被执行**。
7. **HTML Output**：数据和组件都渲染完，Nuxt 把结果和 unhead 设置合并成完整 HTML 发给浏览器。之后调 `app:rendered`，Nitro 在最终发出前调 `render:html`。

## 客户端五步

1. **App plugins**：与服务端类似，含无后缀和 `.client` 后缀的自定义插件。之后同样调 `app:created`。
2. **Route validation**：与服务端一样，走 `definePageMeta` 里的 `validate`。
3. **App middleware**：Nuxt 中间件两端都跑；要在特定环境只跑某些代码，用 `import.meta.client` / `import.meta.server` 分开。
4. **Mount Vue app and hydrate**：调 `app.mount('#__nuxt')` 挂载。若是 SSR/SSG，Vue 执行**水合**让页面可交互——重建应用（**Server Components 除外**）、把每个组件对上 DOM 节点、挂上事件监听。挂载前调 `app:beforeMount`，挂载后调 `app:mounted`。
5. **Vue lifecycle**：官方原话——"Unlike on the server, the browser executes the full Vue lifecycle." 完整生命周期只在这里跑。

## 把它对上 365 那张表

365 讲"谁在哪跑"时说页面和中间件服务端+客户端都跑、组件可强制只客户端。这页就是那结论的来路：同一次访问，服务端那趟把插件、校验、中间件、渲染、取数走一遍拼成 HTML，客户端那趟重新建实例、水合、然后才开始跑真正的 Vue 生命周期。**服务端只做"拼 HTML"，客户端才做"变活"。**

## 自查

- 一次访问＝服务端七步（每初始请求）＋客户端五步（任何模式都走满）。
- 服务端七步：Nitro 插件 → 服务端中间件 → 应用插件 → validate → 路由中间件 → 渲染取数 → HTML 输出。
- 客户端五步：应用插件 → validate → 路由中间件 → 挂载+水合 → 完整 Vue 生命周期。
- 核心分歧：`onMounted` 这类 Vue 钩子服务端不跑，完整生命周期只在客户端第 5 步。

## 下一篇

下一篇 369 专讲七步里最反直觉那条：SSR 期间 `onBeforeMount`/`onMounted` 为什么不执行，以及官方给的判断口径。

## 本文依据

- 全章目标、Server Lifecycle 七步与 Client Lifecycle 五步、各步说明、serverless 插件不 await 的 callout、中间件 return 终止请求的 `::warning`、服务端重定向重置状态、`app:created`/`app:rendered`/`render:html`/`app:beforeMount`/`app:mounted` 钩子位置、水合排除 Server Components、"Unlike on the server, the browser executes the full Vue lifecycle"（`guide/concepts/nuxt-lifecycle`）。
