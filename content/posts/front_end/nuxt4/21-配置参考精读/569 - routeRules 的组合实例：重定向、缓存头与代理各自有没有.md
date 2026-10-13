---
url: /posts/ab3cec563a05a364b061e29c2d39da81/
title: 569 - routeRules 的组合实例：重定向、缓存头与代理各自有没有
date: 2027-04-19T11:00:00+08:00
updated: 2027-04-19T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：路由规则的清单有九个属性，但真正写配置时是拿需求去对号：老链接重定向、接口结果缓存、把某段路径转给后端服务、让部分页面不带 JavaScript。这一篇按这四个需求逐条核对，代码只取官方文档里出现过的写法。**能对上的是三类，对不上的是代理**：九属性里没有 `proxy`，`routeRules` 那一节也没有；文档里唯一一次提到 "routeRules proxy" 是在讲实验性类型开关时顺口说的。另外两处要注意：`redirect` 在官方文档里有**字符串和对象两种形状**，`cache: { maxAge }` **在示例里被当成 routeRules 属性用，却不在九属性清单里**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 路由规则
  - 缓存策略
  - 重定向
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

清单背下来了，写的时候还是会卡住：产品要的四件事——老链接跳新地址、接口结果缓存一小时、`/api` 转给后端服务、博客页不去掉 JS 就白屏——分别是哪个属性？

一件件对。

## 重定向：官方文档里同时存在两种写法

渲染概念页那条示例是**字符串**：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    // Redirects legacy urls
    '/old-page': { redirect: '/new-page' },
  },
})
```

同一条规则在服务端引擎页写成了**带状态码的对象**：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    // Redirection to avoid 404
    '/old-page': {
      redirect: { to: '/new-page', statusCode: 302 },
    },
  },
})
```

九属性清单给的是 `redirect: string`{lang="ts"}。**两处形状不一样，文档没有说明哪种是准的，也没有列出 `statusCode` 允许哪些值**（示例里只出现过 302）。要 301 的场合，先按对象写、测一遍响应状态码。

关于 `redirect` 还有两句别处给出的：

- 服务端引擎页：**"Some route rules (`appMiddleware`, `redirect` and `prerender`) also affect client-side behavior."** —— 它不只管服务端。
- 实验特性页的流式渲染警告里，设了 `redirect` 的请求**不走流式**，会 "short-circuit to a redirect or error response"。

## 缓存：swr 与 isr 是清单里的两个，但示例里还有第三个

