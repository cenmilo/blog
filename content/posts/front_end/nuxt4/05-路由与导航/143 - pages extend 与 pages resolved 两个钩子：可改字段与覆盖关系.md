---
url: /posts/f4e583012ce13ab7bc61b58af1b153d9/
title: pages extend 与 pages resolved 两个钩子：可改字段与覆盖关系
date: 2026-11-28T11:00:00+08:00
updated: 2026-11-28T11:00:00+08:00
author: cmdragon

summary:
  摘要：上一篇把"改路由表"的三条路摆在一起，这一篇只挖其中那条构建期的路，因为官方在钩子表里给了它**两个**相邻的钩子，而名字很容易被当成同一个东西的两种写法。官方对 `pages:extend` 的定义是："**Called after page routes are scanned from the file system.**"（在页面路由**从文件系统被扫描出来之后**调用），对 `pages:resolved` 的定义是："**Called after page routes have been augmented with scanned metadata.**"（在页面路由**被增补上扫描到的元数据之后**调用）。中间那一步"扫描元数据"，扫的正是每个页面文件里 `definePageMeta` 写下的东西。顺序在 v4 被官方明确改写过：升级指南的原话是 "**We now scan page metadata (defined in `definePageMeta`) *after* calling the `pages:extend` hook rather than before.**"，理由是"**This was to allow scanning metadata for pages that users wanted to add in `pages:extend`. We still offer an opportunity to change or override page metadata in a new `pages:resolved` hook.**"，并直接给出迁移指令："**If you want to override page metadata, do that in `pages:resolved` rather than in `pages:extend`.**"。这条顺序决定了两件事：**在 `pages:extend` 里读不到页面自己写的 `definePageMeta`，想盖掉它也应当换到 `pages:resolved`**。而能被构建期读到的 `definePageMeta` 字段是一份**封闭清单**——"`alias`, `name`, `path`, `redirect`, `props` and `middleware`"六个，且"**This only works with static or strings/arrays rather than variables or conditional assignment.**"；额外的 key 要靠 `experimental.extraPageMetaExtractionKeys` 显式加。每条路由记录可改的字段是那七个（`name`/`path`/`file`/`meta`/`alias`/`redirect`/`children`），`path` 官方注明"**If not set, Nuxt will infer it from the file location.**"。类型增强分两处：钩子里写自定义元数据增强 `@nuxt/schema` 的 `NuxtPageMeta`，页面里 `definePageMeta` 用的则是 `#app` 的 `PageMeta`。

categories:
  - 前端开发

tags:
  - nuxt4
  - pages extend
  - pages resolved
  - definePageMeta
  - 钩子

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


我在 `nuxt.config` 里写了 `pages:extend`，想看看某个页面里 `definePageMeta({ layout: 'admin' })` 到底写了什么——结果 `page.meta` 是 `undefined`。不是 bug：**那一刻元数据还没被扫描**。这一篇把这两个钩子的分工、顺序、能改的字段和"谁能盖掉谁"一次讲清。

## 两个钩子，一句话定义

官方钩子表（构建期那半张）里这两行挨着放：

| Hook | Arguments | Description（原文） |
| --- | --- | --- |
| `pages:extend` | `pages` | Called after page routes are scanned from the file system. |
| `pages:resolved` | `pages` | Called after page routes have been augmented with scanned metadata. |
| `pages:routerOptions` | `{ files: Array<{ path: string, optional?: boolean }> }` | Called when resolving `router.options` files. Later items in the array override earlier ones. |

把两句英文直译出来，差异就一个词：

- `pages:extend`：**路由被从文件系统扫描出来之后**。
- `pages:resolved`：路由**被增补上扫描到的元数据**之后。

中间那步"扫描元数据"，扫的就是每个页面文件里 `definePageMeta(...)` 的内容。所以时间线是：

```
扫文件得到路由树 → pages:extend → 扫描各页面的 definePageMeta 并增补到记录上 → pages:resolved
```

**这条线不是我推的**，官方在升级指南里把它当成一次行为变更写了出来：

> We now scan page metadata (defined in `definePageMeta`) *after* calling the `pages:extend` hook rather than before.

## 顺序为什么改：为了让新页面也有元数据

紧接着那句变更说明，官方给了理由：

