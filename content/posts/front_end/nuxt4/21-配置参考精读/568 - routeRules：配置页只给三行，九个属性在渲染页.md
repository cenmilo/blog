---
url: /posts/bebf349b10fc293170c23268ecf7df27/
title: 568 - routeRules：配置页只给三行，九个属性在渲染页
date: 2027-04-19T10:00:00+08:00
updated: 2027-04-19T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：`routeRules` 在 `api/nuxt-config` 里的全部记载是三行：**"Global route options applied to matching server routes."**、**"Experimental: This is an experimental feature and API may change in the future."**，以及一个指向 Nitro route rules 文档的 See；**没有类型行、没有默认值、没有示例**。能对照到的清单在别处：渲染概念页列出**九个属性**（`redirect: string`、`ssr: boolean`、`cors: boolean`、`headers: object`、`swr: number | boolean`、`isr: number | boolean`、`prerender: boolean`、`noScripts: boolean`、`appMiddleware: string | string[] | Record<string, boolean>`），并给了一份八条规则的配置示例；`defineRouteRules` 那一节给的是"写在页面文件里的简写"与它的实验性前提（要开 `experimental.inlineRouteRules`）。也就是说：**被官方标为实验性的，是这个功能的 API 形状，而不是"能不能用"。**

categories:
  - 前端开发

tags:
  - nuxt4
  - 路由规则
  - route-rules
  - 缓存
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

路由规则是 Nuxt 里用得最多的键之一：缓存、重定向、按路径关 SSR 全靠它。

但翻到配置参考那一节，只有三行字。

## 配置页的三行

`## routeRules` 的记载全文：

> Global route options applied to matching server routes.
>
> **Experimental**: This is an experimental feature and API may change in the future.
>
> **See**: Nitro route rules documentation

**没有 Type、没有 Default、没有 Example。** 这一节连配置页最常见的"三件套"格式都没有走完（对比 `routeRules` 的姊妹项 `### nitro.routeRules`——那里至少有一行 `**Type**: object`）。

那句 **Experimental** 的准确范围要读清楚：官方说的是 **"API may change in the future"**（API 形状可能变），不是"不要在生产用"。这与它在其他页被当成稳定能力使用是并存的：部署页用它给全站开 `isr`、预渲染页用它给 `/200.html` 开 `prerender`、渲染概念页整节围绕它讲混合渲染。

## 九个属性的实际清单

清单在渲染概念页的 Route Rules 小节，逐条按官方写法：

| 属性 | 官方说明（要点） |
| --- | --- |
| `redirect: string` | Define server-side redirects. |
| `ssr: boolean` | 关掉该区块 HTML 的服务端渲染，只在浏览器里渲 |
| `cors: boolean` | `cors: true` 自动加 cors 头；要定制输出用 `headers` 覆盖 |
| `headers: object` | 给站点某些区块加特定响应头，例如静态资源 |
| `swr: number \| boolean` | 加缓存头并缓存在服务器或反向代理，TTL 可配；到期先返回缓存、后台重生成；`true` 时加 `stale-while-revalidate` 且无 MaxAge |
| `isr: number \| boolean` | 与 `swr` 行为相同，差别是能进 CDN 缓存（当前 Netlify 或 Vercel）；`true` 时内容留到下次部署 |
| `prerender: boolean` | 构建期预渲染这些路由，并作为静态资源包含进产物 |
| `noScripts: boolean` | 禁用该区块的 Nuxt 脚本与 JS resource hint 渲染 |
| `appMiddleware: string \| string[] \| Record<string, boolean>` | 定义 Vue 应用部分里那些页面路径应/不应跑的中间件（**不含你的 Nitro 路由**） |

