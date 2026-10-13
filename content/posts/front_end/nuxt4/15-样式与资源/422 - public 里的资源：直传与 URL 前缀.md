---
url: /posts/a54221c6c3c0e3fcfbed6f9913d8d081/
title: 422 - public 里的资源：直传与 URL 前缀
date: 2027-03-01T11:00:00+08:00
updated: 2027-03-01T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`public/` 的一句话定义："Files contained within the `public/` directory are served at the root and are not modified by the build process."——**按根路径直出、构建过程不改它们**。官方给的适用判据也很具体：适合那些**必须保留原名**的文件（如 `robots.txt`），或**基本不会变**的文件（如 `favicon.ico`）；正式文件里还补了一句"Files in `public/` keep their original filenames."。URL 前缀这件事有两个必须记住的不对称：静态字面量（如 `<img src="/img/nuxt.png">`）会被包一层、渲染时应用 `app.baseURL`；运行时拼出来的路径（`` `/img/${name}.png` ``）**不会**被加前缀，要自己用 `useRuntimeConfig().app.baseURL` 配合 `joinURL` 拼。另外 `app.cdnURL` 会影响 `public/` 资源的解析，但 `app.head` 里写死的 `href: '/favicon.ico'` 是字面量、**不走 cdnURL 解析**。

categories:
  - 前端开发

tags:
  - nuxt4
  - public目录
  - baseURL
  - cdnURL
  - 静态资源
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`robots.txt` 必须待在固定地址、`favicon.ico` 不能带 hash 后缀——这类文件放 `assets/` 就全错了（421 讲过那里没有静态 URL）。它们属于 `public/`，而 `public/` 的 URL 前缀还有一条不对称，很容易在子路径部署时踩。

## 定义：直出，且不被构建改动

`directory-structure/public` 第一句：

> Files contained within the `public/` directory are served at the root and are not modified by the build process. This is suitable for files that have to keep their names (e.g. `robots.txt`) *or* likely won't change (e.g. `favicon.ico`).

三个信息：**按根路径提供**、**构建过程不改它们**、适用判据是"必须保名的"或"基本不变的"。getting-started/assets 那句"content is served at the server root as-is"以及 421 提到的一条正面对照"Files in `public/` keep their original filenames."是同义复述。

目录与用法官方给的是这一对：

```bash [Directory structure]
-| public/
---| favicon.ico
---| og-image.png
---| robots.txt
```

```vue [app/app.vue]
<script setup lang="ts">
useSeoMeta({
  ogImage: '/og-image.png',
})
</script>
```

访问方式就是根 URL：`public/img/nuxt.png` 变成静态地址 `/img/nuxt.png`。历史口径也记一下——Nuxt 2 里这个目录叫 `static/`。

## 该放什么、不该放什么

判据其实只有一条：**需要以固定 URL 存在的东西**。所以：

- `robots.txt`、`favicon.ico`、`og-image.png` 这类被外部按约定地址来取的（爬虫、社交平台的分享抓取），必须在 `public/`——它们要的 URL 不能带 hash，也不可能去读你的构建产物。
- 需要被打包器处理的东西（要 hash、要按 import 解析）属于 `assets/`（421）。
- 本地字体官方给的位置也是 `public/`：丢进 `public/fonts`，样式表里用 `url()` 引（415 那个 `@font-face` 例子就是 `url('/fonts/FarAwayGalaxy.woff')`）。

## 前缀的不对称：字面量会补，运行时拼的不补

421 引过这段机制，这里从 `public/` 的角度再看一次。官方：

> A public path such as `/img/nuxt.png` is wrapped so that your `app.baseURL` is applied when the page renders ... This resolution only happens for literal paths the build tool can see.

也就是**静态字面量**（`<img src="/img/nuxt.png">`）会被包一层，渲染时补上 `app.baseURL`——这正是 361 那条 `NUXT_APP_BASE_URL` 能在部署时才定的原因：前缀是运行时应用的，路径是构建期识别的。