> This was to allow scanning metadata for pages that users wanted to add in `pages:extend`. We still offer an opportunity to change or override page metadata in a new `pages:resolved` hook.

拆开来是两半：

- **改顺序的收益**：你在 `pages:extend` 里 `push` 一个新页面（上一篇那种 `/profile`、`/preview`），它指向的 `.vue` 文件里写的 `definePageMeta` **也能被扫到**。如果先扫元数据再调 `extend`，那新加的页面永远带不上自己的元数据——这正是 `routes` 函数那条路的毛病（"Nuxt will not augment any new routes you return from the `routes` function with metadata defined in `definePageMeta`"，见 142）。
- **改顺序的代价**：`extend` 跑得比扫描早，所以**它读不到、也不能可靠地盖掉页面里已经写的元数据**。官方给的补偿就是新钩子 `pages:resolved`。

于是官方的迁移指令非常直白：

> If you want to override page metadata, do that in `pages:resolved` rather than in `pages:extend`.

配套的官方示例就是一个 diff，把钩子名换掉、body 不动：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  hooks: {
    'pages:resolved' (pages) {
      const myPage = pages.find(page => page.path === '/')
      myPage.meta ||= {}
      myPage.meta.layout = 'overridden-layout'
    },
  },
})
```

注意这个例子的**写法**本身就是信息：它按 `page.path === '/'` 找记录，然后写 `page.meta.layout`。也就是说 `pages:resolved` 拿到的 `pages` 里，`meta` 已经是**页面自己声明过的那份**，你的赋值是在它之后发生的。

## 一处仍写着"会覆盖"的注释：别把它当保证

有件事得说清楚，否则你会在文档里看到两个互相打架的表述。

中间件那一页的"Setting Middleware at Build Time"小节，示例里有这么一行注释：

```ts
page.meta ||= {}
// Note that this will override any middleware set in `definePageMeta` in the page
page.meta.middleware = ['named']
```

它在 `pages:extend` 里写 `meta.middleware`，并声称这会覆盖页面里 `definePageMeta` 设的中间件。而升级指南说的是"要覆盖元数据请搬到 `pages:resolved`"。

**我的处理原则**：顺序（`extend` 早于扫描）是官方明写的变更事实，"覆盖"那句只是示例里的一行注释。**当两者冲突时按顺序来**——需要保证盖得住，就写到 `pages:resolved`；`pages:extend` 里挂中间件这个用法本身是官方演示过的，适用于"这个页面自己没声明中间件"的场景（这恰好是它的常见用途：批量给一批页面加，而不是和页面打架）。

真要确认你项目里的实际行为，办法很笨但有效：两处钩子都打一行日志，看目标页的 `meta.middleware` 在 `extend` 和 `resolved` 时分别是什么；再在两个同名中间件里各 `console.log` 一句，看导航时实际跑的是哪个。这个检查一次就够，之后按结论写。

## 每条路由记录可改什么：七个字段

`pages:extend` / `pages:resolved` 的 `pages` 是 `NuxtPage[]`，每条记录可写的属性（官方表）：

| 属性 | 必需 | 类型 | 官方说明要点 |
| --- | --- | --- | --- |
| `name` | 否 | `string` | 路由的名字，"Useful for programmatic navigation and identifying routes." |
| `path` | 否 | `string` | "If not set, Nuxt will infer it from the file location." |
| `file` | 否 | `string` | "Path to the Vue file that should be used as the component for the route." |
| `meta` | 否 | `NuxtPageMeta` | "Custom metadata for the route. Can be used in layouts, middlewares, or navigation guards." |
| `alias` | 否 | `string \| string[]` | "One or more alias paths for the route. Useful for supporting multiple URLs." |
| `redirect` | 否 | `RouteRecordRedirectOption` | "Redirect rule for the route. Supports named routes, objects, or string paths." |
| `children` | 否 | `NuxtPage[]` | "Nested child routes under this route for layout or view nesting." |

三个实操点，都来自官方那两个示例：

- **`file` 是指向 Vue 文件的路径字符串**，且可以用 `~` 前缀（`file: '~/extra-pages/profile.vue'`）。这意味着"页面文件不在 `app/pages` 里"照样能成路由——文件式路由的限制只在于**扫描范围**，不在于路由表。
- **删除要自己递归**。官方那段 `removePagesMatching` 的逻辑是：`if (page.file && pattern.test(page.file))` 收集，否则 `removePagesMatching(pattern, page.children)` 下探；最后用 `pages.splice(pages.indexOf(page), 1)` 摘掉。注意匹配的是 `page.file` 而不是 `page.path`。
- **`path` 可以省略**：Nuxt 会从文件位置推断。所以模块里加页面时，`file` 给绝对解析路径、`path` 给字符串是最省心的写法（官方模块示例用 `resolve('runtime/preview.vue')` 生成 `file`）。

## `meta` 里构建期看得到什么：一份六个 key 的封闭清单

这是这两个钩子最反直觉的地方。`definePageMeta` 有十几个字段（135 那张表），但**构建期能被扫出来的只有一小部分**，官方在实验特性页写明了清单：

> Nuxt exposing some route metadata defined in `definePageMeta` at build-time to modules (specifically `alias`, `name`, `path`, `redirect`, `props` and `middleware`).

六个：`alias`、`name`、`path`、`redirect`、`props`、`middleware`。想验证为什么是这六个：它们都是**路由记录本身的构造参数**（`vue-router` 建记录时就需要的东西），而 `layout`、`pageTransition`、`keepalive` 这些是**渲染期行为**，不需要在构建期暴露。

同一页还给了两个限制条件：

- **写法必须能被静态提取**："This only works with static or strings/arrays rather than variables or conditional assignment." 也就是说 `middleware: ['auth']` 能扫到，`middleware: someVar` 或 `middleware: cond ? ['a'] : ['b']` 扫不到。（117 讲 `definePageMeta` 是编译器宏、参数只能是纯函数与导入绑定，是同一套约束的两面。）
- **时机**："By default page metadata is only scanned after all routes have been registered in `pages:extend`. Then another hook, `pages:resolved` will be called." 这句话同时确认了两钩子的顺序，也确认了这套扫描**默认是开着的**；如果它在你项目里引起问题，可以关掉：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    scanPageMeta: false,
  },
})
```

