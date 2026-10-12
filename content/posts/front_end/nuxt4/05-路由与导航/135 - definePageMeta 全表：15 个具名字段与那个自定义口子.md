---
url: /posts/05fca5f668b26e2470ead63fb1914592/
title: definePageMeta 全表：15 个具名字段与那个自定义口子
date: 2026-11-26T09:00:00+08:00
updated: 2026-11-26T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇把 `definePageMeta` 一次列全，作为后面几篇的地图。官方的定义句是："`definePageMeta` 是一个**编译器宏**，你可以用它来为你位于 `app/pages/` 目录（除非另行设置）里的**页面**组件设置元数据。这样一来，你就可以为 Nuxt 应用的每一条静态或动态路由设置自定义元数据。"最小写法是 `definePageMeta({ layout: 'default' })`（文件标注 `app/pages/some-page.vue`）。签名里带名字的字段共 **15 个**，外加一个自定义口子：`validate?: ((route: RouteLocationNormalized) => boolean | Promise<boolean> | Partial<NuxtError> | Promise<Partial<NuxtError>>)`、`redirect?: RouteRecordRedirectOption`、`name?: string`、`path?: string`、`props?: RouteRecordRaw['props']`、`alias?: string | string[]`、`groups?: string[]`、`pageTransition?: boolean | TransitionProps`、`layoutTransition?: boolean | TransitionProps`、`viewTransition?: ViewTransitionPageOptions['enabled'] | ViewTransitionPageOptions`、`key?: false | string | ((route: RouteLocationNormalizedLoaded) => string)`、`keepalive?: boolean | KeepAliveProps`、`layout?: false | LayoutKey | Ref<LayoutKey> | ComputedRef<LayoutKey> | { name?: LayoutKey | false, props?: Record<string, unknown> }`、`middleware?: MiddlewareKey | NavigationGuard | Array<MiddlewareKey | NavigationGuard>`、`scrollToTop?: boolean | ((to, from) => boolean)`，最后是 **`[key: string]: unknown`**。几个容易被漏读的官方细节：`redirect` 是"**如果在路由被直接匹配到时要把用户重定向到别处。重定向发生在任何导航守卫之前，并会用新的目标位置触发一次新的导航。**"；`validate` 是"**验证一条给定路由是否可以合法地由这个页面渲染。如果有效返回 true，否则返回 false。如果找不到其他匹配，这就意味着一个 404。你也可以直接返回一个带 `status`/`statusText` 的对象来立即以错误响应（其他匹配将不再被检查）。**"；`scrollToTop` 附带一条重要说明——"**导航独立于渲染，所以即使页面不重新渲染（例如使用固定的 `key` 时），滚动行为也总会触发。在这种情形下设 `scrollToTop: false` 来禁用滚动。如果你想覆盖 Nuxt 的默认滚动行为，可以在 `~/router.options.ts` 里做。**"；`viewTransition` 标注为实验特性且"**只有当你在 nuxt.config 文件里启用它时才可用**"；`groups` 是 v4.3 起、"**基于目录结构，页面所属的路由组。对位于路由组内的页面会自动填充**"；自定义字段官方说"**除了上面的属性，你也可以设置自定义元数据**"，并建议用类型增强（`declare module '#app' { interface PageMeta { pageType?: string } }` 且必须带 `export {}`）。

categories:
  - 前端开发

tags:
  - nuxt4
  - definepagemeta
  - 页面元数据
  - 速查
  - 类型

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


前面好几篇都在用 `definePageMeta`。这一篇把它的所有字段摊在一张表上——**后面每个字段的深入篇都会回到这张图**，所以先建立全局，再逐格挖。

## 定义与最小写法

> `definePageMeta` 是一个**编译器宏**，你可以用它来为你位于 `app/pages/` 目录（除非另行设置）里的**页面**组件设置元数据。这样一来，你就可以为 Nuxt 应用的每一条静态或动态路由设置自定义元数据。

```vue [app/pages/some-page.vue]
<script setup lang="ts">
definePageMeta({
  layout: 'default',
})
</script>
```

三个限定词都在这一句里：**编译器宏**（117 讲过它的四条限制）、**只对页面组件有效**（`app/pages/`）、**它设置的是路由的元数据**（不是组件的属性）。

## 全表：15 个具名字段