不对称出现在运行时拼路径。官方原话：

> A runtime-built public path like `/img/${name}.png` is therefore **not** prefixed with `app.baseURL`. If your application is deployed below the origin root, prefix it yourself with `useRuntimeConfig().app.baseURL` (for example via `joinURL`).

所以站点跑在子路径（`/app/`）下时，`/img/${name}.png` 会指到站点根、丢掉那一段前缀。处置办法官方直接给了：用 `useRuntimeConfig().app.baseURL` 自己拼，例子提到 ufo 的 `joinURL`。

要按运行时才知道的文件名引 public 资源，官方给的是"引用 by URL"这一条（public 的文件不需要被处理或 hash），不需要 `?url` 那套（那是 421 里 bundled assets 的办法）。

## cdnURL 这条也有坑

361 讲的 `app.cdnURL` 与 `public/` 有一处联动，官方在 SEO 那一节的 `::note` 里写得很具体：

> When you set `app.cdnURL`, assets in your `public/` directory (including `favicon.ico`) are served from that CDN. Nuxt resolves public assets against `cdnURL`, falling back to `app.baseURL`. However, a static `app.head` link such as `href: '/favicon.ico'` above is a literal path and is **not** resolved against `cdnURL`. To point the favicon at the resolved location, build the `href` from runtime config with `useHead()` in `app.vue`.

三条：`public/` 资源会随 `cdnURL` 走、回落到 `app.baseURL`；但 `app.head` 里写死的 `/favicon.ico` 是字面量、**不参与这个解析**；要让 favicon 指向解析后的位置，就在 `app.vue` 里用 `useRuntimeConfig().app` 拿 `cdnURL`/`baseURL` 拼出来再交给 `useHead()`。

## 自查

- `public/`＝按根路径直出、构建不改、保留原文件名；适用判据是"必须保名"（robots.txt）或"基本不变"（favicon.ico）。
- 访问方式是根 URL：`public/img/nuxt.png` → `/img/nuxt.png`；Nuxt 2 里这个目录叫 `static/`。
- 静态字面量路径会被包一层、渲染时补 `app.baseURL`（所以部署期才定的 base 也能生效）；**运行时拼出来的路径不补**，要自己用 `useRuntimeConfig().app.baseURL` + `joinURL` 拼。
- 需要固定 URL 的放这里（含本地字体 `public/fonts`）；要 hash、要 import 解析的放 `assets/`。
- 设了 `app.cdnURL` 时 `public/` 资源从 CDN 出（回落 `app.baseURL`），但 `app.head` 里写死的 `/favicon.ico` 不参与解析——要用 `useHead()` 从 runtime config 拼。

## 下一篇

下一篇 423 起继续 415-434 那几篇：图片与字体的加载策略（`getting-started/assets` 与 Nuxt Image 那条线），把资源这一章的最后一块补齐。

## 本文依据

- "Files contained within the `public/` directory are served at the root and are not modified by the build process."、适用判据（have to keep their names / likely won't change）、目录结构示例与 `useSeoMeta({ ogImage: '/og-image.png' })`、Nuxt 2 称 `static/`（`directory-structure/public`）；两目录对照与"content is served at the server root as-is"、`/img/nuxt.png` 静态 URL 示例、Static vs. Dynamic `src`（字面量被包以应用 `app.baseURL`；仅构建器可见的字面量才解析）、运行时拼路径不带前缀与 `useRuntimeConfig().app.baseURL`+`joinURL`、"Files in public/ keep their original filenames"、public 资源按 URL 引用（不需处理或 hash）（`getting-started/assets`）；本地字体放 `public/fonts` 与 `@font-face` 示例（`getting-started/styling`）；`cdnURL` 解析 `public/` 资源、`app.head` 字面量不解析、改用 `useHead()` 从 runtime config 拼（`getting-started/seo-meta`，另见 361）。
