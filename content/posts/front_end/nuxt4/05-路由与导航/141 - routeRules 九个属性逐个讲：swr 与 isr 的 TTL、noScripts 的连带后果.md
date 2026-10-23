---
url: /posts/30fcb9bdb1f15ee3b3a9ebd39db9c7b5/
title: 141 - routeRules 九个属性逐个讲：swr 与 isr 的 TTL、noScripts 的连带后果
date: 2026-11-28T09:00:00+08:00
updated: 2026-11-28T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇逐个拆官方列出的九个属性（"你可以使用的不同属性如下"）：**`redirect: string` —— 定义服务端重定向**；**`ssr: boolean` —— 用 `ssr: false` 关闭你应用某些板块的 HTML 服务端渲染、让它们只在浏览器里渲染**；**`cors: boolean` —— 用 `cors: true` 自动添加 CORS 头，可以通过 `headers` 覆盖来定制输出**；**`headers: object` —— 为你站点的某些板块（例如你的静态资源）添加特定头**；**`swr: number | boolean` —— 给服务端响应添加缓存头并在服务器或反向代理上按可配置的 TTL（存活时间）缓存它；Nitro 的 `node-server` preset 能够缓存完整响应；当 TTL 过期时，缓存的响应会被发送、同时页面在后台被重新生成；如果使用 `true`，会添加一个 `stale-while-revalidate` 头而不带 MaxAge**；**`isr: number | boolean` —— 行为与 `swr` 相同，除了我们能够在支持此能力的平台（目前是 Netlify 或 Vercel）上把响应加入 CDN 缓存；如果使用 `true`，内容在 CDN 里会一直保留到下次部署**；**`prerender: boolean` —— 在构建时预渲染路由，并把它们作为静态资源包含进你的构建**；**`noScripts: boolean` —— 为你站点的某些板块关闭 Nuxt 脚本与 JS 资源提示的渲染**；**`appMiddleware: string | string[] | Record<string, boolean>` —— 允许你定义在你应用的 **Vue 部分**里，应当或不应当为页面路径运行的中间件**。两条连带事实：官方 note 写明 **"使用 `isr` 或 `swr` 的路由也会在 HTML 旁边生成 `_payload.json` 文件。客户端导航会加载这些被缓存的 payload，而不是重新取数。"**；而混合渲染一节还有一条 **"注意混合渲染在使用 `nuxt generate` 时不可用。"** `noScripts` 那一格在特性页里有完整展开（取值 `false`（默认）/ `'production'`（或 `true`，会被规范化为 `'production'`）/ `'all'`；被省略的四类东西；"CSS 不受影响……页面只是永不水合，这意味着上面没有任何东西是可交互的"；客户端导航到被该规则覆盖的路由会触发**整文档加载**；构建期已知被覆盖的页面会**完全从客户端包里排除**；这类路由**总是用缓冲（非流式）渲染器渲染**；只渲染静态 HTML 的服务端组件没问题，但 `nuxt-client` 水合的组件与交互式岛屿插槽依赖内联脚本搬运 teleport 内容、因此在 noScripts 路由上不会变得可交互）。

categories:
  - 前端开发

tags:
  - nuxt4
  - routerules
  - 缓存
  - noscripts
  - payload

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


上一节说清了 `routeRules` 是哪一层。这一节把九个属性逐个拆开——**尤其 `swr` 与 `isr` 的差别，以及 `noScripts` 那一串连带后果**，这两处最容易在上线后才发现问题。

## 九格先按用途分组

| 组 | 属性 |
| --- | --- |
| 决定"怎么渲染" | `ssr`、`prerender`、`noScripts` |
| 决定"怎么缓存" | `swr`、`isr` |
| 决定"响应长什么样" | `redirect`、`headers`、`cors` |
| 决定"Vue 侧跑什么" | `appMiddleware` |

## 缓存两兄弟：`swr` 与 `isr`

官方的两条定义并排读：

