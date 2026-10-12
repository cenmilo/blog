---
url: /posts/de3a4f9fd6edbcc58bc69f73510e386b/
title: 过渡与保活的三条入口：nuxt.config、NuxtPage 与 definePageMeta
date: 2026-11-20T11:00:00+08:00
updated: 2026-11-20T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`transition` 与 `keepalive` 这两件事，官方一共给了三条入口，而它们的作用范围不一样、官方明确写出的覆盖关系也只有一条，所以搞混之后的表现是"某一层的设置看起来没生效"。三条入口的原文出处是 `<NuxtPage>` 那一页这句话："**默认情况下，Nuxt 不会启用 `<Transition>` 和 `<KeepAlive>`。你可以在 nuxt.config 文件里启用它们，或者通过在 `<NuxtPage>` 上设置 `transition` 和 `keepalive` 属性来启用它们。如果你想定义某个特定的页面，可以在页面组件里通过 `definePageMeta` 来设置。**"也就是说：前两者是**全局启用**的两条路（配置文件里一次，或在出口组件上一次性对所有由它渲染的页面生效），第三者是**单页**的定制口。覆盖关系官方只在过渡那一页写明了一条："**要覆盖全局过渡属性，请使用 `definePageMeta` 为单个 Nuxt 页面定义页面或布局过渡，并覆盖在 `nuxt.config` 文件里定义的任何页面或布局过渡。**"——即 `definePageMeta` 明确覆盖 `nuxt.config`。至于 `<NuxtPage>` 上的属性与单页 `definePageMeta` 同时出现时谁生效，官方没有把它写成一条规则，所以这一层要按实测来定（方法在本篇给出）。`keepalive` 一侧官方给了三种可用写法与一条父路由专用语法："`definePageMeta` 里设 `keepalive: true` 时，Nuxt 会自动把你的页面包进 Vue 的 `<KeepAlive>` 组件……例如在**拥有动态子路由的父路由**里这么做，如果你想跨路由变化保留页面状态"；"**如果你的目标是保留父路由的状态，请使用这种语法：`<NuxtPage keepalive />`**，你也可以设置要传给 `<KeepAlive>` 的 props"；"**你可以在你的 `nuxt.config` 里为这个属性设置一个默认值。**"细粒度控制的官方示例是 `keepalive: { exclude: ['modal'] }`（配合 `key: route => route.fullPath` 与自定义字段 `pageType: 'Checkout'` 出现在同一段里）。两件事还共享同一个结构前提：`<NuxtPage>` 内部是 `RouterView → (Transition) → (KeepAlive) → Suspense → component`，而 `Transition` 与 `KeepAlive` 在官方注释里被标成 Optional——**默认都不开**，因此"保活没生效"与"过渡没生效"要先看的是同一个位置：有没有真的把这两层装上。

categories:
  - 前端开发

tags:
  - nuxt4
  - keepalive
  - 过渡
  - nuxtpage
  - viewtransition

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


上一篇配好了过渡。这一篇解决剩下的问题：同一个"要不要动画 / 要不要保活"的设置，能写在三个地方。写在哪儿决定它管多大范围，也决定它会不会被另一层盖掉。

## 官方那句话给了三条入口

`<NuxtPage>` 那一页的原文，一次说全：

> 默认情况下，Nuxt 不会启用 `<Transition>` 和 `<KeepAlive>`。**你可以在 nuxt.config 文件里启用它们，或者通过在 `<NuxtPage>` 上设置 `transition` 和 `keepalive` 属性来启用它们。如果你想定义某个特定的页面，可以在页面组件里通过 `definePageMeta` 来设置。**

| 入口 | 写法 | 作用范围 |
| --- | --- | --- |
| ① 配置文件 | `app: { pageTransition, layoutTransition, keepalive }` | 全站默认 |
| ② 出口属性 | `<NuxtPage :transition="..." :keepalive="...">` | **由这个出口渲染的所有页面** |
| ③ 页面元数据 | `definePageMeta({ pageTransition, layoutTransition, keepalive })` | **单个页面** |

①和②是"全局启用"的两条路，③是"某个页面不一样"的口。这个区分决定了后面所有排查。

## 覆盖关系：官方明确写了一条

只有一条是官方写死的，来自过渡那一页：

> **要覆盖全局过渡属性，请使用 `definePageMeta` 为单个 Nuxt 页面定义页面或布局过渡，并覆盖在 `nuxt.config` 文件里定义的任何页面或布局过渡。**

```
nuxt.config（全局默认）
      ↓ 被 definePageMeta 覆盖   ← 官方明确写的这一条
definePageMeta（单页）
```

至于**②（`<NuxtPage>` 上的属性）与③（单页 meta）同时出现谁赢**，官方没有把它写成规则。所以别猜，用下面这套最小实验在 30 秒内定论：

