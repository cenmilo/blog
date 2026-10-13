---
url: /posts/65a5bda851b78b3a0804c3ddfa58f03d/
title: 138 - 滚动行为分层：scrollToTop、router 选项与 hashMode 那条限制
date: 2026-11-27T09:00:00+08:00
updated: 2026-11-27T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：换页时"滚不滚回顶部"这件事在 Nuxt 里分三层，混用会得到"我明明禁用了却还是跳顶"。第一层是页面级的 `scrollToTop`（`definePageMeta`）：类型 `boolean | ((to: RouteLocationNormalizedLoaded, from: RouteLocationNormalizedLoaded) => boolean)`，官方说明是"**告诉 Nuxt 在渲染页面之前是否滚动到顶部。导航独立于渲染，所以即使页面不重新渲染（例如使用固定的 `key` 时），滚动行为也总会触发。在这种情形下设 `scrollToTop: false` 来禁用滚动。如果你想覆盖 Nuxt 的默认滚动行为，可以在 `~/router.options.ts` 里做。**"——**这一段同时给出两个结论：固定 `key` 不等于不滚动；要改默认行为得换一层。**第二层是 `router.options.ts`（官方："**这是指定 router options 的推荐方式**"），它可以放函数形态的配置，比如自定义路由那条"**用一个接受被扫描到的路由、返回被定制的路由的函数**"来覆盖或扩展路由，以及自定义 history 那条"**用一个接受 base URL、返回 history 模式的函数来覆盖 history 模式。如果它返回 `null` 或 `undefined`，Nuxt 会回退到默认历史。**"。第三层是 `nuxt.config` 里的 `router.options`，官方明确限定"**只有可 JSON 序列化的选项可配置**"并列出七项：`linkActiveClass`、`linkExactActiveClass`、`end`、`sensitive`、`strict`、`hashMode`、`scrollBehaviorType`。其中两个各带一条重要行为：`scrollBehaviorType` 官方定义为"**自定义 hash 链接的滚动行为**"（类型 `string`、默认 `"auto"`），配成 `smooth` 后加载带 hash 链接的页面（例如 `https://example.com/blog/my-article#comments`）时浏览器会**平滑滚动到这个锚点**；`hashMode` 的限制最硬："**启用后，URL 从不发给服务器，并且不支持 SSR。**"另外两条官方注记也归这一层：在 `future.compatibilityVersion: 5` 下"**路由默认区分大小写以匹配 Nitro，把 `router.options.sensitive` 设为 `false` 可以退出这个行为**"；以及自定义路由那条关键差异——"**Nuxt 不会为你从 `routes` 函数返回的任何新路由增补在组件 `definePageMeta` 里定义的元数据。如果你希望这发生，应当使用在构建期被调用的 `pages:extend` 钩子。**"

categories:
  - 前端开发

tags:
  - nuxt4
  - 滚动
  - router
  - hashmode
  - 配置

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


"点进详情页怎么跳回顶部了"、"我想让它平滑滚到锚点"、"为什么我的 `router.options` 里传函数没反应"——这三问对应的是同一套东西的三层，别在最外层找内层的答案。

## 第一层：页面级的 `scrollToTop`

`definePageMeta` 里这个字段的完整官方说明，两句话都是结论：

> 告诉 Nuxt 在渲染页面之前是否滚动到顶部。**导航独立于渲染，所以即使页面不重新渲染（例如使用固定的 `key` 时），滚动行为也总会触发。**在这种情形下**设 `scrollToTop: false` 来禁用滚动**。如果你想覆盖 Nuxt 的默认滚动行为，你可以在 `~/router.options.ts` 里做。

- 类型：`boolean | ((to, from) => boolean)`——**也能按这次导航的目标与来源来判**。

| 你想要的 | 该写什么 |
| --- | --- |
| 这一页换参数时**别**跳顶 | `definePageMeta({ scrollToTop: false })`（**不是**固定 `key`，见 137） |
| 只有列表→详情才跳顶、返回时不跳 | 用函数形态按 `to`/`from` 判 |
| 改"默认行为"本身（不止这一页） | 去第二层：`router.options.ts` |

**这一层最容易犯的错，是以为"固定 `key` 让页面不重建"会顺带让滚动不动。**官方的原话正相反：滚动跟着**导航**，不跟着渲染。

## 第二层：`app/router.options.ts`（官方推荐的那层）

官方对它的定位：**"这是指定 `router options` 的推荐方式。"**

```ts [app/router.options.ts]
import type { RouterConfig } from '@nuxt/schema'

export default {
} satisfies RouterConfig
```

它比 `nuxt.config` 那一层强的地方是**能放函数**。官方给的两个函数形态示例：

**① 覆盖或扩展被扫描出来的路由**

> 使用 router options，你可以**可选择地**用一个**接受被扫描到的路由、并返回被定制的路由**的函数来覆盖或扩展你的路由。**如果它返回 `null` 或 `undefined`，Nuxt 会回退到默认路由（这对于修改输入数组很有用）。**

