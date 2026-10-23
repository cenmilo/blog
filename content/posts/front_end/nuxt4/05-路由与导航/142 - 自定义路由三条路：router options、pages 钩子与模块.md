---
url: /posts/f94e079b87c1dd16b5e2326023aeac9f/
title: 142 - 自定义路由三条路：router options、pages 钩子与模块
date: 2026-11-28T10:00:00+08:00
updated: 2026-11-28T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：文件式路由之外，官方给了三条加/改/删路由的路，而这三条的差别首先是**运行时机**。开篇定位句是："在 Nuxt 里，你的路由由 `app/pages` 目录里的文件结构定义。**然而，因为它底层使用 vue-router，Nuxt 提供了几种在你的项目里添加自定义路由的方式。**"第一条是 **Router Config**："使用 router options，你可以**可选择地用一个接受被扫描到的路由、并返回被定制的路由的函数**来覆盖或扩展你的路由。**如果它返回 `null` 或 `undefined`，Nuxt 将回退到默认路由（这对于修改输入数组很有用）。**"官方示例是 `routes: _routes => [{ name: 'home', path: '/', component: () => import('~/pages/home.vue') }]`（文件标注 `router.options.ts`）——**但这一条挂着一句分量最重的警告**：**"Nuxt 不会为你从 `routes` 函数返回的任何新路由增补在你提供的组件的 `definePageMeta` 里定义的元数据。如果你希望这发生，应当使用在构建期被调用的 `pages:extend` 钩子。"** 第二条是 **Pages Hook**："**你可以用 `pages:extend` 这个 nuxt 钩子来从被扫描到的路由里添加、修改或移除页面。**"官方示例分别演示加一条（`pages.push({ name: 'profile', path: '/profile', file: '~/extra-pages/profile.vue' })`）与删一批（`removePagesMatching(/\.ts$/, pages)`，它会顺着 `page.children` 递归）。第三条是 **Nuxt Module**："如果你打算添加一整套与某个特定功能相关的页面，你可能更适合使用 Nuxt 模块。**Nuxt kit 提供了几种添加路由的方式：`extendPages`（callback: pages => void）与 `extendRouteRules`（route: string, rule: NitroRouteConfig, options: ExtendRouteRulesOptions）。**"其中 `extendPages` 的参数说明带一条硬要求：**"注意你应当直接修改被提供的 `pages` 数组，因为对被复制的数组所做的改动不会被反映到配置里。"** 而它可写的每条记录有七个属性（`name`、`path`、`file`、`meta`、`alias`、`redirect`、`children`），其中 `path` 官方注明"**如果未设置，Nuxt 会从文件位置推断它**"、`meta` 是"**路由的自定义元数据，可以用在布局、中间件或导航守卫里**"。至于路由的**选项**（不是路由表），官方另给两处：`app/router.options.ts` 是"**指定 router options 的推荐方式**"、可通过 `pages:routerOptions` 钩子加更多文件且"**数组里靠后的覆盖靠前的**"（并注：**在这个钩子里添加 router options 文件会开启基于页面的路由，除非设置了 `optional`**），而 `nuxt.config` 里**只有可 JSON 序列化的七个选项可配**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 自定义路由
  - pages
  - 钩子
  - 模块

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


"加一个 Nuxt 扫不出来的页面"、"把所有 `.ts` 页面从路由表里去掉"、"模块想附带一个 `/preview` 页"——同一件"改路由表"的事有三种写法。它们的差别不是语法，而是**跑在哪一期、以及拿不拿得到 `definePageMeta`**。

## 三条路的定位句

> 在 Nuxt 里，你的路由由 `app/pages` 目录里的文件结构定义。然而，**因为它底层使用 vue-router，Nuxt 提供了几种在你的项目里添加自定义路由的方式。**

| 路 | 官方名 | 时机 | 能做什么 |
| --- | --- | --- | --- |
| ① | **Router Config**（`routes` 函数） | 运行时（客户端建 router 时） | 覆盖/扩展被扫描到的路由 |
| ② | **Pages Hook**（`pages:extend`） | **构建期** | 增、改、删扫描出的页面 |
| ③ | **Nuxt Module**（`extendPages` / `extendRouteRules`） | 构建期（模块 `setup` 里） | 同上，且成套交付 |

