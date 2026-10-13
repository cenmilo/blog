---
url: /posts/56ab0b87b6bdff09d6005fc6617d96c4/
title: 453 - 图片与字体：CLS 与 LCP 的处理
date: 2027-03-12T09:00:00+08:00
updated: 2027-03-12T09:00:00+08:00
author: cmdragon

summary:
  摘要：官方性能那一节只在三处点名指标，位置很能说明问题：LCP 挂在"未优化的图片"上、CLS 挂在字体的**度量回退**上、INP 与 LCP 一起挂在第三方脚本上。这一篇按这三条线整理：图片这一头的抓手是 433 讲过的两组属性与 `width`/`height` 驱动的缩放；字体这一头是 Nuxt Fonts 自动做的五步——解析字体、生成 `@font-face`、代理并缓存到 `/_fonts`、创建回退度量、打进构建并设长缓存头，其中"创建回退度量"就是官方唯一直接指向 CLS 的动作。脚本那条线留给 454。

categories:
  - 前端开发

tags:
  - nuxt4
  - LCP
  - CLS
  - 字体优化
  - 图片优化
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

Lighthouse 报告里两条红色：一条 LCP、一条 CLS。你想知道分别该动哪个文件——图片、字体，还是脚本。

官方性能那一节其实已经把指标和能力绑好了，只是绑在正文里而不是一张大表里。把绑定关系抽出来，第一件事就清楚了。

## 三个指标各自挂在谁身上

| 指标 | 官方原话里的归属 | 出处位置 |
| --- | --- | --- |
| LCP | 未优化的图片"对 LCP 分数有显著负面影响" | Images 一节开头 |
| LCP 与 INP | 第三方资源（统计、视频嵌入、地图、社媒集成）"can significantly degrade user experience and negatively impact INP and LCP scores" | Scripts 一节开头 |
| CLS | Nuxt Fonts 的"Creates fallback metrics – Adjusts local system fonts to match web fonts, reducing layout shift" | Fonts 五步的第 4 步 |

三条之外**官方没再点名任何指标**，所以"这个改动会改善 FID""那个能提 FCP"这类说法在这份文档里没有依据。

术语补一句：**CLS** 是加载过程中布局跳动的累计程度——文字被后来的字体顶下去、图片到位时下面整块内容位移，都算。

## LCP 这条线：图片的四个抓手

433 那篇已经把两段官方示例和六个属性逐个落地了，这里只归纳"哪一件事影响 LCP 的哪一段"：

| 抓手 | 官方依据 | 作用在 LCP 的哪一段 |
| --- | --- | --- |
| 按重要性分组加载 | 首屏就要送达的与可以晚点的分两类；第一组 `:preload="{ fetchPriority: 'high' }"` 加 `loading="eager"` | 决定这张图**什么时候开始下载** |
| 现代格式 | "Converts `src` to provider optimized URLs with modern formats such as WebP or Avif" | 决定**下载多少字节** |
| `width` 与 `height` | "Automatically resizes images based on `width` and `height`" | 决定**下载的尺寸对不对**（原图 2000 px 却显示 200 px 就是白下载） |
| `sizes` | "Generates responsive sizes when providing `sizes` option" | 响应式尺寸要显式给 `sizes` 才生成，官方没写它会自动猜 |

前置条件还是 423 那条：**这两个组件来自 Nuxt Image 模块**，要先 `npx nuxt module add image`。没装模块却照抄属性，什么都不会发生。

## CLS 这条线：字体的五步

Nuxt Fonts 那一节的定性是"automatically optimize your fonts (including custom fonts) and remove external network requests for improved privacy and performance"，并说明 reduced layout shift 靠的是底层包 **fontaine**。

然后是官方编号列出的五步，触发条件是"when it encounters a font-family declaration"——**它处理你 CSS 里所有声明**：