自定义 key 默认不在清单里，得显式加：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    extraPageMetaExtractionKeys: ['foo'],
  },
  hooks: {
    'pages:resolved' (ctx) {
      // ✅ foo is available
    },
  },
})
```

官方对这段的说明："This allows modules to access additional metadata from the page metadata in the build context. If you are using this within a module, it's recommended also to **augment the `NuxtPage` types** with your keys."

## 类型增强有两处，别写错地方

页面里和钩子里的自定义元数据是**两个不同的接口**，各自增强：

```ts [index.d.ts]
declare module '#app' {
  interface PageMeta {
    pageType?: string
  }
}

// It is always important to ensure you import/export something when augmenting a type
export {}
```

```ts [index.d.ts]
declare module '@nuxt/schema' {
  interface NuxtPageMeta {
    requiresAuth?: boolean
  }
}

export {}
```

| 你在写 | 增强 | 依据原文 |
| --- | --- | --- |
| 页面里的 `definePageMeta({ pageType: 'blog' })` | `#app` 的 `PageMeta` | "If you add custom metadata for your pages, you may wish to do so in a type-safe way. It is possible to augment the type of the object accepted by `definePageMeta`" |
| 钩子里的 `page.meta.requiresAuth = true`、或模块里的 `extendPages` | `@nuxt/schema` 的 `NuxtPageMeta` | "You can augment the `NuxtPageMeta` interface to add type-safe custom metadata for pages when using `extendPages` or the `pages:extend` hook" |

`export {}` 那行不是装饰，官方特意注释："It is always important to ensure you import/export something when augmenting a type"——没有它，`declare module` 会变成**替换**而不是**增强**该模块。

顺带一个文档自身的小不一致：中间件页示例写 `import type { NuxtPage } from 'nuxt/schema'`，自定义路由页与 kit 页写的是 `'@nuxt/schema'`。两处都出现在官方示例里，跟着你项目里已有的写法走即可。

## 该用哪一个：按动作分

