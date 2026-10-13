---
url: /posts/7d6b0542bc71bda9d5b1b9bd2e2b0db9/
title: 382 - nuxt generate 与静态托管的适配点
date: 2027-02-16T10:00:00+08:00
updated: 2027-02-16T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：`nuxt generate` 把每个路由预渲染成纯 HTML、结果丢在 `.output/public` 里，可以扔到任何静态主机。但静态主机没有一直跑着的服务，几条适配点必须自己顾：① `ssr: true` 走 SSG（默认预渲染、还产 200/404 回退页），`ssr: false` 走纯 SPA（`<div id="__nuxt">` 是空的、丢 SEO，官方建议改用 `<ClientOnly>` 只包不能服务端渲的部分）；② 预渲染的站点**不含服务端**，用不了 server 端点，要后端逻辑就 `nuxt build`；③ 200.html/404.html 要在主机侧配 rewrite/回退，两家主机认的不一样；④ generate 命令的选项表（`--target`/`--dotenv`/`--envName` 等）。

categories:
  - 前端开发

tags:
  - nuxt4
  - generate
  - 静态托管
  - SSG
  - 部署
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

379-381 都在讲预渲染，最后收口到落地：把预渲染产物丢到静态主机上，有几件事框架替不了你。`api/commands/generate` 讲命令本身，`getting-started/deployment` 的 Static Hosting 一节讲适配点。

## `nuxt generate` 是什么

官方定性：

> The `generate` command pre-renders every route of your application and stores the result in plain HTML files that you can deploy on any static hosting services. The command triggers the `nuxt build` command with the `prerender` argument set to `true`.

它预渲染**每一个**路由、存成纯 HTML，可部署到任何静态主机；本质是带 `prerender: true` 的 `nuxt build`（379 见过）。产物在 `.output/public`。

## 命令的选项表

`nuxt generate` 完整签名带这些可选项（`ROOTDIR` 是唯一的参数，默认 `.`）：

| 选项 | 默认 | 说明（依文档） |
| --- | --- | --- |
| `--cwd=<directory>` | | 指定 Nuxt 项目根目录 |
| `--logLevel=<silent\|info\|verbose>` | | 构建期日志级别 |
| `--target=<target>` | | 服务端构建器的部署目标（如 `node-server`、`vercel`、`netlify`） |
| `--dotenv=<path>...` | | 加载哪个 `.env` 文件（相对根目录；可重复，靠后的优先） |
| `--envName=<environment>` | | 解析配置覆盖时用哪个环境（**构建时默认 `production`、跑 dev server 时默认 `development`**） |
| `-e, --extends=<layer-name>...` | | 从某个 Nuxt layer 扩展 |
| `--profile=<verbose>` | | 性能剖析，退出时写 V8 CPU profile 与 JSON 报告；`--profile=verbose` 出完整控制台报告 |

`--envName` 这行顺带证实了 363 那条：`$production` 块在构建时默认生效、`$development` 在 dev 时默认生效。`--target` 是配给服务端构建器的部署目标（331 讲过 preset）。

## 适配点一：SSG 还是纯 SPA

deployment 的 Static Hosting 说静态托管有两条路：

- **`ssr: true` 的 SSG**：构建期预渲染路由，这是 `nuxt generate` 的默认行为；还会生成 `/200.html` 和 `/404.html` 两个 SPA 回退页（能在客户端渲染动态路由或 404，但你**可能得在主机上配一下**）。
- **`ssr: false` 的纯 SPA**：预渲染出来的页面在 `<div id="__nuxt"></div>` 处是**空的**，会丢掉预渲染的大部分 SEO 好处。官方建议：**别整站 `ssr: false`**，而是用 `<ClientOnly>` 只包那些确实没法服务端渲染的部分（如果有）。

若就是想要纯 SPA 又不预渲染路由，`ssr: false` 下 `nuxt generate` 会输出 `.output/public/index.html` 加 JS bundle，像经典客户端 Vue 应用。`ClientOnly` 见 376。

## 适配点二：预渲染站没有服务端

这是最容易上线才发现的一条。365 那页 `::warning` 讲过：

> If you prerender your app with the `nuxt generate` or `nuxt build --prerender` commands, then you will not be able to use any server endpoints as no server will be included in your output folder. If you need server functionality, use `nuxt build` instead.

`nuxt generate` / `nuxt build --prerender` 的输出**不含服务器**，所以任何 `server/api` 端点都不可用。要服务端功能就得用 `nuxt build`（产物带一台 Nitro 服务器）。这也是 367 那条"hybrid 在 `nuxt generate` 下不可用"的同一个根因——没有一直跑的服务器，就没有服务端注册和缓存层。

## 适配点三：200.html 与 404.html 要在主机上配

静态主机需要一个 HTML 壳来处理客户端路由和不存在的路径，Nuxt 发两个回退：

- `200.html`：给不匹配的路径用，让**客户端路由**去处理这个 URL。
- `404.html`：给应当保留 404 状态码的路径用。

`nuxt generate` 和 `nuxt build --prerender` 会自动生成这两个；若你用 `nuxt build` 配 routeRules 只预渲染部分路由，要显式加回退：

```ts [nuxt.config.ts]
routeRules: {
  '/200.html': { prerender: true },
}
```

两个回退默认都是空壳；把 `experimental.prerenderErrorPages` 打开，就能让服务端把 `error.vue` 渲进 `404.html`（361 相关）。最关键一句：**有的主机用 `200.html`、有的用 `404.html`、有的两个都能配——部署后去查你自己主机的静态回退/rewrite 设置。**

## 自查

- `nuxt generate` ＝ 预渲染每个路由成纯 HTML（带 `prerender:true` 的 `nuxt build`），产物 `.output/public`。
- 两条路：`ssr: true` SSG（默认、产 200/404）；`ssr: false` 纯 SPA（`#__nuxt` 空、丢 SEO，官方建议改 `<ClientOnly>` 局部处理）。
- 预渲染输出不含服务器 → 用不了 server 端点，要后端就 `nuxt build`（同 367 hybrid 不可用的根因）。
- 200/404 回退默认空壳，部分预渲染要显式 `prerender` 回退；`experimental.prerenderErrorPages` 渲 error.vue 进 404；主机 rewrite/回退设置各不同，部署后必查。

## 下一篇

下一篇 383 回到渲染，讲路由切换动画的渲染层原理：`<NuxtPage>` 的过渡怎么套 Vue `<Transition>`、`page:transition:finish` 钩子、以及 view transitions API 那条实验线。

## 本文依据

- `nuxt generate` 定义（pre-render every route / triggers `nuxt build` with `prerender: true`）、ROOTDIR 与七个选项表（含 `--envName` 构建默认 production、dev 默认 development；`--target`；`--profile`）（`api/commands/generate`）。
- Static Hosting 两条路（`ssr: true` SSG 产 200/404 vs `ssr: false` 纯 SPA 空 `#__nuxt`、建议 `<ClientOnly>`）、`ssr: false` 下 generate 输出 index.html、200/404 语义与自动生成、部分预渲染显式加回退、`experimental.prerenderErrorPages`、主机 rewrite 差异（`getting-started/deployment`）。
- 预渲染无服务器端点的 `::warning`、`nuxt build --prerender` 等价（`guide/concepts/rendering`，见 365/367）。