**②③ 都在构建期，①不在**——这一点直接决定了下一节那条警告。

## ① Router Config：一条会让元数据消失的路

官方的说明与示例：

```ts [router.options.ts]
import type { RouterConfig } from '@nuxt/schema'

export default {
  // https://router.vuejs.org/api/interfaces/routeroptions#routes
  routes: _routes => [
    {
      name: 'home',
      path: '/',
      component: () => import('~/pages/home.vue'),
    },
  ],
} satisfies RouterConfig
```

两个官方行为：

- 函数**接受被扫描到的路由、返回被定制的路由**；
- "**如果它返回 `null` 或 `undefined`，Nuxt 将回退到默认路由（这对于修改输入数组很有用）。**"

然后是这一节最重要的一句，官方专门用 note 标出来：

> **Nuxt 不会为你从 `routes` 函数返回的任何新路由增补在你提供的组件的 `definePageMeta` 里定义的元数据。如果你希望这发生，应当使用在构建期被调用的 `pages:extend` 钩子。**

**这条警告的实际含义很具体**：用 `routes` 函数手工加的那条 `/`，即使它指向的 `home.vue` 里写了 `definePageMeta({ layout: 'admin', middleware: ['auth'], validate })`，**这些也不会生效**（135 讲过 `definePageMeta` 的元数据是**编译期被提取到路由记录上**的，编译器只处理它扫描到的那批页面）。

于是选择标准出来了：

| 你要的 | 用 |
| --- | --- |
| 只是**改掉整张路由表**（或就地修改输入数组） | `routes` 函数（返回 `null` 可回退默认） |
| 新加/改动的页面**需要** `layout`、`middleware`、`validate` 等元数据 | **`pages:extend`** |

## ② Pages Hook：增、删都直接改数组

官方定位："**你可以用 `pages:extend` 这个 nuxt 钩子来从被扫描到的路由里添加、修改或移除页面。**"

```ts [nuxt.config.ts]
import type { NuxtPage } from '@nuxt/schema'

export default defineNuxtConfig({
  hooks: {
    'pages:extend' (pages) {
      // add a route
      pages.push({
        name: 'profile',
        path: '/profile',
        file: '~/extra-pages/profile.vue',
      })

      // remove routes
      function removePagesMatching (pattern: RegExp, pages: NuxtPage[] = []) {
        const pagesToRemove: NuxtPage[] = []
        for (const page of pages) {
          if (page.file && pattern.test(page.file)) {
            pagesToRemove.push(page)
          } else {
            removePagesMatching(pattern, page.children)
          }
        }
        for (const page of pagesToRemove) {
          pages.splice(pages.indexOf(page), 1)
        }
      }
      removePagesMatching(/\.ts$/, pages)
    },
  },
})
```

三个可复用的写法点：

| 点 | 说明 |
| --- | --- |
| 加一条记录用 `name` / `path` / `file` | 与 ③ 里 `extendPages` 的字段是同一套（下表） |
| `file` 可以写 `~/extra-pages/profile.vue` | **页面文件不必在 `app/pages/` 里也能成为路由**（这正是"扫不出来的页面"的解法） |
| 删除要**递归 `children`** | 官方那个 `removePagesMatching` 就是"匹配不到 file 时下探子路由"的写法 |

另外记一句 129 的连带规则：在这一层写 `page.meta.middleware`，**官方注明"这会覆盖页面里 `definePageMeta` 设置的任何中间件"**——构建期对元数据的写入是最终答案（143 展开）。

## ③ Nuxt Module：成套交付

官方定位："如果你打算添加一整套与某个特定功能相关的页面，你可能更适合使用 Nuxt 模块。Nuxt kit 提供了几种添加路由的方式：`extendPages`（callback: pages => void）与 `extendRouteRules`（route: string, rule: NitroRouteConfig, options: ExtendRouteRulesOptions）。"

`extendPages` 的用法与那条硬要求：

```ts
import { createResolver, defineNuxtModule, extendPages } from '@nuxt/kit'

export default defineNuxtModule({
  setup (options) {
    const { resolve } = createResolver(import.meta.url)

    extendPages((pages) => {
      pages.unshift({
        name: 'prismic-preview',
        path: '/preview',
        file: resolve('runtime/preview.vue'),
      })
    })
  },
})
```

官方对 callback 参数的原话：

