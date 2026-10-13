---
url: /posts/53aee09d7fb61697ce47df2789eef9a6/
title: 447 - writeEarlyHints：首字节前的提示头
date: 2027-03-10T09:00:00+08:00
updated: 2027-03-10T09:00:00+08:00
author: cmdragon

summary:
  摘要：`experimental.writeEarlyHints` 是这一批里官方写得最短的一项——整节只有一句"Enables writing of early hints when using node server"加一段配置示例。唯一的限定条件是 **node server**。另外要分清同名的两件事：`writeEarlyHints` 同时还是 h3 里的一个函数，官方在"要越过 `nuxt/server` 表面"那一节把它和 `readRawBody`、`proxyRequest`、`fetchWithEvent` 们列在一起，明确说要**从 `h3` 导入**，因为"nuxt/server isn't a re-export of h3"。哪些提示会被提前写、默认值、与其他渲染路径的关系，文档都没写。

categories:
  - 前端开发

tags:
  - nuxt4
  - Early Hints
  - writeEarlyHints
  - 首屏
  - experimental
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

首屏 HTML 到手之后，浏览器才从 `<head>` 里读到那些 `<link rel="preload">`，然后开始取 JS chunk。也就是说，**服务端拼 HTML 的那段时间里，浏览器是干等着的**。

Early hints 想解决的就是这段等待：文档还没拼完，先把提示头发出去。Nuxt 这边对应的开关只有一句话。

## 官方整节就一句

`## writeEarlyHints` 的全部内容：

> Enables writing of early hints when using node server.

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    writeEarlyHints: true,
  },
})
```

这一句里唯一的限定条件是 **when using node server**。也就是说官方把启用条件写死在"跑的是 node 服务端"上。

这一节没写默认值句（这一节里凡是默认打开的开关，都会明写一句"This flag is enabled by default, but you can disable this feature"），所以它的默认状态按"要显式开"处理。至于**具体哪些提示会被提前写出去、写到哪一层、在非 node 部署上是什么行为、和流式渲染或预渲染怎么相互作用，文档未说明**——这四项都没在这一节出现。

## 别和 h3 的那个函数混了

`writeEarlyHints` 这个名字在文档里出现在两个地方，指两件不同的事。

一个是刚才那个**配置键**，在 `experimental` 节下。

另一个是 **h3 的函数**。官方在讲服务端导入边界那一节写着：

> For a handler that needs h3's own helpers, such as `readRawBody`, `readMultipartFormData`, `assertMethod`, `proxyRequest`, `fetchWithEvent` or `writeEarlyHints`, or that reads `event.node` or `event.waitUntil()`, import `defineEventHandler` from `h3` for that handler.

给的写法是这样一个文件：

```ts [server/api/upload.post.ts]
import { defineEventHandler, readMultipartFormData } from 'h3'

export default defineEventHandler(async (event) => {
  const parts = await readMultipartFormData(event)
  return { received: parts?.length ?? 0 }
})
```

为什么要特意点出来，因为官方同一节还有一句理由：

> `nuxt/server` isn't a re-export of h3, so if you need something else that h3 and Nitro offer, such as the server lifecycle (`defineNitroPlugin` and `defineNitroErrorHandler`) or Nitro's own runtime hooks, you would import it directly.

也就是说，**这个函数不在 `nuxt/server` 提供的表面里**，要在自己的服务端路由里手动写提示头，得从 `h3` 导入。两条线各自独立：配置键决定 Nuxt 自己的渲染路径要不要写 early hints，函数决定你在自定义 handler 里能不能主动写。

## Nuxt 现有的提示有哪些

要判断"提前发出去"到底发的是什么，可以先看文档列出的提示清单。`noScripts` 生效时被整体省掉的东西，官方列成四条（`guide/going-further/features`）：

- Nuxt 入口 `<script>` 标签
- 入口 chunk 的 import map
- 内联的 payload `<script>`（开了 payload extraction 时还包括 payload 的 preload link）
- **JavaScript 资源提示**，也就是给 JS chunk 的 `preload` 与 `prefetch` 链接

最后那条就是 early hints 在概念上要抢时间的那一类。**但官方没有把这两者写在一起**——`writeEarlyHints` 那一节没有列出它写哪些头，`noScripts` 那节也没提 early hints。这条对应关系是我按名字和用途做的推断，不是文档记载，落地上要靠实测确认。

## 怎么验证有没有真的发出去

开关只有一行，验证要两步，而且**必须在 node 服务端产物上做**（官方那句限定条件的直接推论）：

1. 构建并用 node 起产物（`nuxt build` 之后按 029 那篇说的方式跑起来），不要用 dev 模式——开发模式的渲染路径与产物不是一回事。
2. 用能看见**信息性响应**的方式打一次：浏览器开发者工具的 Network 面板里，这条文档请求下面有没有一个 103 状态的同源条目；或者用 `curl -v` 看原始响应序列里有没有 103。

再补一句实测时容易误判的地方：本地直连和经过反向代理/CDN 时看到的结果可能不一样。**文档没有写任何中间层对这类头的处理方式**，所以如果线上看不到而本机能看到，先把代理链一段一段剥开测，不要直接下结论说配置没生效。

## 自查

- 官方全文一句：启用条件是 **using node server**；默认值句缺失，按需显式开处理。
- 配置键与 h3 的 `writeEarlyHints()` 是两件事；后者要从 `h3` 导入，`nuxt/server` 不复出它。
- 能被抢时间的提示主要是 JS chunk 的 `preload`/`prefetch` 链接，但这个对应关系文档没写。
- 验证只能在 node 产物上做，且要能看到 1xx 信息的工具；中间层行为文档未说明。

## 下一篇

下一篇 448 讲 payload 的体积从哪两头控制：`renderJsonPayloads`（默认开着，管的是 JSON payload 能不能还原复杂类型）与 `purgeCachedData`（默认开着，管路由跳转时 `useAsyncData` 与 `nuxtApp.static.data` 的缓存要不要清掉），再加上 `payloadExtraction` 三档各自的取舍。

## 本文依据

- `## writeEarlyHints` 整节：定义原文一句（含 "when using node server"）与配置示例；对照那一节其他默认开启开关的 "This flag is enabled by default" 措辞（`guide/going-further/experimental-features`）
- h3 helpers 清单中出现 `writeEarlyHints`、"import `defineEventHandler` from `h3` for that handler" 与 `server/api/upload.post.ts` 示例、"`nuxt/server` isn't a re-export of h3" 的理由句（`guide/going-further/server-imports`）
- `noScripts` 生效时被省略的四类内容清单（入口 script、import map、内联 payload 与 payload preload link、给 JS chunk 的 preload/prefetch 资源提示）与"CSS is unaffected"（`guide/going-further/features`）