| 属性 | 官方说明（要点） |
| --- | --- |
| `swr: number \| boolean` | 给服务端响应**添加缓存头**，并在**服务器或反向代理**上按**可配置的 TTL（存活时间）**缓存它；**Nitro 的 `node-server` preset 能够缓存完整响应**；**当 TTL 过期时，缓存的响应会被发送、同时页面在后台被重新生成**；如果使用 `true`，会添加一个 `stale-while-revalidate` 头**而不带 MaxAge** |
| `isr: number \| boolean` | **行为与 `swr` 相同**，除了**我们能够在支持此能力的平台（目前是 Netlify 或 Vercel）上把响应加入 CDN 缓存**；如果使用 `true`，**内容在 CDN 里会一直保留到下次部署** |

从这两条能读出三个决策点：

1. **`swr` 的落点在服务器/反代，`isr` 才谈 CDN**，而 CDN 那一层**依赖平台**（官方点名 Netlify 或 Vercel）。**部署在不支持的平台时用 `isr`，得到的效果就是 `swr`。**
2. **数字就是 TTL 秒数**：官方示例注释写明 `'/products/**': { swr: 3600 }` 是"缓存 1 小时（3600 秒）"、`'/blog': { isr: 3600 }` 同。
3. **`true` 是另一种语义**：`swr: true` = 加 `stale-while-revalidate` 头但**没有 MaxAge**；`isr: true` = **保留到下次部署**。官方对 `/products` 与 `/blog/**` 的注释分别是"cached until API response changes"和"generated on demand once until next deployment"。

还有一条对"过期"的直觉要修正：TTL 到期**不是**让请求等重算——**"当 TTL 过期时，缓存的响应会被发送、同时页面在后台被重新生成。"**

## `_payload.json`：isr/swr 路由的第二份产物

官方 note：

> 使用 `isr` 或 `swr` 的路由也会在 HTML 旁边生成 **`_payload.json`** 文件。**客户端导航会加载这些被缓存的 payload，而不是重新取数。**

这来自取数结果的序列化机制。官方对 payload extraction 的完整说明值得连着读：

> 当 Nuxt 在服务端渲染一个页面时，它会把你的数据取用（`useAsyncData` 与 `useFetch`）与应用状态（`useState`）的结果序列化进一个 payload，好让客户端无需重新取数就能水合。**启用 payload extraction 后，Nuxt 还会把这份 payload 写入该路由 HTML 旁边的 `_payload.json` 文件**：预渲染的路由在构建期生成 payload 文件；**使用 ISR 或 SWR 缓存的路由在该路由第一次被渲染时生成 payload 文件，即使是在混合（非静态）站点上。**
>
> **在客户端导航期间，Nuxt 会取回目标路由的 `_payload.json` 文件并复用被抽取的数据，而不是在浏览器里再跑一遍数据取用。**

开关是 `experimental.payloadExtraction`，三个取值与默认：

| 取值 | 官方说明 |
| --- | --- |
| `'client'` | payload 内联在 HTML 里用于首次渲染，并为客户端导航抽取到 `_payload.json` 文件。**首次加载没有额外网络请求。** |
| `true` | 首次渲染与客户端导航**都**抽取到单独的 `_payload.json`。**HTML 更小、payload 文件可被 CDN 缓存，代价是首次加载多一次请求。** |
| `false` | 关闭抽取。payload 总是内联在 HTML 里，**不生成 `_payload.json`**。 |

官方给的默认值：**"默认是 `true`，或在设置 `compatibilityVersion: 5` 时是 `'client'`。当设置了 `ssr: false` 时它被强制为 `false`。"**

三条官方点名的实际后果：

- 在**完全静态**的站点上，客户端导航复用的是**构建时**捕获的数据，所以**数据可能一直过期到下次重新构建**。
- 对 ISR/SWR 路由，**CDN 可以把 payload 文件与 HTML 一起缓存**，改善被缓存路由的客户端导航性能；像 `pages/[...slug].vue` 这样的动态路由可以用 `'/**': { isr: true }` 这类 glob 模式加入。
- payload 用 devalue 序列化，所以**自定义类型（例如类实例）需要带自定义 reducer 与 reviver 的 payload 插件才能撑过这次往返**。

