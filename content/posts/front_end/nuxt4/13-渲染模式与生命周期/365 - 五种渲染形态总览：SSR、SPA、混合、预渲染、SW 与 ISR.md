---
url: /posts/8d5272edfdd3aeea57445f0f760a3b0a/
title: 五种渲染形态总览：SSR、SPA、混合、预渲染、SW 与 ISR
date: 2027-02-10T11:00:00+08:00
updated: 2027-02-10T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：SSR、CSR、SSG、ISR、hybrid——这些词在 Nuxt 里各自指什么、默认走哪条、能不能混着来，这一篇先认全名字。官方把渲染形态分四大块：universal rendering（服务端把 Vue 跑成完整 HTML 再水合）、client-side rendering（浏览器才生成 HTML）、hybrid rendering（按路由用 route rules 定模式与缓存）、edge-side rendering（Nitro 把渲染推到 CDN 边缘，官方说它更像部署目标而非独立形态）。默认的 universal 一行配置就能切成纯客户端（`ssr: false`）。还澄清一件事：同一个 `.vue` 里哪些代码在服务端跑、哪些只在浏览器跑。

categories:
  - 前端开发

tags:
  - nuxt4
  - 渲染模式
  - SSR
  - 水合
  - routeRules
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

打开 Nuxt 文档或跟别人讨论，一堆缩写砸过来：SSR、CSR、SSG、ISR、hybrid、edge……它们不是六套互不相干的系统，官方其实只分了几大块，其余是块里的策略。这一篇先把这些名字对应到官方 `guide/concepts/rendering` 那一页的四个小节，让你知道"默认是哪条、想换怎么说"。

## 先定义"渲染"

官方开头把词义钉死：

> Both the browser and server can interpret JavaScript code to turn Vue.js components into HTML elements. This step is called **rendering**.

**渲染 = 把 Vue 组件变成 HTML**。服务器和浏览器都能干这件事。Nuxt 支持"两边都能渲染"的 universal 和"只在浏览器渲染"的 client-side 两条主线，另加 hybrid 和 edge 两种能力。

> By default, Nuxt uses **universal rendering** ... but you can switch rendering modes in one line of configuration.

**默认是 universal**，换形态只要一行配置。

## 一、Universal Rendering（SSR + 水合）

这就是常说的服务端渲染。浏览器请求一个 URL，Nuxt 在服务器环境跑 Vue 的 JS，返回一整页拼好的 HTML（如果这页提前生成过，也可能直接从缓存返回）。

关键的第二步叫**水合（Hydration）**：

> Once the HTML document has been downloaded ... The same JavaScript code that once ran on the server runs on the client (browser) **again** in the background now enabling interactivity ... This is called **Hydration**.

浏览器先把服务器给的静态 HTML 显示出来，再让同一份 JS 在浏览器跑第二遍、把事件监听挂上去，页面就开始能交互。好处是首屏快 + 内容已在 HTML 里、爬虫不用等 JS 就能索引。

## 二、Client-Side Rendering（SPA）

传统 Vue 应用的做法：浏览器先下载并解析全部 JS，再由 Vue 生成 HTML。用户得等 JS 下完、跑完才看到内容。Nuxt 里切到这条，只要：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  ssr: false,
})
```

官方给了一条配套提示：`ssr: false` 时应该在 `~/spa-loading-template.html` 放一个 HTML，作为水合完成前的加载屏。

## 三、Hybrid Rendering（按路由分模式与缓存）

以前整个应用只能选一种模式（全 universal 或全 client-side）。Hybrid 让你**按路由**用 Route Rules 分别决定：哪些页构建期预渲染、哪些只客户端渲染、哪些走缓存策略。官方那个例子把几种策略并排：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    '/': { prerender: true },
    '/products': { swr: true },
    '/products/**': { swr: 3600 },
    '/blog': { isr: 3600 },
    '/blog/**': { isr: true },
    '/admin/**': { ssr: false },
    '/api/**': { cors: true },
    '/old-page': { redirect: '/new-page' },
  },
})
```

所以你看到的 **预渲染（`prerender`）、SW（`swr`，stale-while-revalidate）、ISR（`isr`，incremental static regeneration）** 不是三种独立"渲染模式"，而是 hybrid 里 route rules 的几个策略。它们的逐项含义、`ssr: false` 对服务端包体的影响，367 展开。

## 四、Edge-Side Rendering（更像部署目标）

ESR 借 Nitro 把渲染推到离用户更近的 CDN 边缘节点：请求被最近的边缘服务器截住，由它生成 HTML 再回给使用者，缩短数据物理距离、降延迟。官方给它定位得很清楚：

> Note that ESR is more a deployment target than an actual rendering mode.

也就是说 ESR 不是"第五种和 SSR 并列的渲染形态"，而是一种**部署到哪**的选择；并且 Hybrid 也能配合 ESR 用（"Hybrid Rendering can be used when using Edge-Side Rendering with route rules."）。

## 一个 `.vue` 里，谁在哪跑

初学最糊涂的一件事：universal 模式下，同一个组件文件到底哪些在服务端跑、哪些只在浏览器跑。官方用一段 counter 说清：

- `const counter = ref(0)`：**服务端和浏览器都执行**（首屏请求时服务端初始化它、水合时浏览器再初始化一次）。
- `handleClick` 的函数体：**只在浏览器执行**（首次请求时它的从没被跑过，只是水合时才绑到按钮上）。

官方又按目录补了一遍：

> [Middlewares] and [pages] run in the server and on the client during hydration. [Plugins] can be rendered on the server or client or both. [Components] can be forced to run on the client only as well. [Composables] and [utilities] are rendered based on the context of their usage.

| 东西 | 在哪跑 |
| --- | --- |
| 中间件、页面 | 服务端 + 客户端（水合期间） |
| 插件 | 服务端 / 客户端 / 两者皆可 |
| 组件 | 可强制只在客户端 |
| composables、utils | 按使用它们的上下文决定 |

## 自查

- 四大块：universal（SSR+水合）、client-side（`ssr: false`）、hybrid（route rules 按路由分策略）、edge（ESR，偏部署目标）。
- 默认 universal，切纯客户端一行 `ssr: false`。
- 预渲染/SW/ISR 是 hybrid 里的 route rules 策略，不是并列的独立渲染模式。
- 同一个组件里：`ref` 初值两端都跑、事件回调体只浏览器跑；页面/中间件两端跑、组件可强制只客户端。

## 下一篇

下一篇 366 讲每种形态适用的真实场景与代价：SSR 的性能/SEO 好处对上开发约束/服务器成本，CSR 的开发快/便宜/离线对上首屏慢/SEO 弱，以及该给什么站选什么。

## 本文依据

- 渲染定义、默认 universal 与一行切换、hydration 定义、`ssr: false` 与 SPA loading 模板、counter 谁在哪跑、按目录的执行位置表、route rules 多策略示例、ESR "more a deployment target than an actual rendering mode"（`guide/concepts/rendering`）。