| 字段 | 类型 | 官方的作用说明（要点） | 深入篇 |
| --- | --- | --- | --- |
| `validate` | `(route: RouteLocationNormalized) => boolean \| Promise<boolean> \| Partial<NuxtError> \| Promise<Partial<NuxtError>>` | 验证这条路由能否由本页面渲染；**返回 `false` 意味着 404（找不到其他匹配时）**；也可返回带 `status`/`statusText` 的对象**立即以错误响应，其他匹配不再检查** | 111 |
| `redirect` | `RouteRecordRedirectOption` | 路由被直接匹配到时重定向到别处；**"重定向发生在任何导航守卫之前"**，并触发一次到新目标的新导航 | 136 |
| `name` | `string` | 为这条路由定义名字；**默认名字基于它在 `app/pages/` 里的路径生成** | 112 |
| `path` | `string` | 比文件名所能表达的更复杂的模式时用（自定义正则） | 113 |
| `props` | `RouteRecordRaw['props']` | 允许把路由 `params` 作为 props 传给页面组件 | 136 |
| `alias` | `string \| string[]` | "Aliases for the record"，额外路径表现如同该记录的复制；**`alias` 与 `path` 的所有值必须共享同样的 params** | 113 |
| `groups` | `string[]`（v4.3 徽章） | 基于目录结构，页面所属的路由组；**对位于路由组内的页面自动填充** | 116 |
| `pageTransition` | `boolean \| TransitionProps` | 设置本页面的过渡名；**设 `false` 可禁用页面过渡** | 118、119 |
| `layoutTransition` | `boolean \| TransitionProps` | 设置本页面所用布局的过渡名；同样可 `false` | 118、119 |
| `viewTransition` | `boolean \| 'always' \| ViewTransitionPageOptions` | **实验特性，只有在 nuxt.config 里启用后才可用**；`true` 时浏览器匹配 `prefers-reduced-motion: reduce` 就不应用过渡（推荐），`always` 则始终应用；对象形态可配 `enabled`/`types`/`toTypes`/`fromTypes` | 119 |
| `key` | `false \| string \| ((route: RouteLocationNormalizedLoaded) => string)` | "**当你需要更多地控制 `<NuxtPage>` 组件何时重新渲染时设置 `key` 值**" | 137 |
| `keepalive` | `boolean \| KeepAliveProps` | 想在跨路由变化时保留页面状态就设 `true`；用 `KeepAliveProps` 做细粒度控制 | 119 |
| `layout` | `false \| LayoutKey \| Ref<LayoutKey> \| ComputedRef<LayoutKey> \| { name?: LayoutKey \| false, props?: Record<string, unknown> }` | 为每条路由设置静态或动态的布局名；**设 `false` 用于需要禁用默认布局的情形**；对象形态可传 `name` 与 `props`，**布局用 `defineProps` 声明后这些 props 会在 `definePageMeta` 里被完整类型推断** | 136、139 |
| `middleware` | `MiddlewareKey \| NavigationGuard \| Array<MiddlewareKey \| NavigationGuard>` | 直接在这里定义匿名或命名中间件 | 128-132 |
| `scrollToTop` | `boolean \| ((to, from) => boolean)` | 告诉 Nuxt 在渲染页面前是否滚到顶部；**导航独立于渲染，所以即使页面不重渲染（例如固定 `key`）滚动行为也总会触发，此时设 `false` 禁用**；要改默认滚动行为去 `~/router.options.ts` | 138 |

自定义口子在最后：**`[key: string]: unknown`**，官方说明为"**除了上面的属性，你也可以设置自定义元数据。你可能希望以类型安全的方式来做，即增强 `meta` 对象的类型。**"

## 按意图分四组，比按字母好记

| 组 | 字段 |
| --- | --- |
| **能不能进这一页** | `validate`、`redirect`、`middleware` |
| **这一页长什么样** | `layout`、`pageTransition`、`layoutTransition`、`viewTransition`、`scrollToTop` |
| **这一页什么时候重建 / 留不留状态** | `key`、`keepalive` |
| **这条路由叫什么、匹配什么、属于哪组** | `name`、`path`、`alias`、`props`、`groups` |

第四组是"路由表层面"的，前三组是"这一次访问层面"的——**同名冲突、抢匹配那类问题只会出在第四组，不会靠第三组解决**（113 那个 `[postId]-[postSlug]` 与 `[categorySlug]` 的例子，正确答案是 `path` 而不是 `key`）。

## 这张表上有三个"顺序陷阱"

**① `redirect` 早于所有守卫。** 官方原话：重定向发生在任何导航守卫之前。所以一条设了 `redirect` 的路由，`middleware` 与组件守卫里的逻辑都轮不到它（127、134）。

**② `validate` 返回对象会跳过其他匹配。** 官方原话：可以直接返回 `status`/`statusText` 的对象"**来立即以错误响应（其他匹配将不再被检查）**"。这跟"返回 `false`"不同——返回 false 时还会去找别的匹配，找不到才 404。