```ts [router.options.ts]
import type { RouterConfig } from '@nuxt/schema'

export default {
  routes: _routes => [
    {
      name: 'home',
      path: '/',
      component: () => import('~/pages/home.vue'),
    },
  ],
} satisfies RouterConfig
```

**这里有一条与 136/143 直接相关的警告**，官方写得非常直白：

> **Nuxt 不会为你从 `routes` 函数返回的任何新路由增补在你提供的组件的 `definePageMeta` 里定义的元数据。如果你希望这发生，应当使用在构建期被调用的 `pages:extend` 钩子。**

也就是说：**在这里"手工加"的路由拿不到 `definePageMeta` 的 `layout`、`middleware`、`validate`……** 要给新页面配元数据，必须走 `pages:extend`（它跑在构建期，见 142、143）。

**② 覆盖自定义 history**

> 你可以可选择地用一个**接受 base URL、返回 history 模式**的函数来覆盖 history 模式。**如果它返回 `null` 或 `undefined`，Nuxt 会回退到默认历史。**

```ts [router.options.ts]
import type { RouterConfig } from '@nuxt/schema'
import { createMemoryHistory } from 'vue-router'

export default {
  history: base => import.meta.client ? createMemoryHistory(base) : null, /* default */
} satisfies RouterConfig
```

那个 `null /* default */` 就是上面那句"回退到默认历史"的写法示范。

**多份 router options 文件**也可以：官方说"通过在 `pages:routerOptions` 钩子里加文件，可以添加更多 router options 文件。**数组里靠后的会覆盖靠前的。**"并且挂了一条会影响渲染模式的注记："**在该钩子里添加一个 router options 文件会开启基于页面的路由，除非设置了 `optional`，那种情况下它只会在基于页面的路由已经启用时才应用。**"

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  hooks: {
    'pages:routerOptions' ({ files }) {
      const resolver = createResolver(import.meta.url)
      // add a route
      files.push({
        path: resolver.resolve('./runtime/router-options'),
        optional: true,
      })
    },
  },
})
```

（"加 router options 文件会开启基于页面的路由"这条与 057 那句"pages 目录不存在时不引入 vue-router"是同一枚硬币的两面。）

## 第三层：`nuxt.config` 里的 `router.options`——只有七格

官方在这一层开头就加了 **Note**："**只有可 JSON 序列化的选项可配置**"，然后列出七项：

```ts [nuxt.config]
export default defineNuxtConfig({
  router: {
    options: {},
  },
})
```

| 可配项 | 官方说明 |
| --- | --- |
| `linkActiveClass` | 激活状态的链接类名 |
| `linkExactActiveClass` | 精确激活状态的链接类名 |
| `end` | 路径匹配是否必须完整 |
| `sensitive` | 路径匹配是否区分大小写 |
| `strict` | 是否允许结尾斜杠 |
| `hashMode` | 见下方那条硬限制 |
| `scrollBehaviorType` | 自定义 hash 链接的滚动行为 |

**这七格之外的一律去第二层**——这是"我传了个函数怎么没生效"的标准答案。

### `scrollBehaviorType`：只管 hash 链接

配置参考里它的定义是"**自定义 hash 链接的滚动行为**"，类型 `string`、**默认 `"auto"`**。自定义路由那一节给了唯一被点名的另一种取值：

> 你可以可选择地自定义 hash 链接的滚动行为。当你把配置设为 `smooth`，并且你加载一个带 hash 链接的页面（例如 `https://example.com/blog/my-article#comments`），你会看到浏览器**平滑滚动到这个锚点**。

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  router: {
    options: {
      scrollBehaviorType: 'smooth',
    },
  },
})
```

**它的范围要记住：这一格说的是 hash 链接（锚点），不是"换页时滚到顶部"。**后者属于第一、二层。

### `hashMode`：一条会连带 SSR 的限制

官方定义：

> 你可以在 SPA 模式下用 `hashMode` 配置启用 hash history。**在这种模式下，router 在实际内部传递的 URL 之前使用一个 `#` 字符。启用后，URL 从不发给服务器，并且不支持 SSR。**

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  ssr: false,
  router: {
    options: {
      hashMode: true,
    },
  },
})
```

这条限制有两层连带后果要一起接受：**服务器拿不到路径**（所以任何按路径做事的东西都失效——route rules、`appLayout`、服务端按 URL 取数全部无从判断），以及**没有 SSR**（103、104 那条"首屏来自服务端 HTML"的整条链断掉，白屏期交给 `spa-loading-template.html`）。

### `sensitive`：版本相关的一条默认值变化

官方注记：**"在 `future.compatibilityVersion: 5` 下，路由默认区分大小写以匹配 Nitro。把 `router.options.sensitive` 设为 `false` 可以退出这个行为。"**

## 一张分层图

```
换页时滚不滚顶？
├─ 这一页要例外吗？            → 第 1 层  definePageMeta({ scrollToTop: false })
│                                 （注意：与 key 是否固定无关）
├─ 默认行为本身要改？           → 第 2 层  app/router.options.ts（函数形态；官方推荐入口）
│                                 （routes 函数、history 函数都在这层，但拿不到 definePageMeta 元数据）
├─ hash 锚点要平滑滚动？        → 第 3 层  router.options.scrollBehaviorType: 'smooth'
│                                 （默认 'auto'，只管 hash 链接）
└─ 要不要 hashMode？            → 第 3 层，但先接受两条后果：URL 不发给服务器 + 不支持 SSR
```

## 自查

1. `scrollToTop` 的两种类型形态分别是什么？官方说它与渲染的关系是什么？
2. 固定 `key` 的页面，换参数时会不会跳顶？要禁用该改哪里？
3. 想覆盖 Nuxt 的默认滚动行为，官方指定在哪个文件里做？
4. `nuxt.config` 的 `router.options` 里可配的七项是哪些？为什么只有这七项？
5. `scrollBehaviorType` 的作用范围是什么？默认值？
6. 启用 `hashMode` 会连带哪两条限制？
7. 用 `routes` 函数新加的路由，拿得到组件里 `definePageMeta` 的元数据吗？官方给的替代是什么？
8. 在 `pages:routerOptions` 钩子里加 router options 文件，会对基于页面的路由产生什么影响？怎么避免？

## 下一篇

139 讲运行时换布局的 `setPageLayout`：它依赖 Nuxt 上下文、官方要求服务端必须赶在布局被 Vue 渲染之前调用（否则水合不匹配），以及 v4.3 起如何连 props 一起传。

## 本文依据

- `https://nuxt.com/docs/4.x/api/utils/define-page-meta`
  - 原文（`scrollToTop`）："Tell Nuxt to scroll to the top before rendering the page or not. Navigation is independent from rendering, so scroll behavior is always triggered even when the page doesn't re-render (e.g. when using a fixed `key`). Set `scrollToTop: false` to disable scrolling in such cases. If you want to overwrite the default scroll behavior of Nuxt, you can do so in `~/router.options.ts`."；Type `boolean | ((to: RouteLocationNormalizedLoaded, from: RouteLocationNormalizedLoaded) => boolean)`
