---
url: /posts/63f1ce202e8717e2720014b8b72501aa/
title: 335 - 从环境变量到 runtimeConfig：一整条链路
date: 2027-01-31T11:00:00+08:00
updated: 2027-01-31T11:00:00+08:00
author: cmdragon

summary:
  摘要：一个配置值从 .env 到代码里被读到，中间经过好几段，搞不清就会遇到"我明明设了环境变量怎么读不到"。这一篇把链路一次画完：.env 只在 dev/build/generate 被读、值在 nuxt.config 里通过 runtimeConfig 声明（应用代码用 runtimeConfig 而不是裸 process.env）；生产运行阶段 .env 不再被读（serverless/edge 可能没文件系统），要你在部署环境自己设；runtimeConfig 的值被匹配的 NUXT_ 前缀环境变量在运行时覆盖；删掉 .env 里的变量不会取消已设过的值；本地用 nuxt preview 做生产预览是个便利例外。

categories:
  - 前端开发

tags:
  - nuxt4
  - 环境变量
  - runtimeConfig
  - env
  - 部署
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


"配一个值、代码里能读到"这句话，在 Nuxt 里其实跨了构建期和运行期两个阶段。把这条链路走一遍，很多"设了没生效"的问题就通了。

## 第一段：.env 只在 dev/build/generate 被读

官方 env 那一节：

> Nuxt CLI has built-in `.env` support via c12 in development mode and when running `nuxt build` and `nuxt generate`.

> if you have a `.env` file in your project root directory, it will be automatically loaded **at dev, build and generate time**. Any environment variables set there will be accessible within your `nuxt.config` file and modules.

也就是说 `.env` 只在**开发、构建、生成**这三个时机被读进来。有一条要记（note）：

> removing a variable from `.env` or removing the `.env` file entirely will not unset values that have already been set.

删了 `.env` 里的变量，不会把"已经设过的值"取消掉——这是"我删了它怎么还在"的答案。

## 第二段：值进 runtimeConfig，不在应用代码裸读 process.env

env 那一节有一条 important，直接定应用代码该怎么读：

> In your application code, you should use Runtime Config instead of plain env variables.

所以链路是：`.env` 里的变量 → 在 `nuxt.config` 的 `runtimeConfig` 里声明对应键 → 代码里用 `useRuntimeConfig()` 读（340/341）。不是组件里到处 `process.env.XXX`。

一个细节（use-runtime-config note）：`.env` 里的变量在**开发**和**build/generate** 阶段可以用 `process.env` 在应用里读到——但那是那两阶段的便利，不代表生产也行（看第三段）。

## 第三段：生产不读 .env，得你在部署环境设

这是最容易翻车的一段。官方：

> **After your server is built**, you are responsible for setting environment variables when you run the server. Your `.env` files will not be read at this point.

为什么这么设计，官方给的理由：

> This design decision was made to ensure compatibility across various deployment environments, some of which may not have a traditional file system available, such as serverless platforms or edge networks like Cloudflare Workers.

因为 serverless/edge 可能压根没有传统文件系统给你放 `.env`。所以生产阶段你得用部署平台的方式设环境变量：命令行传（`NODE_ENV=production DATABASE_HOST=... node .output/server/index.mjs`）、shell 配置、或 Vercel/Netlify/AWS 的控制台/CLI。

还有一条决定"能不能覆盖 runtimeConfig"（important）：

> `runtimeConfig` won't pick up environment variables that don't start with `NUXT_` in production.

生产里，只有 `NUXT_` 前缀的环境变量会被 runtimeConfig 认。这条下一段展开。

## 第四段：NUXT_ 前缀在运行时覆盖 runtimeConfig

runtime-config 那一节：

> Runtime config values are **automatically replaced by matching environment variables at runtime**.

两个前提：

1. 键要在 `nuxt.config` 里声明过（"This ensures that arbitrary environment variables are not exposed to your application code."）。
2. 环境变量是 `NUXT_` 开头、全大写、用下划线表示层级和大小写。`runtimeConfig.public.apiBase` 对应 `NUXT_PUBLIC_API_BASE`。

命名映射的细节 342 专讲，这里只要知道"覆盖发生在运行时、且只认对得上名字的 `NUXT_` 变量"。还有一条：把某个键的默认值指向"另一个名字"的变量（`myVar: process.env.OTHER_VARIABLE`）只在构建期有效、运行时会断（因为运行时只认命名对得上的）。

## 一段特例：本地生产预览

env 那一节给的例外（Production Preview）：

> For local production preview purpose, we recommend using `nuxt preview` since using this command, the `.env` file will be loaded into `process.env` for convenience. Note that this command requires dependencies to be installed in the package directory.

本地想验"生产行为"，用 `nuxt preview`——它会图方便把 `.env` 读进 `process.env`。但这是预览的便利，别误以为生产也这样。另外：纯静态站预渲染后无法再设 runtime config。

## 全链路一图

| 阶段 | .env 被读？ | 怎么覆盖 runtimeConfig |
| --- | --- | --- |
| dev | 是（c12） | 改 .env 自动重启进程 |
| build / generate | 是 | — |
| 生产运行 | **否** | 由部署环境设的 `NUXT_*` 变量在运行时覆盖 |
| nuxt preview（本地预览） | 是（便利例外） | 装依赖时 |

## 自查

- `.env` 只 dev/build/generate 读；生产运行不读，得在部署平台自己设环境变量。
- 应用代码用 `useRuntimeConfig()` 读，不裸读 process.env；键先在 nuxt.config 声明。
- 运行时覆盖只认 `NUXT_` 前缀、名字对得上的变量；删 .env 变量不会取消已设值。
- 本地验生产行为用 nuxt preview（它会读 .env，是例外）；纯静态站预渲染后设不了 runtime config。

## 下一篇

下一篇 336 把"只在服务端有值的组合式"归一：useRequestEvent/useRequestHeaders 在浏览器分别是 undefined/空对象，一张表看清哪些取请求信息的 API 服务端专用。