| 我要做的事 | 用 | 为什么 |
| --- | --- | --- |
| 加一条 Nuxt 扫不出来的页面 | `pages:extend`（或模块里 `extendPages`） | 这是它的定义用途；且新页面的 `definePageMeta` 也会被扫到 |
| 批量删掉一批页面（含子路由） | `pages:extend` | 官方示例就在这层，记得递归 `children` |
| 给一批页面挂同一个命名中间件 | `pages:extend` | 官方"Setting Middleware at Build Time"演示的写法 |
| 读某个页面自己声明的 `alias`/`name`/`path`/`redirect`/`props`/`middleware` | `pages:resolved` | 扫描发生在 `extend` 之后 |
| 覆盖页面里已经写在 `definePageMeta` 的字段 | `pages:resolved` | 官方迁移指令原话 |
| 读到某个自定义 key（如 `foo`） | `experimental.extraPageMetaExtractionKeys` + `pages:resolved` | 默认清单外，需显式加 |
| 换掉整张路由表 | `router.options.ts` 的 `routes` 函数 | 但要接受"新路由没有页面元数据"（142） |
| 只是想让某些 URL 不 SSR / 加缓存头 | 别动路由表，用 `routeRules`（140、141） | 那是服务端那一层的事 |

## 自查

1. `pages:extend` 与 `pages:resolved` 在官方钩子表里的定义分别是什么？中间那一步"扫描元数据"扫的是什么？
2. v4 为什么把扫描元数据改到 `pages:extend` **之后**？这个改动让哪类页面第一次拿到了自己的元数据？
3. 官方给的"要覆盖页面元数据"的正确去处是哪个钩子？它那个示例改的是哪个字段？
4. 构建期能扫到的 `definePageMeta` 字段有哪六个？为什么 `layout` 不在里面？
5. `middleware: someVar` 这种写法为什么扫不到？官方原话的限制是什么？
6. `pages:extend` 里删页面时，官方的匹配条件用的是 `page.file` 还是 `page.path`？为什么要递归？
7. 钩子里写自定义元数据，要增强哪个接口？页面里 `definePageMeta` 用哪个？`export {}` 起什么作用？
8. 中间件页那行"this will override any middleware set in `definePageMeta`"和升级指南的迁移指令有张力时，我按什么原则处理？

## 下一篇

144 转到无障碍播报这一侧：`useRouteAnnouncer` 与 `useAnnouncer` 各自的用法、`politeness` 与 `atomic` 两个属性怎么影响播报，以及页面内自己控制播报时为什么不该再用路由播报器。

## 本文依据

- `https://nuxt.com/docs/4.x/api/advanced/hooks`
  - 原文（页面相关三行）："`pages:extend` | `pages` | Called after page routes are scanned from the file system."；"`pages:resolved` | `pages` | Called after page routes have been augmented with scanned metadata."；"`pages:routerOptions` | `{ files: Array<{ path: string, optional?: boolean }> }` | Called when resolving `router.options` files. Later items in the array override earlier ones."
  - 原文（两张表的位置）："## Nuxt Hooks (Build Time)"及其 "Check the [schema source code](https://github.com/nuxt/nuxt/blob/main/packages/schema/src/types/hooks.ts#L83) for all available hooks."；"## App Hooks (Runtime)"及 "Check the app source code for all available hooks."
- `https://nuxt.com/docs/4.x/getting-started/upgrade`
  - 原文（变更）："We now scan page metadata (defined in `definePageMeta`) *after* calling the `pages:extend` hook rather than before."
  - 原文（理由）："This was to allow scanning metadata for pages that users wanted to add in `pages:extend`. We still offer an opportunity to change or override page metadata in a new `pages:resolved` hook."
  - 原文（迁移步骤）："If you want to override page metadata, do that in `pages:resolved` rather than in `pages:extend`."；diff 示例由 `'pages:extend'(pages)` 改为 `'pages:resolved'(pages)`，body 为 `const myPage = pages.find(page => page.path === '/')` + `myPage.meta ||= {}` + `myPage.meta.layout = 'overridden-layout'`
- `https://nuxt.com/docs/4.x/guide/going-further/experimental-features`
  - 原文（`scanPageMeta`）："Nuxt exposing some route metadata defined in `definePageMeta` at build-time to modules (specifically `alias`, `name`, `path`, `redirect`, `props` and `middleware`)."；"This only works with static or strings/arrays rather than variables or conditional assignment. See original issue for more information and context."；"By default page metadata is only scanned after all routes have been registered in `pages:extend`. Then another hook, `pages:resolved` will be called."；"You can disable this feature if it causes issues in your project."；示例 `experimental: { scanPageMeta: false }`
  - 原文（`extraPageMetaExtractionKeys`）：示例 `experimental: { extraPageMetaExtractionKeys: ['foo'] }` 配 `hooks: { 'pages:resolved' (ctx) { // ✅ foo is available } }`；"This allows modules to access additional metadata from the page metadata in the build context. If you are using this within a module, it's recommended also to augment the `NuxtPage` types with your keys."