## `ssr`、`prerender` 与"服务端包体积"

`ssr` 的官方定义：**"用 `ssr: false` 关闭你应用某些板块的 HTML 服务端渲染，让它们只在浏览器里渲染。"**

这一格有一条与包体积相关的连带说明（"Server Bundle Size with `ssr: false`"）：

> 被 `ssr: false` 覆盖的路由只在浏览器里渲染，**所以 Nuxt 会把它的页面组件从服务端包里排除**。当覆盖到通往该页面的每一条路径的规则**都能在构建期被解析**时，这同样适用——包括位于 `/products/**` 规则之下的 `pages/products/[id].vue` 这样的动态路由。
>
> **当通往该页面的任一路径仍可能在服务端被渲染时，这个页面会保留在服务端包里：**
> - 某个更具体的规则在 client-only 区域之下重新启用了 SSR（`'/admin/**': { ssr: false }` 配上 `'/admin/report': { ssr: true }`）
> - 该页面有一个 alias、或一个用绝对路径声明的子路由，落在 client-only 区域之外
> - 该页面是一个渲染了仍然被服务端渲染的子页面的**父壳**
>
> **这只是一项构建期优化；它不改变服务器发给浏览器的内容。**

**这三条例外正是"我明明关了 SSR，包却没变小"的标准答案**——尤其第三条：一个只是提供外壳的父页面，因为子页面还在服务端渲染，它自己也留在服务端包里。

`prerender` 的定义是：**"在构建时预渲染路由，并把它们作为静态资源包含进你的构建。"**（与 111 讲的 `200.html`/`404.html` 由 `generate`/`--prerender` 写出相呼应。）

## 响应形状三格：`redirect`、`headers`、`cors`

| 属性 | 官方说明 |
| --- | --- |
| `redirect: string` | 定义**服务端**重定向 |
| `headers: object` | 为你站点的某些板块（例如你的静态资源）添加特定头 |
| `cors: boolean` | 用 `cors: true` **自动添加 CORS 头**，**可以通过 `headers` 覆盖来定制输出** |

`cors` 那句里的"用 `headers` 覆盖来定制"说明两者是**同一层响应头的两个来源**——`cors: true` 是快捷方式，精细控制走 `headers`。

`redirect` 也要与 135 那个 `definePageMeta({ redirect })` 区分：这一格是**服务端在响应阶段**改道（"定义服务端重定向"，官方示例注释是"Redirects legacy urls"——旧 URL 搬家用它），而记录属性那条是**客户端路由匹配到之后**的、发生在任何导航守卫之前的重定向（113、136）。

## `appMiddleware`：Vue 侧的那一格

官方定义：**"允许你定义在你应用的 Vue 部分里，应当或不应当为页面路径运行的中间件（也就是说，不是你的 Nitro 路由）。"**

类型 `string | string[] | Record<string, boolean>`——**第三种形态用键值来"关"某个中间件**（`Record<string, boolean>` 的布尔值正是"应当或不应当"）。

这一格与 133 那条"路由中间件不为 `/api/*` 运行"是同一分界线的两侧：**它管的是页面导航链上的中间件，管不到 Nitro 路由。**

## `noScripts`：最容易被低估的一格

route rules 里它只有一句话：**"为你站点的某些板块关闭 Nuxt 脚本与 JS 资源提示的渲染。"**，并指向特性页。特性页的展开决定了一整串后果：

**定位**："**关掉 Nuxt 脚本与 JavaScript 资源提示的渲染，让页面以纯 HTML 与 CSS 交付。**"

| 取值（`features.noScripts`） | 官方说明 |
| --- | --- |
| `false`（默认） | 脚本正常渲染 |
| `'production'`（或 `true`，会被**规范化为 `'production'`**） | **只在生产构建里省略脚本，所以在 `nuxt dev` 里 HMR 等开发特性照常工作** |
| `'all'` | 开发与生产**都**省略脚本 |