**③ `key` 与 `scrollToTop` 互相独立。** 官方对 `scrollToTop` 的那句解释正是为此写的：**即使页面不重新渲染（例如用了固定的 `key`），滚动行为也总会触发**。所以"参数变了但页面复用"时，如果你不想要每次换参数都跳回顶部，要显式 `scrollToTop: false`。

## 自定义字段要类型安全，两处增强别写混

| 你要增强的类型 | 模块与接口 | 出现在 |
| --- | --- | --- |
| `definePageMeta` 接受的对象（应用侧） | `#app` 里的 `PageMeta` | 117、135 |
| `pages:extend` / `extendPages` 里的 `page.meta` | `@nuxt/schema` 里的 `NuxtPageMeta` | 131 |

两处官方都补了同一句要求：`// It is always important to ensure you import/export something when augmenting a type` + 文件末尾 `export {}`。

## 一表看清"哪一层的默认值在哪配"

| 字段 | nuxt.config 里有对应默认值吗 | 依据 |
| --- | --- | --- |
| `layout` | 无（但 `app.layoutTransition` 之类有） | 布局本身靠 `app/layouts/` + `<NuxtLayout>`（094） |
| `pageTransition` / `layoutTransition` | **有**：`app.pageTransition`、`app.layoutTransition`；官方还提醒"要覆盖全局过渡属性，请使用 `definePageMeta`" | 118 |
| `keepalive` | **有**：官方说"你可以在你的 `nuxt.config` 里为这个属性设置一个默认值" | 119 |
| `viewTransition` | **有**（实验开关 + `app.viewTransition` 改全局默认） | 119 |
| `validate` / `redirect` / `middleware` | 无全局默认，但可以**批量给**（`pages:extend` 里改 `page.meta.middleware`，官方注明这会覆盖页面的 `definePageMeta`） | 129 |

## 自查

1. `definePageMeta` 官方定义里的三个限定词是什么？它作用于哪里的组件？
2. 15 个具名字段里，哪三个是"能不能进这一页"这一组的？
3. `redirect` 与导航守卫的先后关系是什么？这对你的 `middleware` 有什么影响？
4. `validate` 返回 `false` 与返回 `{ status, statusText }` 的差别是什么？
5. `key` 与 `scrollToTop` 为什么会被放在一起讲？固定 `key` 时滚动会怎样？
6. `layout` 的对象形态能给什么？为什么官方强调"布局用 `defineProps` 声明后 props 会被完整类型推断"？
7. 自定义元数据的两处类型增强分别改哪个接口？两处都共同要求什么？

## 下一篇

136 把 `layout`、`middleware`、`alias`、`redirect` 四个字段拉到一起做实操：布局的字符串/ref/computed/false 与对象形态、中间件与父子路由的合并、alias 的 params 硬约束，以及 redirect 早于守卫的后果。

## 本文依据