- `https://nuxt.com/docs/4.x/guide/recipes/custom-routing`
  - 原文（Pages Hook）："You can add, change or remove pages from the scanned routes with the `pages:extend` nuxt hook."；"For example, to prevent creating routes for any `.ts` files:"；示例 `import type { NuxtPage } from '@nuxt/schema'` + `pages.push({ name: 'profile', path: '/profile', file: '~/extra-pages/profile.vue' })` + `function removePagesMatching (pattern: RegExp, pages: NuxtPage[] = []) { for (const page of pages) { if (page.file && pattern.test(page.file)) { pagesToRemove.push(page) } else { removePagesMatching(pattern, page.children) } } for (const page of pagesToRemove) { pages.splice(pages.indexOf(page), 1) } }` + `removePagesMatching(/\.ts$/, pages)`
  - 原文（routes 函数的元数据限制）："Nuxt will not augment any new routes you return from the `routes` function with metadata defined in `definePageMeta` of the component you provide. If you want that to happen, you should use the `pages:extend` hook which is [called at build-time]."
- `https://nuxt.com/docs/4.x/api/kit/pages`
  - 原文（`NuxtPage` 七属性表）：`name` — "The name of the route. Useful for programmatic navigation and identifying routes."；`path` — "The route URL path. If not set, Nuxt will infer it from the file location."；`file` — "Path to the Vue file that should be used as the component for the route."；`meta` — "Custom metadata for the route. Can be used in layouts, middlewares, or navigation guards."；`alias` — "One or more alias paths for the route. Useful for supporting multiple URLs."；`redirect` — "Redirect rule for the route. Supports named routes, objects, or string paths."；`children` — "Nested child routes under this route for layout or view nesting."
  - 原文（Typing Custom Page Metadata）："You can augment the `NuxtPageMeta` interface to add type-safe custom metadata for pages when using `extendPages` or the `pages:extend` hook"；示例 `declare module '@nuxt/schema' { interface NuxtPageMeta { requiresAuth?: boolean } }` + `export {}`
  - 原文（原地修改数组的要求）："**Note: You should modify the provided `pages` array directly, as changes made to a copied array will not be reflected in the configuration.**"
- `https://nuxt.com/docs/4.x/directory-structure/app/middleware`
  - 原文（Setting Middleware at Build Time）："Instead of using `definePageMeta` on each page, you can add named route middleware within the `pages:extend` hook."；示例 `import type { NuxtPage } from 'nuxt/schema'` + `function setMiddleware (pages: NuxtPage[]) { for (const page of pages) { if (/* some condition */ Math.random() > 0.5) { page.meta ||= {}; // Note that this will override any middleware set in `definePageMeta` in the page \n page.meta.middleware = ['named'] } if (page.children) { setMiddleware(page.children) } } }` + `setMiddleware(pages)`
- `https://nuxt.com/docs/4.x/directory-structure/app/pages`
  - 原文（Typing Custom Metadata）："If you add custom metadata for your pages, you may wish to do so in a type-safe way. It is possible to augment the type of the object accepted by `definePageMeta`"；示例 `declare module '#app' { interface PageMeta { pageType?: string } }` + "// It is always important to ensure you import/export something when augmenting a type" + `export {}`
- `https://nuxt.com/docs/4.x/api/utils/define-page-meta`
  - 对照用（见 117、135）：`export function definePageMeta (meta: PageMeta): void` 与 `interface PageMeta { validate?…, redirect?…, name?…, path?…, props?…, alias?…, groups?…, pageTransition?…, layoutTransition?…, viewTransition?…, keepalive?… }`；`groups` 那行带 v4.3 标记，原文为 "Route groups the page belongs to, based on the folder structure. Automatically populated for pages within route groups."；`alias` 原文为 "Aliases for the record. Allows defining extra paths that will behave like a copy of the record. Allows having paths shorthands like `/users/:id` and `/u/:id`. All `alias` and `path` values must share the same params."