那份示例八条规则（`/` 预渲染、`/products` 与 `/products/**` 两种 `swr`、`/blog` 与 `/blog/**` 两种 `isr`、`/admin/**` 关 `ssr`、`/api/**` 开 `cors`、`/old-page` 重定向）**没有一条用 `headers`、`noScripts` 或 `appMiddleware`**——也就是说清单里三个属性在官方示例中缺席，**它们的取值细节只能回到上表那句说明**。

## 匹配语法：本节一个字都没写

九个属性解决"匹配到之后做什么"，**"怎么写匹配"这里没讲**。文档里的证据只有三处形状：

- 前缀通配：`/blog/**`、`/products/**`、`/admin/**`、`/api/**`。
- 全站：`/**`（部署页环境覆盖示例用的就是 `'/**': { isr: true }`）。
- 精确路径：`/`、`/products`、`/200.html`、`/old-page`。

`See` 指向的是 Nitro 的 route rules 文档，`serverHandlers` 那句也提过 `route` 遵循 **rou3** 的约定——**但 `routeRules` 的键是否用同一套语法、`:param` 这类段能不能写，Nuxt 文档没有说**。要动动态段的路由规则，只能实测。

还有一条与 `appMiddleware` 有关的边界值得留意：它管的是 **Vue 应用侧的中间件**，官方在括号里明确 "that is, not your Nitro routes"——**服务端中间件不在这个属性的控制范围内**。

## 简写：写在页面文件里的 defineRouteRules

`api/utils/define-route-rules` 那一节给的是同一条规则的另一半用法：

> As a shorthand, you can also configure this in a page file using `defineRouteRules`.

并当场标注它的前提：**这是实验性功能，要用必须在配置里开 `experimental.inlineRouteRules`**。示例是页面里 `defineRouteRules({ prerender: true })`，官方说明这一句会被翻译成：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    '/': { prerender: true },
  },
})
```

**这个"翻译"关系是文档明确写的**，也就是说简写不是新能力，而是同一份 `routeRules` 的另一种书写位置。它也被列在实验性开关那份清单里（`inlineRouteRules`）。

## 自查

- 配置页的 `## routeRules` 只有三行：全局路由选项一句、Experimental（指 API 形状可能变）、指向 Nitro；**无 Type、无 Default、无 Example**。
- 九个属性的清单在渲染概念页，官方示例八条规则里没有用到 `headers`、`noScripts`、`appMiddleware`。
- `appMiddleware` 只管 Vue 应用侧中间件，官方明写不含 Nitro 路由。
- 匹配语法本节不讲，文档里能看到的只有 `/**`、`/x/**` 与精确路径三种；动态段能不能写需要实测（`route` 语法归 rou3 是 `serverHandlers` 那句给的线索）。
- 页面内简写是 `defineRouteRules`，前提是开 `experimental.inlineRouteRules`，且官方说明它会被翻译成顶层 `routeRules` 条目。

## 下一篇

下一篇 569 按"能不能做到"来核对组合用法：重定向、缓存头、代理、静态文件服务这四类里，**九个属性中没有 `proxy`、也没有 `cache`**——代理类需求在 4.x 的配置参考里没有对应属性。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## routeRules` 三行全文含 Experimental 句与 See；`### nitro.routeRules` 仅 `Type: object`；`## serverHandlers` 中 `route` 遵循 rou3 约定的那句）
- https://nuxt.com/docs/4.x/guide/concepts/rendering（Route Rules 九属性逐条原文、八条规则示例及其注释、"Routes using `isr` or `swr` also generate `_payload.json`" 的 `::note`）
- https://nuxt.com/docs/4.x/api/utils/define-route-rules（shorthand 定位句、`experimental.inlineRouteRules` 前提、被翻译成顶层 `routeRules: { '/': { prerender: true } }` 的示例）
- https://nuxt.com/docs/4.x/getting-started/deployment 与 https://nuxt.com/docs/4.x/getting-started/prerendering（`'/**': { isr: true }` 环境覆盖示例、`'/200.html': { prerender: true }` 兜底页示例）