按路由关闭则用 route rule（**"你也可以用 `noScripts` 路由规则按路由关闭脚本，它在所有模式下都适用"**）：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    '/blog/**': { noScripts: true },
  },
})
```

当 `noScripts` 作用于一个被渲染的页面时，官方列出**从 HTML 里被省略的四类东西**：

- Nuxt 入口 `<script>` 标签
- 入口 chunk 的 import map
- 内联的 payload `<script>`（以及当 payload 抽取启用时的 payload 预加载 link）
- JavaScript 资源提示（针对 JS chunk 的 `preload` 与 `prefetch` link）

**并给出一条会被误解的结论**："**CSS 不受影响：样式表链接与内联样式仍然会被渲染，所以页面看起来完全一样。页面只是永不水合，这意味着上面没有任何东西是可交互的。**"

其余四条官方点名的行为：

| 行为 | 官方说明 |
| --- | --- |
| 导航方式变了 | "客户端导航到被 `noScripts` 路由规则覆盖的路由时，会改为触发一次**整文档加载**，这样该路由实际就是被不带脚本地提供服务，而不是由客户端路由器渲染。" |
| 包体积 | "**在构建期已知被这类规则覆盖的页面，也会被完全从客户端包里排除。**" |
| 预取补偿 | Nuxt 会在这些页面上发出一个 **speculation rules** 标签（是声明式 JSON、**不执行任何 JavaScript**）。规则**作用域限定在你的页面路由**（那些可以安全 `GET` 的），而不是每一个同源链接，所以**一个指向服务端路由（例如 `/logout`）的链接绝不会被预取**；但"**一个 catch-all 页面（例如 `pages/[...slug].vue`）会匹配每一个路径，这又把范围放宽回所有同源链接**" |
| 渲染与岛屿 | "带 `noScripts` 路由规则的路由**总是用缓冲（非流式）渲染器渲染**。"；"只渲染静态 HTML 的服务端组件工作正常，但**用 `nuxt-client` 水合的组件与交互式岛屿插槽依赖一段内联脚本来搬运 teleport 内容，因此它们在 noScripts 路由上不会变成可交互的**" |

第四条正好接上 106/107 那个 teleport 内联脚本的限制——**`noScripts` + 交互式岛屿 = 静默不可交互**。官方还给了配套读法：把 `noScripts` 与预渲染、岛屿、懒水合结合的"mostly-static"配方。

## 一张决策表

| 需求 | 属性 |
| --- | --- |
| 整块板块只要界面、不要服务端渲染 | `ssr: false` |
| 这一页构建时定死 | `prerender: true` |
| 服务器侧缓存 N 秒、过期后台重算 | `swr: N` |
| 能上 CDN 且平台支持 | `isr: N` / `isr: true`（到下次部署） |
| 旧 URL 搬家 | `redirect: '/new-page'` |
| 给静态资源区加头 / 自定义 CORS 输出 | `headers`（`cors: true` 是快捷方式） |
| 只要 HTML+CSS、永不水合 | `noScripts: true` |
| 某块路径不跑某个 Vue 侧中间件 | `appMiddleware` |

## 自查

1. `swr` 与 `isr` 的缓存落点分别在哪？`isr` 依赖什么？两者的 `true` 各是什么语义？
2. TTL 过期那一刻，用户拿到的是新内容还是旧内容？后台做什么？
3. 使用 `isr`/`swr` 的路由会在 HTML 旁边多产出什么文件？客户端导航为什么变快？
4. `payloadExtraction` 的三个取值与默认值分别是什么？`ssr: false` 时会发生什么？
5. 关了某块的 SSR，页面组件却还留在服务端包里，官方给的三条例外是什么？
6. `noScripts` 三种取值差别是什么？官方说"页面看起来完全一样"，那到底少了什么？
7. 为什么 `noScripts` 路由上的 `nuxt-client` 组件不会变可交互？这与 106 哪条限制同源？
8. `appMiddleware` 管得到 Nitro 路由吗？官方原话怎么说？

## 下一篇

142 讲文件式路由之外的三条路：`router.options.ts` 里的 `routes` 函数（以及它拿不到 `definePageMeta` 元数据这件事）、`pages:extend` 钩子的增删改，与模块侧的 `extendPages`。
