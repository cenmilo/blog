---
url: /posts/e7ff51269571b58efde8b52a2ca99e57/
title: crossOriginPrefetch 与 prefetchPreloadTags
date: 2027-03-08T11:00:00+08:00
updated: 2027-03-08T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：这两个实验开关都带 prefetch 字样，管的却是两件不同的事。`crossOriginPrefetch` 换的是预取的实现机制——用 Speculation Rules API 做跨域预取；`prefetchPreloadTags` 加的是预取的附带内容——链接被预取时，把目标页通过 `useHead` 或 `<NuxtImg preload>` 设的 `<link rel="preload">` 提示转发到当前文档，并统一降级成 `rel="prefetch"`，只有 `as="image"` 保留 preload 但剥掉 `fetchpriority`。后者默认关闭，官方给的理由是与 `<NuxtLink>` 默认的可见性预取叠加会一次触发大量跨路由预取，而且转发的图片请求不带 `Purpose: prefetch` 头，会按普通流量计入图片 CDN 配额。

categories:
  - 前端开发

tags:
  - nuxt4
  - 预取
  - experimental
  - Speculation Rules
  - 性能
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

442 那篇讲的是 Nuxt 自己那套预取。往实验开关那一页翻，还有两个名字里带 prefetch 的键，看起来像"更强的预取"，实际各管一件事：一个改**怎么预取**，一个改**预取的时候顺带带上什么**。

## crossOriginPrefetch：换的是机制

官方给的话只有一句：

> Enables cross-origin prefetch using the Speculation Rules API.

开法：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    crossOriginPrefetch: true,
  },
})
```

NuxtLink 那一页有对应的两句。一句是配置示例前的说明——"To enable cross-origin prefetching, you can set the `crossOriginPrefetch` option in your `nuxt.config`. This will enable cross-origin prefetching using the Speculation Rules API."；另一句藏在 `prefetch` 属性的定义里：**"Used by the experimental `crossOriginPrefetch` config."**

这半句很关键：这个开关不是独立工作的，它挂在 `prefetch` 这条链上。也就是说 442 里那些关预取的写法（`noPrefetch`、`prefetch: false`、全局 `defaults.nuxtLink.prefetch: false`）对它同样有效。

Speculation Rules API 是浏览器侧的能力，官方给了指向规范的 read-more 链接，**但整页没有列哪些浏览器支持、跨域预取要满足什么响应头条件**。这两点是文档未说明的，只能开之后在目标浏览器里看 Network 面板确认。

## prefetchPreloadTags：加的是附带内容

这一项官方写了一整段，逐句都是条件与后果。原文第一句是触发前提：

> When a `<NuxtLink>` is prefetched and the destination route has payload extraction enabled (the default for prerendered and cached routes), forward any `<link rel="preload">` hints that the destination set via `useHead` (or via modules like `@nuxt/image`'s `<NuxtImg preload>`) into the current document.

拆成三个条件：**链接正在被预取**、**目标路由开了 payload extraction**、**目标页自己声明过 preload 提示**（来源只有两个：`useHead`，或像 `<NuxtImg preload>` 这样的模块产物）。

第二句是转发时的改写规则：

> Forwarded links are downgraded from `rel="preload"` to `rel="prefetch"`, apart from `as="image"` hints, which keep `rel="preload"` with any `fetchpriority` stripped.

翻译过来：转发过来的提示一律**从 preload 降级成 prefetch**，只有一个例外——`as="image"` 的图片提示保留 `rel="preload"`，但会把 `fetchpriority` **剥掉**。这正好接着 433 那篇：你在首屏图上写的 `:preload="{ fetchPriority: 'high' }"`，被别的页面转发过来时这个优先级就没了。

第三句划范围：

> Only user-defined head tags are forwarded; build-time JS/CSS chunk preloads are already handled separately by the prefetch pipeline.

**只有你自己写的 head 标签会被转发**，构建期的 JS/CSS chunk 预加载走的是另一条通道，不在这个开关管的事里。

## 为什么默认关，以及开了会挨什么

官方给了默认状态和理由，这句是决策依据：

> This flag is off by default because, combined with `prefetchOn: 'visibility'` (the `<NuxtLink>` default), it could trigger a lot of cross-route prefetches at once. Enable it once you are confident the destination preloads are worth forwarding for the links your users typically encounter.

一个列表页有五十个链接，默认策略是按可见性预取——开关一开，这五十个目标页各自的 preload 提示都会被搬进当前文档。官方的建议是先确信"这些目标页的预加载值得转发"再开。

代价有两条，都写得很具体：

- 控制台警告：**Chromium 和 Safari 会为用户根本没访问的路由上的每一个转发图片提示**打一条 `was preloaded using link preload but not used within a few seconds`。
- 流量计费：转发的图片请求**不带 `Purpose: prefetch` 头**，所以它们**按普通流量计入图片 CDN 的配额、限流和统计**。

第二条对你账单的影响是实打实的，跟用户体验无关，做 CDN 成本预算时要单独算。

## 两个键并排

| 键名 | 官方定性 | 默认 | 前置条件 | 官方写明的代价 |
| --- | --- | --- | --- | --- |
| `experimental.crossOriginPrefetch` | 用 Speculation Rules API 启用跨域预取 | 该页未写"enabled by default"，即需显式开 | 走 `prefetch` 这条链 | 浏览器支持范围文档未说明 |
| `experimental.prefetchPreloadTags` | 转发目标页的 `<link rel="preload">` 提示 | **明写关闭** | 链接被预取 + 目标路由开 payload extraction | 未访问路由刷控制台警告；图片请求按普通流量计 CDN 配额 |

## 自查

- 两个键不可互换：一个动机制（跨域、交给浏览器），一个动内容（转发目标页的提示标签）。
- `crossOriginPrefetch` 挂在 `prefetch` 上，关预取的手段照样能关掉它。
- `prefetchPreloadTags` 转发时把 preload 降级成 prefetch，`as="image"` 例外但丢 `fetchpriority`。
- chunk 预加载不在这个开关的作用范围内。
- 默认关闭是有理由的，别当"性能开关"随手打开，先看你的链接密度与 CDN 计费口径。

## 下一篇

下一篇 444 讲手动预热的两个函数：`preloadComponents` 与 `prefetchComponents`。它们只对**全局注册过的组件**生效、名字必须写 PascalCase、在服务端完全没有效果，而且官方自己写明后者的当前实现和前者一模一样。

## 本文依据

- `crossOriginPrefetch` 定性原文、配置示例、指向 Speculation Rules API 规范的 read-more（`guide/going-further/experimental-features`）
- 启用跨域预取的配置说明原文、`prefetch` 属性定义中"Used by the experimental crossOriginPrefetch config"（`api/components/nuxt-link`）
- `prefetchPreloadTags` 全节：触发前提句（链接被预取、目标路由开 payload extraction、`useHead` 与 `@nuxt/image` 的 `<NuxtImg preload>` 两个来源）、降级规则与 `as="image"` 例外、只转发用户自定义 head 标签、默认关闭的理由句、Chromium/Safari 警告原文、无 `Purpose: prefetch` 头与 CDN 配额/限流/统计、配置示例、指向 issue #34953 的 read-more（`guide/going-further/experimental-features`）
- `experimental.payloadExtraction` 三档含义与默认值（`guide/going-further/experimental-features`、`getting-started/prerendering`）
