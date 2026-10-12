---
url: /posts/2794533ac925a6e000a20e5b45d02503/
title: 渲染模式与 SEO：SSR 与预渲染的选择
date: 2027-02-18T10:00:00+08:00
updated: 2027-02-18T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：SEO 的唯一硬道理：爬虫能索引到的，只有它抓回来的那份 HTML 里有啥。SSR 把整页 HTML 连内容一次给全（"crawlers can index it without overhead"）；纯客户端渲染要等浏览器跑完 JS 才有内容，爬虫第一次来抓不等你，索引/更新更慢；预渲染则是构建期就把 HTML 写死、天生可索引。head/meta 那套（Unhead 驱动的 `useHead`/`useSeoMeta`）也必须被**渲染进 HTML** 才有用，所以它们跟着渲染模式一起决定 SEO 上限。这一篇把"要 SEO → 选哪种渲染"讲清，并给出 `ssr: false` 那类页面怎么局部补救（用 `<ClientOnly>` 只包不能服务端渲的部分，而不是整站 CSR）。

categories:
  - 前端开发

tags:
  - nuxt4
  - SEO
  - 渲染模式
  - 预渲染
  - useSeoMeta
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

366 已经给过一句结论——要 SEO 就用 SSR、别纯客户端。这一篇把"为什么"和"具体怎么选"落到渲染模式上，并且把 head/meta 这条线接上：光有正确的 `<title>`/meta 不够，它们得真的出现在爬虫拿到的 HTML 里。

## 一条不变的原理

爬虫抓到的是 HTML、不是一段还没跑的 JS。所以 SEO 上限直接由"那份 HTML 里有没有内容"决定——这就是 365/366 反复出现的那条：

- **SSR（universal）**：服务器把整页 HTML 交给浏览器，"Web crawlers can directly index the page's content"，官方把它定位成"任何你想快速被索引的内容"的好选择。
- **纯客户端（CSR）**：内容要等浏览器下载、解析、跑完 JS 才出现；"search engine crawlers won't wait for the interface to be fully rendered on their first try to index the page"，所以索引和更新更慢。

一句话：能被立即索引的内容，得在爬虫拿到的那一刻就已经在 HTML 里。

## 预渲染：更早、更省地可索引

预渲染（379/382）更进一步——HTML 在构建期就写成静态文件，请求时直接给成品，天生带全文。对纯内容页，这比每次现算 SSR 更适合被索引，也更快。

所以三种渲染取向对 SEO 的排序大致是：**预渲染 ≥ SSR ≫ 纯 CSR**。内容型、要被索引的页往前两者靠；重交互、不需索引的页（366 点名的 SaaS/后台/游戏）才考虑 CSR。

## `ssr: false` 丢 SEO，别整站这么干

382 那条又回到这里：`ssr: false` 预渲染出来的页面在 `<div id="__nuxt">` 处是空的，官方说"you will lose many SEO benefits of prerendering your site"，并建议——

> so it is suggested instead to use `<ClientOnly>` to wrap the portions of your site that cannot be server rendered (if any).

**别整站 `ssr: false`**。如果只是某些块没法服务端渲，用 `<ClientOnly>`（376）局部包住，其余仍走 SSR/预渲染保住 SEO。

## head/meta 也要被渲进 HTML

SEO 不只是正文，还有 `<title>`、meta、`og:` 这些。Nuxt 的 head 管理由 Unhead 驱动（seo-meta 页开篇）：

- `app.head`（nuxt.config）静态定制全站 head，官方 `::important` 点明"does not allow you to provide reactive data"，适合放不变的默认标题、`lang`、favicon。
- 要响应式/按页的 meta，用 `useHead()` / `useSeoMeta()`（支持响应式输入）在 `app.vue` 或页面里设。

关键点接回渲染模式：这些标签是**渲染进服务器那份 HTML** 才轮到爬虫读。走 SSR/预渲染，`<head>` 里的 SEO 标签就在首屏 HTML 里；走纯 CSR，它们要等 JS 跑起来才出现——和正文同样慢。所以"配了 useSeoMeta 但线上搜不到"，先查这页到底 SSR 还是 CSR。

（顺带 324 那条也在这条线上：开 `ssrStreaming` 时，晚于 HTML 壳注册进 head 的标签会变成客户端补丁，某些爬虫读不到——SEO 关键的 canonical/社交 meta 要保持同步注册或把该路由退出流式。细节见 324。）

## 场景速查

| 你要的 | 渲染取向 | SEO 后果 |
| --- | --- | --- |
| 内容/营销页要索引 | 预渲染或 SSR（默认） | HTML 里就有全文+meta，可索引 |
| 登录后工具/后台 | 该路由 `ssr: false` | 不需要被索引，放弃 SEO 换开发简单 |
| 只想让个别块跳过 SSR | `<ClientOnly>` 局部包 | 其余仍进 HTML，SEO 保住 |
| 每页 title/description | `useHead`/`useSeoMeta` + SSR | 标签随 HTML 出，爬虫读得到 |

## 自查

- SEO 由"爬虫抓到的 HTML 里有没有内容/标签"决定：SSR 一次给全文、CSR 要等 JS 所以慢、预渲染构建期写死最快可索引。
- 排序：预渲染 ≥ SSR ≫ 纯 CSR。内容页往前两者靠。
- `ssr: false` 丢 SEO，别整站用；个别块用 `<ClientOnly>` 局部处理。
- head/meta（Unhead：`app.head` 静态、`useHead`/`useSeoMeta` 响应式）也得被渲进 HTML 才生效——纯 CSR 下同样要等 JS。

## 下一篇

下一篇 389 做 P13 排错：把"页面闪烁、二次请求、状态串页"三类现象各自的成因和检查顺序归位。

## 本文依据

- SSR "crawlers can directly index"、CSR "search engine crawlers won't wait for the interface to be fully rendered"、预渲染构建期出静态 HTML（`guide/concepts/rendering`、`getting-started/prerendering`）；`ssr: false` 空 `#__nuxt` 与"lose many SEO benefits ... use `<ClientOnly>`"（`getting-started/deployment`、见 382）；head 管理 Unhead、`app.head` 静态不支持响应式的 `::important`、`useHead`/`useSeoMeta` 响应式（`getting-started/seo-meta`）；流式下 head 晚注册变客户端补丁见 324。
