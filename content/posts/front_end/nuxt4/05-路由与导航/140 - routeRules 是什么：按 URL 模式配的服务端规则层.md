---
url: /posts/1efdd70a64645a7b447f941f1664c657/
title: 140 - routeRules 是什么：按 URL 模式配的服务端规则层
date: 2026-11-27T11:00:00+08:00
updated: 2026-11-27T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`routeRules` 在配置参考里的定义只有一句：**"应用于匹配到的服务端路由的全局路由选项。"**，并紧跟两条标注：**"**实验性：这是一个实验特性，API 未来可能变化。**"以及一个指向 **Nitro route rules 文档**的"另见"。它解决的问题在混合渲染那一节说得最清楚：**"混合渲染允许**按路由**使用不同的缓存规则，并决定服务器在给定 URL 上新请求应如何响应。"**官方补的历史背景是："**以前，一个 Nuxt 应用和服务器的每个路由/页面都必须使用同一种渲染模式，通用渲染或客户端渲染。在各种情况下，某些页面可以在构建时生成，而另一些应当客户端渲染。例如，想想一个带后台管理区的网站：每个内容页应当主要是静态的、被生成一次，而后台管理区需要注册、行为更像动态应用。**"官方对实现机制的说明是一句常被忽略的话：**"Nuxt 服务器会自动注册对应的**中间件**，并使用 Nitro 缓存层用缓存处理器包裹路由。"**——**注意这里的"中间件"在服务端那一侧，与 133 的路由中间件不是一回事。**官方给的整段示例就是这一层的用法样本：`'/': { prerender: true }`（首页在构建时预渲染）、`'/products': { swr: true }`、`'/products/**': { swr: 3600 }`（按需生成、后台再验证、缓存 1 小时）、`'/blog': { isr: 3600 }`、`'/blog/**': { isr: true }`（一次生成直到下次部署、缓存在 CDN）、`'/admin/**': { ssr: false }`（后台只在客户端渲染）、`'/api/**': { cors: true }`（给 API 路由加 CORS 头）、`'/old-page': { redirect: '/new-page' }`（重定向旧 URL）。除了写在 `nuxt.config`，还有两个入口：**页面级的 `defineRouteRules`**（官方定义为"在页面级为混合渲染定义路由规则"，但它同样是实验特性——"**该特性是实验性的，为了使用它你必须在 `nuxt.config` 里启用 `experimental.inlineRouteRules` 选项**"，示例 `defineRouteRules({ prerender: true })` 会被翻译成 `routeRules: { '/': { prerender: true } }`，并附三条映射规则）和**模块侧的 `extendRouteRules`**（Kit，签名 `extendRouteRules(route: string, rule: NitroRouteConfig, options?: ExtendRouteRulesOptions)`，options 里 `override` 默认 `false` 用于"覆盖路由规则配置"）。还有一条全局限制必须知道：**混合渲染在使用 `nuxt generate` 时不可用。**

categories:
  - 前端开发

tags:
  - nuxt4
  - routerules
  - 混合渲染
  - 缓存
  - 服务端

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


`ssr: false` 是整站一个开关。但真实项目里通常是：**营销页要静态、后台要像应用、博客要缓存一小时、旧链接要 301**。这些按 URL 分别配置的东西，就是 `routeRules`。

## 官方给它的定义，只有两句

| 出处 | 原文 |
| --- | --- |
| 配置参考 | "**应用于匹配到的服务端路由的全局路由选项。**" |
| 配置参考 | "**实验性：这是一个实验特性，API 未来可能变化。**" + 另见 Nitro route rules 文档 |

**两条标注都要读**：它按"**匹配到的服务端路由**"生效（不是按页面组件），而且 API 自己声明可能变——所以升级 Nuxt 大版本时，这一块的配置要重新核。

混合渲染那一节补上了它的目标：