> 一个会被传入 pages 配置的函数。你可以通过添加、删除或修改其元素来改变这个数组。**注意：你应当直接修改被提供的 `pages` 数组，因为对被复制的数组所做的改动不会被反映到配置里。**

**也就是说：`pages.slice()` 再改、或者 `map` 出一个新数组赋回局部变量，都是白做**——必须原地改（`push`/`unshift`/`splice`/改元素）。

每条记录可写的七个属性（官方表）：

| 属性 | 必需 | 官方说明 |
| --- | --- | --- |
| `name` | 否 | 路由的名字。"**对程序化导航与标识路由很有用。**" |
| `path` | 否 | 路由 URL 路径。**"如果未设置，Nuxt 会从文件位置推断它。"** |
| `file` | 否 | 应被用作该路由组件的 **Vue 文件的路径** |
| `meta` | 否 | 路由的自定义元数据。**"可以用在布局、中间件或导航守卫里。"** |
| `alias` | 否 | 该路由的一个或多个别名路径。"**对支持多个 URL 很有用。**" |
| `redirect` | 否 | 该路由的重定向规则。"**支持命名路由、对象或字符串路径。**" |
| `children` | 否 | 该路由下的嵌套子路由，"**用于布局或视图嵌套**" |

**`path` 可省略这一格值得注意**：不写时 Nuxt 从文件位置推断——与文件式路由是同一套推导（110）。`meta` 要带类型的做法在 131 引过（增强 `@nuxt/schema` 的 `NuxtPageMeta`）。

## 别和"路由的选项"混：那是另一处

改**路由表**用上面三条；改**router 的行为**官方另给两处（138 展开过，这里只放对照）：

| 想改的东西 | 去处 |
| --- | --- |
| 路由表内容（增/删/改记录） | ①②③ |
| router 的选项（含函数形态：`scrollBehavior`、`history`、`routes`） | `app/router.options.ts`——**"这是指定 router options 的推荐方式"** |
| 多份 router options 文件 | `pages:routerOptions` 钩子里 push；**"数组里靠后的覆盖靠前的"**；且"**添加一个 router options 文件会开启基于页面的路由，除非设置了 `optional`**" |
| 只有 JSON 可序列化的七个 router 选项 | `nuxt.config` 的 `router.options`：`linkActiveClass`、`linkExactActiveClass`、`end`、`sensitive`、`strict`、`hashMode`、`scrollBehaviorType` |

（最后那七个里 `hashMode` 有两条连带硬限制——URL 从不发给服务器、不支持 SSR——见 138。）

## 三条路怎么选

| 你的场景 | 选 |
| --- | --- |
| 一次性的、需要 `definePageMeta` 元数据的新页面 | **`pages:extend`**（②） |
| 只想换掉/整理整张路由表，或就地修改输入数组 | `routes` 函数（①），并接受"新路由没有页面元数据" |
| 一个功能包（页面 + 规则 + 中间件）要发布复用 | 模块：`extendPages` + `extendRouteRules`（③），**原地改数组** |
| 只是想让某些路径不 SSR / 缓存 / 重定向 | 别动路由表，用 `routeRules`（140、141） |
| 想保护接口数据 | 服务端那道门（133），不是路由表 |

## 自查

1. 官方说 Nuxt 提供自定义路由方式的**前提**是什么（"因为……"那半句）？
2. `routes` 函数返回 `null` 或 `undefined` 会怎样？为什么"这对于修改输入数组很有用"？
3. 用 `routes` 函数新加的路由，为什么拿不到 `definePageMeta` 的元数据？官方给的替代是什么？
4. `pages:extend` 里官方怎么演示"递归删除"？为什么必须处理 `children`？
5. `extendPages` 的 callback 参数上，官方对"怎么改数组"提了什么硬要求？
6. 每条记录七个属性里，`path` 未设置时会怎样？`meta` 能用在哪些地方？
7. 想改 `scrollBehavior` 这类函数形态的 router 选项，该去哪个文件？`nuxt.config` 那一层为什么不行？

## 下一篇

143 深挖 `pages:extend`：能改哪些字段、与 `definePageMeta` 的覆盖关系、`NuxtPage` 与 `PageMeta` 两套类型怎么分别增强，以及为什么"在钩子里设 `meta.middleware` 会盖掉页面里的声明"。
