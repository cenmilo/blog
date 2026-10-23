---
url: /posts/0308c61568410075c9e62491c48f1ea0/
title: 385 - navigationRepaint 与 ssrStreaming：导航体验相关开关
date: 2027-02-17T10:00:00+08:00
updated: 2027-02-17T10:00:00+08:00
author: cmdragon

summary:
  摘要：这两个 experimental 开关优化的是不同指标，别混。`navigationRepaint`（**默认开**）在导航前先等一个 animation frame，让浏览器重绘、确认用户交互，"can reduce INP when navigating on prerendered routes"——改善的是 Interaction to Next Paint。`ssrStreaming`（324 详讲过）改善的是 TTFB：服务端先把 HTML 壳（head、样式、preload 提示、入口脚本）立即发出，再用 Vue `renderToWebStream` 流式吐 body。两个都对首屏体验有用，但作用维度不同：一个管"点了多快有反馈"，一个管"多久收到第一字节"。这一篇把 `navigationRepaint` 讲透，`ssrStreaming` 按体验开关的角度带过并指回 324。

categories:
  - 前端开发

tags:
  - nuxt4
  - navigationRepaint
  - ssrStreaming
  - 实验特性
  - 性能
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


experimental-features 那页里，和"点了链接之后感受"最相关的两个开关是 `navigationRepaint` 和 `ssrStreaming`。它们常被放在一起谈，但一个改善 INP、一个改善 TTFB，是两码事。

## navigationRepaint：导航前等一帧

官方描述：

> Wait for a single animation frame before navigation, which gives an opportunity for the browser to repaint, acknowledging user interaction. It can reduce INP when navigating on prerendered routes.

拆开：

- 它在做的事——**导航开始前先等一个 animation frame**，让浏览器有机会重绘，先把用户的点击"确认"了（视觉上有个反馈），再去切页。
- 改善的指标——**INP**（Interaction to Next Paint，从交互到下一帧绘制的延迟），尤其在**预渲染路由**上导航时。
- **默认开启**，想关：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    navigationRepaint: false,
  },
})
```

预渲染路由没有服务端往返（379/382），切页几乎是即时的，这时"等一帧再导航"能让浏览器先把点击态画出来，INP 数字更好看。关掉它就少等这一帧。

## ssrStreaming：先发壳再流 body

`ssrStreaming` 在 324 已详拆过（流式边界、退回缓冲的清单、bot 处理、head 晚注册变客户端补丁等），这里只按"体验开关"的角度重述一句它的定位：

> Enables SSR streaming to dramatically improve Time to First Byte (TTFB). When enabled, the server sends the HTML shell (including `<head>`, styles, preload hints, and entry scripts) immediately, then streams the rendered body content progressively using Vue's `renderToWebStream`.

开法：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    ssrStreaming: true,
  },
})
```

改善的指标是 **TTFB**——服务端不等整页渲完，先把 HTML 壳（head、样式、preload 提示、入口脚本）立刻发出，body 内容再用 Vue 的 `renderToWebStream` 一段段流出去。用户更早收到第一字节、更早看到骨架。机制细节（哪些请求会退回缓冲渲染、状态头必须壳刷出前设好、bot 不流式等）看 324。

## 两者别混

| 开关 | 默认 | 改善指标 | 做什么 |
| --- | --- | --- | --- |
| `navigationRepaint` | 开 | INP | 导航前等一帧，让浏览器先重绘、确认点击 |
| `experimental.ssrStreaming` | 关 | TTFB | 先发 HTML 壳、body 流式渲染 |

一个作用在**已经加载之后、点了到反馈之间**（INP），一个作用在**第一次拿到字节之前**（TTFB）。它俩互补，不是同一件事。

## 自查

- `navigationRepaint` 默认开：导航前等一个 animation frame、让浏览器重绘确认交互，在预渲染路由上能降 INP；`experimental.navigationRepaint: false` 关。
- `experimental.ssrStreaming` 降 TTFB：先发 head/样式/preload/入口脚本的壳，再用 `renderToWebStream` 流式吐 body；细节见 324。
- 两者指标不同（INP vs TTFB）、时机不同（点击后反馈 vs 首字节前），互补。

## 下一篇

下一篇 386 讲 SPA 加载模板：`ssr: false` 时页面在水合完成前是空白，用 `spaLoadingTemplate`（或 `~/spa-loading-template.html`）填一段加载 UI，顺带和 dev 的 `loadingTemplate` 分清。