> **混合渲染允许按路由使用不同的缓存规则（使用 Route Rules），并决定服务器在给定 URL 上新请求应如何响应。**
>
> 以前，一个 Nuxt 应用和服务器的每个路由/页面都必须使用同一种渲染模式，通用渲染或客户端渲染。……例如，想想一个带后台管理区的网站：每个内容页应当主要是静态的、被生成一次，而后台管理区需要注册、行为更像动态应用。

## 官方那段示例就是用法样本

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    // Homepage pre-rendered at build time
    '/': { prerender: true },
    // Products page generated on demand, revalidates in background, cached until API response changes
    '/products': { swr: true },
    // Product pages generated on demand, revalidates in background, cached for 1 hour (3600 seconds)
    '/products/**': { swr: 3600 },
    // Blog posts page generated on demand, revalidates in background, cached on CDN for 1 hour (3600 seconds)
    '/blog': { isr: 3600 },
    // Blog post page generated on demand once until next deployment, cached on CDN
    '/blog/**': { isr: true },
    // Admin dashboard renders only on client-side
    '/admin/**': { ssr: false },
    // Add cors headers on API routes
    '/api/**': { cors: true },
    // Redirects legacy urls
    '/old-page': { redirect: '/new-page' },
  },
})
```

读这张示例要抓到三件事：

- **键是 URL 模式，不是路由名**：`/products/**` 这种 glob 写法。
- **一个键可以覆盖一堆路径**：`'/admin/**': { ssr: false }` 只关掉那一块的 SSR，整站其它部分照常通用渲染。
- **`/api/**` 也在这一层**：`cors: true` 给接口加 CORS 头——这与 133 那条"路由中间件管不到 `/api/*`"是同一件事的另一面：**要按路径配服务端行为，就来这里。**

## 机制：它会注册服务端中间件

官方这一句常被跳过，但它决定了很多现象：

> Nuxt 服务器会**自动注册对应的中间件**，并使用 [Nitro 缓存层](https://nitro.build/guide/cache) 用缓存处理器包裹路由。

也就是说：**`routeRules` 不是"提示"，而是被翻译成服务端中间件与缓存包装。**这带来两个实际后果：

1. 它作用在**进入服务端的那次请求**上，与客户端导航无关（133 的那条链）。
2. 官方另外还写了：**"只要可能，route rules 会被自动应用到部署平台原生的规则上以获得最佳性能（目前支持 Netlify 与 Vercel）。"**——同一段配置在支持的平台上可能由平台侧承担。

## 三个入口，按"什么时候决定"排

| 入口 | 写在哪 | 官方定位 | 是否实验 |
| --- | --- | --- | --- |
| `routeRules` | `nuxt.config` | 应用于匹配到的服务端路由的全局路由选项 | **是**（API 可能变化） |
| `defineRouteRules` | **页面文件里** | "在页面级为混合渲染定义路由规则。" | **是**（需 `experimental.inlineRouteRules`） |
| `extendRouteRules` | **模块里**（`@nuxt/kit`） | "扩展路由规则" | Kit 工具 |

### 页面级：`defineRouteRules`

官方的示例与"翻译结果"：

```vue [app/pages/index.vue]
<script setup lang="ts">
defineRouteRules({
  prerender: true,
})
</script>

<template>
  <h1>Hello world!</h1>
</template>
```

> 将被翻译成：`routeRules: { '/': { prerender: true } }`

并注一条执行侧的说明："**在运行 `nuxt build` 时，首页会被预渲染到 `.output/public/index.html` 并被静态提供服务。**"

官方给了三条映射规则（决定"这条页面规则最终管哪些路径"）：

> - 在 `~/pages/foo/bar.vue` 里定义的规则会应用到 `/foo/bar` 请求。
> - 在 `~/pages/foo/[id].vue` 里的规则会应用到 `/foo/*` 请求。
> - 在一个只有有限可选集合的页面里（例如自定义 `path` 为 `/:locale(en|fr)/about`）的规则，会为每个可选项生成一条规则（`/en/about` 与 `/fr/about`）。

以及一条会**在构建期警告**的边界：

> 如果一个页面路径无法被转换成等价的 route rule 模式（例如带正则的参数 `/:id(\d+)`、部分段 `/prefix-:id`、或可重复参数 `/:slug+`），**该页面的规则不会被应用，Nuxt 会在构建期间发出警告。**这种情况下，请把规则显式定义在 `nuxt.config` 里的 **`nitro.routeRules`**。

官方最后再补一句建议：**"想要更多控制——例如你在页面里用了自定义的 `path` 或 `alias`——你应当直接在 `nuxt.config` 里设置 `routeRules`。"**（回想 113：`path` 正是能写出 `/:postId(\d+)-:postSlug` 那种模式的地方——**两者天然冲突**，一个用了自定义正则的路由，页面级规则就落不下去。）

### 模块侧：`extendRouteRules`

签名 `function extendRouteRules (route: string, rule: NitroRouteConfig, options?: ExtendRouteRulesOptions): void`，官方对参数的说明是"**route**：要匹配的路径模式；**rule**：应用到被匹配路由上的路由规则配置"，options 只有一格：**`override`**（`boolean`、默认 `false`、"**覆盖路由规则配置，默认为 false**"）。官方的用法示例：

```ts
extendRouteRules('/preview', {
  redirect: {
    to: '/preview-new',
    statusCode: 302,
  },
})

extendRouteRules('/preview-new', {
  cache: {
    maxAge: 60 * 60 * 24 * 7,
  },
})
```

**注意它里面能写 `redirect: { to, statusCode }` 与 `cache: { maxAge }`**——那是 Nitro 层的配置形状，与 141 讲的那九个属性是同族但不完全同名。

## 一条全局限制

官方用 important 收口：

> **注意，在使用 `nuxt generate` 时混合渲染不可用。**

这条与 103/111 说过的"纯静态站点没有服务器"是同一逻辑：**`generate` 的产物里没有能按请求决策的进程**，所以按 URL 配服务端行为这件事不成立。要混合渲染就用 `nuxt build`（可带 `--prerender`）。

## 该在哪一层配：三句话判据

| 你的需求 | 该去哪 |
| --- | --- |
| 让某些路径不 SSR / 加 CORS 头 / 缓存 / 重定向 | `routeRules`（或支持的平台原生规则） |
| 规则天然属于这一页、且路径能一一对应 | `defineRouteRules`（先开 `experimental.inlineRouteRules`） |
| 用了自定义 `path`/`alias`、或含正则/部分段/可重复参数 | **`nuxt.config` 里显式写**（页面级规则会不生效并警告） |
| 模块要随包附带规则 | `extendRouteRules`（同名要不要替换看 `override`） |
| 想保护接口数据本身 | 服务端那道门（133），不是 route rules |

## 自查

1. `routeRules` 在配置参考里的定义是什么？它带了哪两条标注？
2. 混合渲染解决了以前什么限制？官方举的示例场景是什么？
3. 官方说 Nuxt 服务器会为 route rules 自动做什么？这为什么与客户端导航无关？
4. `defineRouteRules` 需要哪个实验开关？`~/pages/foo/[id].vue` 里的规则会应用到哪些请求？
5. 哪三类页面路径会让规则"落不下去"并在构建期警告？此时官方让把规则写在哪？
6. 为什么 `nuxt generate` 下混合渲染不可用？
7. 模块里给某条路径配重定向与缓存，用哪个 Kit 函数、怎么写？

## 下一篇

141 把那九个属性逐个拆开：`swr` 与 `isr` 的行为差别与 TTL 语义、`cors` 与 `headers` 的关系、`noScripts` 的三种取值与"整文档加载"后果、`appMiddleware` 的作用域限制，以及 `isr`/`swr` 路由会额外产出 `_payload.json`。