| 步 | 官方原文 | 做什么 |
| --- | --- | --- |
| 1 Resolves fonts | Looks for font files in public/, then checks web providers like Google, Bunny, and Fontshare. | 先找 `public/` 里的字体文件，再查 Google、Bunny、Fontshare 这些在线提供商 |
| 2 Generates `@font-face` rules | Injects CSS rules to load fonts from the correct sources. | 注入规则，让字体从正确的来源加载 |
| 3 Proxies & caches fonts | Rewrites URLs to `/_fonts`, downloads and caches fonts locally. | URL 重写到 `/_fonts`，本地下载并缓存（这就是"去掉外部网络请求"的落点） |
| 4 Creates fallback metrics | Adjusts local system fonts to match web fonts, reducing layout shift (CLS). | **调整本地系统字体去匹配 Web 字体的度量**——直接降 CLS |
| 5 Includes fonts in build | Bundles fonts with your project, hashing file names and setting long-lived cache headers. | 打进项目、文件名加 hash、设长时效缓存头 |

第 4 步值得慢看：它在**Web 字体还没到位的那段时间**里，让系统字体用一套尽量接近的度量先顶上，所以字体加载完成时文字位置不跳。这一条是官方全页里唯一直接把某个动作和 CLS 绑起来的句子。

第 3 步还顺带解决另一件事：外链请求没了，隐私与依赖外部域的失败风险也一起少一项。**官方没有给"用了模块之后 CLS 能降到多少"这类数字。**

## 两条线之外：能不能把 JS 整个拿走

官方在 Lazy Hydration 那节末尾给了一句组合方案，位置和措辞都值得关注：

> For content and marketing sites that need little or no client-side interactivity, you can go further and combine prerendering, the `noScripts` route rule, server components and lazy hydration to ship near-zero JavaScript.

这是**四条能力叠起来**的配方，不是某个开关。它同时把图片与字体这两条线压到最简——页面几乎不发 JS。代价在 456 那篇讲可访问性冲突时再说：交互一旦需要，就不能这么砍。

## 症状先归线，再动手

| 你看到的 | 归哪条线 | 该查的官方条目 |
| --- | --- | --- |
| 首屏大图迟迟出现、报告里 LCP 元素就是它 | LCP / 图片 | 分组加载那三件套；`width`/`height` 是否与实际显示尺寸一致 |
| 图片到位时下面内容整体下移 | 不是官方点名的线 | 文档未把这类位移归给图片；先确认容器尺寸是否写了 |
| 文字先是一种字体、随后变成另一种且位置跳动 | CLS / 字体 | Fonts 第 4 步（fallback metrics）是否生效 |
| 页面上有统计、视频嵌入或地图，交互后卡顿 | INP 与 LCP / 脚本 | 454 |

第二行要特别说明：**CLS 这一节官方只绑了字体**，图片引起的布局位移没有被这份文档写成任何机制，别把它当成"文档说图片会造成 CLS"。

## 自查

- 官方页只点名 LCP、CLS、INP，绑定分别是图片、字体度量回退、第三方脚本。
- 图片四个抓手：分组加载、现代格式、`width`/`height` 缩放、显式给 `sizes`；前提是装了 Image 模块。
- 字体五步里只有第 4 步（fallback metrics，靠 fontaine）直接对 CLS，第 3 步把外部字体请求变成 `/_fonts` 自托管。
- 没有官方数字告诉你能降多少；效果要在自己的报告里比对。
- 想更彻底就叠四条能力做 near-zero JS，但这会牵动可访问性。

## 下一篇

下一篇 454 讲第三条线：第三方脚本。官方给的不是"怎么加脚本"，而是 `useHead` 的 `script` 数组、`useHeadSafe` 那句对不可信来源的提醒，以及 Nuxt Scripts 那种"注册后由它安排加载"的路子——包括 `trigger: 'manual'` 先不加载、用 `proxy` 排队事件、`onLoaded` 等脚本就绪三种写法。

## 本文依据

- 三个指标的归属原话：图片与 LCP 那句、第三方脚本与 INP/LCP 那句、字体第 4 步与 CLS 那句；Nuxt Image 的定性（plug-and-play、内置优化器或你的图片 CDN）与 NuxtImg 五条增强复述；Nuxt Fonts 的定性、fontaine、五步编号清单原文、多 provider 可插拔那句；near-zero JavaScript 的四条组合原话（`guide/best-practices/performance`）
- `<NuxtImg>` 五条能力与"outputs a native `img` tag directly (without any wrapper around it)"、Setup 命令；`<NuxtPicture>` 的三句（`api/components/nuxt-img`、`api/components/nuxt-picture`）
- 两组图片属性示例及其注释、`experimental.prefetchPreloadTags` 默认关闭（`guide/best-practices/performance`、`guide/going-further/experimental-features`）