- `https://nuxt.com/docs/4.x/api/utils/define-page-meta`
  - 原文（描述）："Define metadata for your page components."；"`definePageMeta` is a compiler macro that you can use to set metadata for your **page** components located in the `app/pages/` directory (unless set otherwise). This way you can set custom metadata for each static or dynamic route of your Nuxt application."；示例 `definePageMeta({ layout: 'default' })`（文件标注 `app/pages/some-page.vue`）
  - 原文（Type 签名，15 个具名字段 + 自定义口子）：`export function definePageMeta (meta: PageMeta): void` / `interface PageMeta { validate?: ((route: RouteLocationNormalized) => boolean | Promise<boolean> | Partial<NuxtError> | Promise<Partial<NuxtError>>) ; redirect?: RouteRecordRedirectOption; name?: string; path?: string; props?: RouteRecordRaw['props']; alias?: string | string[]; groups?: string[]; pageTransition?: boolean | TransitionProps; layoutTransition?: boolean | TransitionProps; viewTransition?: ViewTransitionPageOptions['enabled'] | ViewTransitionPageOptions; key?: false | string | ((route: RouteLocationNormalizedLoaded) => string); keepalive?: boolean | KeepAliveProps; layout?: false | LayoutKey | Ref<LayoutKey> | ComputedRef<LayoutKey> | { name?: LayoutKey | false, props?: Record<string, unknown> }; middleware?: MiddlewareKey | NavigationGuard | Array<MiddlewareKey | NavigationGuard>; scrollToTop?: boolean | ((to: RouteLocationNormalizedLoaded, from: RouteLocationNormalizedLoaded) => boolean); [key: string]: unknown }`
  - 原文（`redirect`）："Where to redirect if the route is directly matched. The redirection happens before any navigation guard and triggers a new navigation with the new target location."
  - 原文（`validate`）："Validate whether a given route can validly be rendered with this page. Return true if it is valid, or false if not. If another match can't be found, this will mean a 404. You can also directly return an object with `status`/`statusText` to respond immediately with an error (other matches will not be checked)."
  - 原文（`name`）："You may define a name for this page's route. By default, name is generated based on path inside the `app/pages/` directory."
  - 原文（`path`）："You may define a custom regular expression if you have a more complex pattern than can be expressed with the file name."
  - 原文（`props`）："Allows accessing the route `params` as props passed to the page component."
  - 原文（`alias`）："Aliases for the record. Allows defining extra paths that will behave like a copy of the record. Allows having paths shorthands like `/users/:id` and `/u/:id`. All `alias` and `path` values must share the same params."
  - 原文（`groups`，badge v4.3）："Route groups the page belongs to, based on the folder structure. Automatically populated for pages within route groups."
  - 原文（`keepalive`）："Set to `true` when you want to preserve page state across route changes or use the `KeepAliveProps` for a fine-grained control."
  - 原文（`key`）："Set `key` value when you need more control over when the `<NuxtPage>` component is re-rendered."
  - 原文（`layout`）："Set a static or dynamic name of the layout for each route. This can be set to `false` in case the default layout needs to be disabled."；"You can also pass an object with `name` and `props` to pass typed props to your layout component. When your layout defines props with `defineProps`, they will be fully typed in `definePageMeta`."
  - 原文（`layoutTransition` / `pageTransition`）："Set name of the transition to apply for current layout. You can also set this value to `false` to disable the layout transition."；"Set name of the transition to apply for current page. You can also set this value to `false` to disable the page transition."
  - 原文（`viewTransition`）："Experimental feature, only available when enabled in your nuxt.config file. Enable/disable View Transitions for the current page. If set to true, Nuxt will not apply the transition if the users browser matches `prefers-reduced-motion: reduce` (recommended). If set to `always`, Nuxt will always apply the transition."；对象字段 `enabled`/`types`/`toTypes`/`fromTypes`
  - 原文（`middleware`）："Define anonymous or named middleware directly within `definePageMeta`. Learn more about route middleware."
  - 原文（`scrollToTop`）："Tell Nuxt to scroll to the top before rendering the page or not. Navigation is independent from rendering, so scroll behavior is always triggered even when the page doesn't re-render (e.g. when using a fixed `key`). Set `scrollToTop: false` to disable scrolling in such cases. If you want to overwrite the default scroll behavior of Nuxt, you can do so in `~/router.options.ts`."
  - 原文（`[key: string]`）："Apart from the above properties, you can also set **custom** metadata. You may wish to do so in a type-safe way by augmenting the type of the `meta` object."
  - 原文（Passing Props to a Layout）："You can pass props to a layout by using the object syntax for `layout`. If your layout defines props with `defineProps`, the props will be fully typed."；示例 `definePageMeta({ layout: { name: 'panel', props: { sidebar: true, title: 'Dashboard' } } })` 与 `app/layouts/panel.vue` 里 `defineProps<{ sidebar?: boolean; title?: string }>()`；（tip）"Layout props set via `definePageMeta` are fully typed based on the layout's `defineProps`. You'll get autocomplete and type-checking in your editor."
  - 原文（Defining Middleware 示例）：`middleware: [function (to, from) { ... }]`、`middleware: 'auth'`、`middleware: ['auth', 'another-named-middleware']`
  - 原文 Source：`packages/nuxt/src/pages/runtime/composables.ts`
- `https://nuxt.com/docs/4.x/directory-structure/app/pages`
  - 对照用（见 117）：Page Metadata 与编译器宏四条限制；（Special Metadata）"middleware ... It will be merged with all the other middleware used in any matching parent/child routes."；（keepalive）"You can set a default value for this property in your `nuxt.config`."；（Typing Custom Metadata）`declare module '#app' { interface PageMeta { pageType?: string } }` + `export {}`
- `https://nuxt.com/docs/4.x/api/kit/pages`
  - 对照用（见 131）：`NuxtPageMeta` 的增强走 `@nuxt/schema`；`meta` — "Custom metadata for the route. Can be used in layouts, middlewares, or navigation guards."
- `https://nuxt.com/docs/4.x/getting-started/transitions`
  - 对照用（见 118）：`app.pageTransition` / `app.layoutTransition` 全局默认与 "To override the global transition property, use the `definePageMeta`..."