这两个属性都在九属性清单内，且官方明写行为相同，差别只在缓存放哪：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    // Products page generated on demand, revalidates in background, cached until API response changes
    '/products': { swr: true },
    // Product pages generated on demand, revalidates in background, cached for 1 hour (3600 seconds)
    '/products/**': { swr: 3600 },
    // Blog posts page generated on demand, revalidates in background, cached on CDN for 1 hour (3600 seconds)
    '/blog': { isr: 3600 },
    // Blog post page generated on demand once until next deployment, cached on CDN
    '/blog/**': { isr: true },
  },
})
```

清单里那两句说明拆开来是这样的：

- `swr: number | boolean`——加缓存头并把响应缓存在**服务器或反向代理**，TTL 可配；`node-server` 预设能缓存完整响应；TTL 到期后先返回缓存的响应，同时后台重新生成；传 `true` 时加一个 `stale-while-revalidate` 头，**没有 MaxAge**。
- `isr: number | boolean`——行为同上，差别是**在支持的平台上能把响应放进 CDN 缓存**，官方当前点名的是 Netlify 和 Vercel；传 `true` 时内容**保留到下次部署**。

第三条是这一篇要单独提醒的：九属性清单里**没有 `cache`**，但服务端引擎页的示例这么写了，注释还写着 "Cached for 1 hour"：

```ts [nuxt.config.ts]
'/api/*': { cache: { maxAge: 60 * 60 } },
```

**两处都在用，清单里没有。** 同一页还把它当成既有能力描述了一整句：一个覆盖该路径的 `cache` 路由规则，会在 GET/HEAD 未命中时把它的 `maxAge` 作为 `Cache-Control` 发出去，让 CDN 能吃掉对未知路径的重复请求；同一页另一处警告也把 `cache` 与 `isr`、`swr`、`redirect`、`noScripts`、`streaming: false` 并列为 "routeRules setting"。**所以 `cache: { maxAge }` 不是示例里的笔误，但它确实不在渲染概念页那份清单里**——这条属性算不算正式 API，文档没有裁决，用之前先实测。只求稳的话用清单里的 `swr`。

另外记一笔 `::note`：用了 `isr` 或 `swr` 的路由会**在 HTML 旁边生成 `_payload.json`**，客户端导航时加载这些缓存好的 payload，而不是重新取数据。

## 响应头：cors 与 headers 是一对

清单里 `cors: boolean` 与 `headers: object` 挨着写，关系也写在同一行里：

> Automatically adds cors headers with `cors: true` - you can customize the output by overriding with `headers`

也就是说 `cors: true` 给你自动那套，想改具体值就用 `headers` 覆盖。`headers` 自己的说明是 "Add specific headers to sections of your site - for example, your assets"。

**官方那份八条规则的示例里没有任何 `headers` 条目**，具体头名怎么写文档没给样例——它是 h3/Nitro 那侧的响应头对象，写法要按实测或去看 Nitro 的 route rules 文档。

## 代理：这个需求在清单里找不到对应属性

要把 `/api/**` 转给另一个后端服务，最直觉的做法是找 `proxy`。结果是这样：

- 九属性清单里没有 `proxy`。
- 配置参考的 `## routeRules` 那一节只有一句 "Global route options applied to matching server routes"，没有属性清单。
- 整套 4.x 文档里 `proxy` 和 `routeRules` 同时出现只有一次，而且是在讲另一个功能时顺口提的：实验特性页说 Nuxt 看不见所有请求的应答方，可能是 nitro 中间件、**"a `routeRules` proxy"**、或 catch-all handler。

这句是既成事实式的提及，**没有键名、没有形状、没有示例**。配置页那句 `See` 指向 Nitro 的 route rules 文档——代理的写法要到那里查，Nuxt 4.x 文档本身给不出可照抄的代理配置。

## 静态文件服务：prerender 加 noScripts

"这一批页面完全不要 JavaScript" 是第四个需求，官方在静态站配方页给的组合是两条属性并写：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    '/blog/**': { prerender: true, noScripts: true },
    '/about': { prerender: true, noScripts: true },
  },
})
```

`noScripts` 命中时 HTML 里会省掉四样：Nuxt 入口 `<script>` 标签、入口 chunk 的 import map、内联 payload `<script>`（以及启用 payload 提取时的 payload preload link）、JS resource hints（JS chunk 的 `preload` 与 `prefetch`）。**CSS 不动**，样式链接和内联样式照旧渲染，所以页面看起来一样。

那条 `::important` 是这个组合最容易被忽略的代价：`noScripts` 的页面**不水合**，`@click` 全部失效，`<NuxtLink>` 不再预取（链接退化成普通 `<a>`），离开这一页也是整页加载。而且进出这类页面都会触发整文档加载。

还有个只出现在生产构建的坑。`features.noScripts` 默认值是 `'production'`，即**开发环境照常发 JavaScript**，于是依赖客户端 JS 的东西（懒水合策略、server component 里的 `nuxt-client` 组件）dev 里看着正常，上线后静默失效——官方为这条留了错误码 E8007。Resolution 给两条：把客户端依赖从受影响路由去掉，或者改用 `routeRules` 里的 `noScripts`，因为**路由规则在开发期同样生效，问题当场就能看见**；要么把 `features.noScripts` 设成 `'all'`，让 dev 与生产行为一致。

## 清单外还有一个：appLayout

布局页里有一条不在九属性里的规则，标了 `v4.3`：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    // Set layout for specific route
    '/admin': { appLayout: 'admin' },
    // Set layout for multiple routes
    '/dashboard/**': { appLayout: 'dashboard' },
    // Disable layout for a route
    '/landing': { appLayout: false },
  },
})
```

三种用法都在：按精确路径指定、按前缀批量指定、传 `false` 关掉布局。官方给的适用理由是"想在配置里集中管布局，或者要把布局套到**没有对应页面组件的路由**上（比如能匹配很多路径的 catchall 页）"。

顺带一提，`appMiddleware` 也在这条线上——但它管的是 Vue 应用侧中间件，括号里明写 "that is, not your Nitro routes"。

## 组合层面有三条硬边界

- **`nuxt generate` 时混合渲染不可用**（渲染概念页 `::important` 原句：Hybrid Rendering is not available when using `nuxt generate`）。
- 平台原生规则优先：官方说 whenever possible，路由规则会被自动翻译成部署平台的原生规则，**当前点名支持的是 Netlify 和 Vercel**。
- `ssr: false` 会影响服务端包体积，但只在建时能确定规则才生效：如果同一区域下有更具体的规则把 SSR 打开（`'/admin/**': { ssr: false }` 配 `'/admin/report': { ssr: true }`）、页面有落在客户端渲染区域之外的 alias 或绝对路径子路由、或者它还是某个服务端渲染子页面的父壳，**这个页面会留在服务端包里**。官方补了一句：这是构建期优化，不改变服务端发给浏览器的内容。

## 自查

拿你手上那份需求对着过一遍：

- 重定向：**两种形状文档里都有**，`redirect: string` 是清单写的，`{ to, statusCode }` 是服务端页示例用的；statusCode 取值范围没写。
- 缓存：`swr` / `isr` 在清单里，`cache: { maxAge }` **不在清单里但被示例使用**；两者选一，选 `cache` 要先实测。
- 响应头：`cors: true` 出自动那套，`headers` 覆盖定制；官方没有 `headers` 的示例。
- 代理：**四个需求里唯一没有对应属性的**，4.x 文档查不到写法，只有配置页那句指向 Nitro 的 `See`。
- 静态：`prerender` + `noScripts` 成立，代价是整页不水合；E8007 提醒 `features.noScripts` 默认只在生产剥离。
- 别忘 `appLayout`（v4.3）不在九属性清单里。

## 下一篇

下一篇 570 换 `runtimeConfig`：这一节给了五行说明、一个 Type 和一份带四个 `app` 键的默认值 JSON——**默认值里那四个键是框架自己填的，你能写的是它们之外的部分**，而"只有服务端读得到"这句的准确范围要看 `public` 和 `app` 两个键名。

## 本文依据

- https://nuxt.com/docs/4.x/guide/concepts/rendering（九属性逐条原文、八条规则示例、`_payload.json` 的 `::note`、`ssr: false` 与服务端包体积三种保留条件、平台原生规则那句、`::important` 关于 `nuxt generate`）
- https://nuxt.com/docs/4.x/getting-started/server（`cache: { maxAge: 60 * 60 }` 与 `redirect: { to, statusCode: 302 }` 示例、"Some route rules also affect client-side behavior" 那句）
- https://nuxt.com/docs/4.x/guide/recipes/mostly-static-sites（`prerender` + `noScripts` 组合示例、省略项清单、不水合的 `::important`、进出均为整页加载）
- https://nuxt.com/docs/4.x/guide/going-further/features（`noScripts` 省略的四样东西、CSS 不受影响、按路由禁用那句）
- https://nuxt.com/docs/4.x/errors/e8007（dev 正常/生产静默失效的成因、两条 Resolution）
- https://nuxt.com/docs/4.x/guide/going-further/experimental-features（`routeRules` proxy 那句、流式渲染回退清单里的 `cache` 与 `streaming: false`）
- https://nuxt.com/docs/4.x/directory-structure/app/layouts（`appLayout` 三条规则与 tip，标 v4.3）
- https://nuxt.com/docs/4.x/api/nuxt-config（`## routeRules` 三行与指向 Nitro 的 See）