- `https://nuxt.com/docs/4.x/guide/recipes/custom-routing`
  - 原文（Router Config）："Using router options, you can optionally override or extend your routes using a function that accepts the scanned routes and returns customized routes."；"If it returns `null` or `undefined`, Nuxt will fall back to the default routes (useful to modify input array)."；示例 `routes: _routes => [{ name: 'home', path: '/', component: () => import('~/pages/home.vue') }]`
  - 原文（note）："Nuxt will not augment any new routes you return from the `routes` function with metadata defined in `definePageMeta` of the component you provide. If you want that to happen, you should use the `pages:extend` hook which is called at build-time."
  - 原文（Using `router.options`）："This is the recommended way to specify router options."；"It is possible to add more router options files by adding files within the `pages:routerOptions` hook. Later items in the array override earlier ones."；（callout）"Adding a router options file in this hook will switch on page-based routing, unless `optional` is set, in which case it will only apply when page-based routing is already enabled."；示例 `files.push({ path: resolver.resolve('./runtime/router-options'), optional: true })`
  - 原文（Using `nuxt.config`）："**Note:** Only JSON serializable options are configurable: `linkActiveClass`、`linkExactActiveClass`、`end`、`sensitive`、`strict`、`hashMode`、`scrollBehaviorType`"；示例 `router: { options: {} }`
  - 原文（note）："With `future.compatibilityVersion: 5`, routing is case-sensitive by default to match Nitro. Set `router.options.sensitive` to `false` to opt out."
  - 原文（Hash Mode (SPA)）："You can enable hash history in SPA mode using the `hashMode` config. In this mode, router uses a hash character (#) before the actual URL that is internally passed. When enabled, the **URL is never sent to the server** and **SSR is not supported**."；示例 `ssr: false` + `router: { options: { hashMode: true } }`
  - 原文（Scroll Behavior for hash links）："You can optionally customize the scroll behavior for hash links. When you set the config to be `smooth` and you load a page with a hash link (e.g. `https://example.com/blog/my-article#comments`), you will see that the browser smoothly scrolls to this anchor."；示例 `router: { options: { scrollBehaviorType: 'smooth' } }`
  - 原文（Custom History (advanced)）："You can optionally override history mode using a function that accepts the base URL and returns the history mode. If it returns `null` or `undefined`, Nuxt will fallback to the default history."；示例 `history: base => import.meta.client ? createMemoryHistory(base) : null, /* default */`
  - 原文（Pages Hook / Nuxt Module）："You can add, change or remove pages from the scanned routes with the `pages:extend` nuxt hook."（示例 `pages.push({ name: 'profile', path: '/profile', file: '~/extra-pages/profile.vue' })` 与 `removePagesMatching(/\.ts$/, pages)`）；"The Nuxt kit provides a few ways to add routes: `extendPages` (callback: pages => void)、`extendRouteRules` (route: string, rule: NitroRouteConfig, options: ExtendRouteRulesOptions)"
- `https://nuxt.com/docs/4.x/api/nuxt-config`
  - 原文（`scrollBehaviorType`）："Customize the scroll behavior for hash links."；Type `string`；Default `"auto"`