| 步骤 | 做法 |
| --- | --- |
| 1 | 在 `app.vue` 的出口上放一个显眼的前缀：`<NuxtPage :transition="{ name: 'outlet', mode: 'out-in' }" />` |
| 2 | 在其中一个页面里放另一个：`definePageMeta({ pageTransition: { name: 'single' } })` |
| 3 | 把两组 CSS 都写全（`.outlet-enter-active` 与 `.single-enter-active`），去那个页面来回导航 |
| 4 | 用开发者工具看包裹元素上实际出现的类名前缀——**出现哪个名字，哪一层就生效** |

结论要按你自己的 Nuxt 版本记下来，并在团队里统一一条写法：**同一件事只在一个入口配**（下面有推荐分配）。

## 结构前提：这两层默认都不存在

官方给出的 `<NuxtPage>` 内部结构里，两层都被标成可选：

```vue
<template>
  <RouterView v-slot="{ Component }">
    <!-- Optional, when using transitions -->
    <Transition>
      <!-- Optional, when using keep-alive -->
      <KeepAlive>
        <Suspense>
          <component :is="Component" />
        </Suspense>
      </KeepAlive>
    </Transition>
  </RouterView>
</template>
```

所以"过渡没生效"和"保活没生效"的第一现场是同一个：**`<Transition>` 与 `<KeepAlive>` 是否真的被装上了**。三个入口只是装载方式，不装就都没有。

同一份文档还有一句和保活直接相关的限制：**如果你在页面组件里启用 `<Transition>`，请确保该页面只有一个根元素**（过渡那一页的 important 块把这条说得更完整，118 已引）。

## `keepalive` 的三种写法与一条专用语法

官方在 `app/pages/` 那一页给了完整说法：

> 如果你在 `definePageMeta` 里设置 `keepalive: true`，**Nuxt 会自动把你的页面包进 Vue 的 `<KeepAlive>` 组件**。这可能有用，例如在**一个拥有动态子路由的父路由**里，如果你想跨路由变化保留页面状态。

> **如果你的目标是保留父路由的状态，请使用这种语法：`<NuxtPage keepalive />`。** 你也可以设置要传给 `<KeepAlive>` 的 props。

> **你可以在你的 `nuxt.config` 里为这个属性设置一个默认值。**

| 想要的效果 | 官方给的写法 |
| --- | --- |
| 某个页面跨导航保留状态 | `definePageMeta({ keepalive: true })` |
| **父路由**下那些动态子页保留父状态 | **`<NuxtPage keepalive />`**（写在父页面的出口上） |
| 细粒度控制（谁不缓存） | `keepalive: { exclude: ['modal'] }` |
| 全站默认保活 | 在 `nuxt.config` 里设默认值 |

`definePageMeta` 的参数表也把类型与用途对上了：`keepalive` 类型是 `boolean | KeepAliveProps`，"**当你想在跨路由变化时保留页面状态就设为 `true`，或者用 `KeepAliveProps` 做细粒度控制。**"

官方的示例段把三件事放在一起演示（它的说明是：`key` 可以是一个返回值的函数；`keepalive` 属性使得在多个组件之间切换时 `<modal>` 组件不被缓存；并添加 `pageType` 作为自定义属性）：

```vue [app/pages/some-page.vue]
<script setup lang="ts">
definePageMeta({
  key: route => route.fullPath,

  keepalive: {
    exclude: ['modal'],
  },

  pageType: 'Checkout',
})
</script>
```

`key` 与 `keepalive` 挨着写不是巧合——**`key` 决定"要不要重建"，`keepalive` 决定"重建时留不留状态"**（092、116）。想验证保活真的生效，最直接的观察点是：换页回来时组件里的 `ref` 值还在不在；而一旦 `key` 随路径变化，组件被重建，再谈保活就没有意义了。

## 第四个入口：View Transitions（实验特性）

`definePageMeta` 里还有一个独立的 `viewTransition`，官方标注是："**实验特性，只有当你在 `nuxt.config` 文件里启用它时才可用。为当前页面启用/禁用 View Transitions。如果设为 `true`，当用户浏览器匹配 `prefers-reduced-motion: reduce` 时 Nuxt 不会应用该过渡（推荐做法）。如果设为 `always`，Nuxt 将始终应用该过渡。**"

类型是 `boolean | 'always' | ViewTransitionPageOptions`，对象形态官方列了四项：

| 字段 | 官方说明 |
| --- | --- |
| `enabled` | `boolean \| 'always'` — 启用/禁用该过渡 |
| `types` | `string[] \| (to, from) => string[]` — 应用于**涉及本页面的任何**过渡的类型 |
| `toTypes` | 只在**导航进入**这个页面时应用的类型 |
| `fromTypes` | 只在**从这个页面导航出去**时应用的类型 |

它与上一篇那套 CSS 过渡是两个体系：CSS/JS 钩子走 Vue 的 `<Transition>`，View Transitions 走浏览器那套 API。两者同时开着时会叠加，所以官方在这个键上仍要求"先在 `nuxt.config` 里启用"这一道总开关。

## 一条分配建议

按"作用范围从小到大"来分配入口，冲突就消失了：

| 层 | 建议放什么 |
| --- | --- |
| `nuxt.config` | 只放**是否需要**（比如 `pageTransition: false` 全局关掉，或全站统一的一个默认 name） |
| `<NuxtPage>` 出口 | 需要给某个出口统一控制时**只用这一处**（父路由保活就用 `<NuxtPage keepalive />`） |
| `definePageMeta` | 单页差异（换 name、换布局过渡、单独禁用、`keepalive` 细粒度 exclude） |
| 内联 `middleware` | 只在需要按 `to`/`from` 动态决定过渡名时用（118 那个方向判断） |

**同一层不要既在②又在③写同一个属性**——官方没写优先级的地方，就是未来要实测的地方。

## 自查

1. 官方那句话给的三条入口分别是什么？各自的作用范围怎么划分？
2. 官方明确写出的一条覆盖关系是哪两个入口之间？另一对为什么需要实测？给出你的实测步骤。
3. `<Transition>` 与 `<KeepAlive>` 的默认状态是什么？这对排查顺序意味着什么？
4. 想保住"父路由"的状态，官方给的专用写法是什么？
5. `key` 与 `keepalive` 各自控制什么？为什么说 `key` 随路径变化时谈保活没意义？
6. `viewTransition` 的 `true` 与 `'always'` 差别在哪？它和 `pageTransition` 是同一套机制吗？

## 下一篇

120 讲 `useRoute()`：官方为什么把 `import { useRoute } from 'vue-router'` 标成错误、Nuxt 版包装到底多做了什么，以及为什么它只该在组件 setup 或插件里用。

## 本文依据

- `https://nuxt.com/docs/4.x/api/components/nuxt-page`
  - 原文："By default, Nuxt does not enable `<Transition>` and `<KeepAlive>`. You can enable them in the nuxt.config file or by setting the `transition` and `keepalive` properties on `<NuxtPage>`. If you want to define a specific page, you can set it in `definePageMeta` in the page component."
  - 原文（结构）：`<RouterView v-slot="{ Component }">` / `<!-- Optional, when using transitions --> <Transition>` / `<!-- Optional, when using keep-alive --> <KeepAlive>` / `<Suspense>` / `<component :is="Component" />`
  - 原文（warning）："If you enable `<Transition>` in your page component, ensure that the page has a single root element."
  - 原文（Props）："`transition`: define global transitions for all pages rendered with the `NuxtPage` component."，type `boolean` or `TransitionProps`；"`keepalive`: control state preservation of pages rendered with the `NuxtPage` component."，type `boolean` or `KeepAliveProps`；"`pageKey`: control when the `NuxtPage` component is re-rendered."
- `https://nuxt.com/docs/4.x/directory-structure/app/pages`
  - 原文（`keepalive`）："Nuxt will automatically wrap your page in the Vue `<KeepAlive>` component if you set `keepalive: true` in your `definePageMeta`. This might be useful to do, for example, in a parent route that has dynamic child routes, if you want to preserve page state across route changes."；"When your goal is to preserve state for parent routes use this syntax: `<NuxtPage keepalive />`. You can also set props to be passed to `<KeepAlive>` (see a full list)."；"You can set a default value for this property in your `nuxt.config`."
  - 原文（单根）："Pages **must have a single root element** to allow route transitions between pages. HTML comments are considered elements as well."
- `https://nuxt.com/docs/4.x/api/utils/define-page-meta`
  - 原文（`keepalive`）："Set to `true` when you want to preserve page state across route changes or use the `KeepAliveProps` for a fine-grained control."；Type `boolean | KeepAliveProps`
  - 原文（`key`）："Set `key` value when you need more control over when the `<NuxtPage>` component is re-rendered."；Type `false | string | ((route: RouteLocationNormalizedLoaded) => string)`
  - 原文（`viewTransition`）："**Experimental feature, only available when enabled in your nuxt.config file.** Enable/disable View Transitions for the current page. If set to true, Nuxt will not apply the transition if the users browser matches `prefers-reduced-motion: reduce` (recommended). If set to `always`, Nuxt will always apply the transition."；Type `boolean | 'always' | ViewTransitionPageOptions`；对象字段 `enabled`: `boolean | 'always'`、`types`: `string[] | (to, from) => string[]` "types applied to any transition involving this page"、`toTypes` "types applied only when navigating **to** this page"、`fromTypes` "types applied only when navigating **from** this page"
  - 原文（Example - Basic Usage）："The example below demonstrates: how `key` can be a function that returns a value; how `keepalive` property makes sure that the `<modal>` component is not cached when switching between multiple components; adding `pageType` as a custom property" + `definePageMeta({ key: route => route.fullPath, keepalive: { exclude: ['modal'] }, pageType: 'Checkout' })`
- `https://nuxt.com/docs/4.x/getting-started/transitions`
  - 原文（override）："To override the global transition property, use the `definePageMeta` to define page or layout transitions for a single Nuxt page and override any page or layout transitions that are defined globally in `nuxt.config` file."
  - 对照用（见 118）：`app.pageTransition` / `app.layoutTransition` 的全局写法与 `mode: 'out-in' // default`；（note）"If you are changing layouts as well as pages, the page transition you set here will not run."；（warning）"If you change the `name` property, you also have to rename the CSS classes accordingly."
