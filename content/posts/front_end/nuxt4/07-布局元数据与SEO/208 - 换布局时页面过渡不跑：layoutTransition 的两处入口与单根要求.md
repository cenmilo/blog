---
url: /posts/9820eb1e2257bf93b4ecb26c3bcdbb2d/
title: 换布局时页面过渡不跑：layoutTransition 的两处入口与单根要求
date: 2026-12-20T10:00:00+08:00
updated: 2026-12-20T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：118 已经把两个全局默认、四组 CSS 类名与"同时换布局时页面过渡不跑"这条讲过了，这一篇只补布局那一侧的细节，并把官方过渡页里跟布局有关的原话逐条落地。第一句就是那条规则的原文："If you are changing layouts as well as pages, the page transition you set here will not run. Instead, you should set a layout transition."——它是 note，不是脚注，我一开始把它当例外读，其实是默认行为。第二句是硬约束，而且是 warning 级别的："Because Nuxt uses Vue's `<Transition>` component, a page or layout you want to animate must have a **single root element**. A page or layout with multiple root elements (a fragment) cannot be animated, so the transition will not run and navigating between routes may error. Nuxt warns about this in development."——**注意后果不止"动画没有"，还包括"导航可能报错"**。第三句是入口：`app.layoutTransition`（配置节原话 "Default values for layout transitions. This can be overridden with `definePageMeta` on an individual page. Only JSON-serializable values are allowed."，类型 `boolean | TransitionProps`）与页面里的 `definePageMeta({ layout: 'orange', layoutTransition: { name: 'slide-in' } })`（官方示例把两者写在同一页上）。另外官方只在 `pageTransition` 的示例里演示过 JS 钩子，`layoutTransition` 能不能用同一套钩子，文档没写；而想把内置 Vue 过渡关掉以让位给原生 View Transitions，文档给的是一个**全局中间件**配方，里面同时改 `to.meta.pageTransition` 和 `to.meta.layoutTransition`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 过渡动画
  - layoutTransition
  - View Transitions
  - 布局

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先解释两个词，避免后面读不下去：

- **过渡（transition）**：页面或外壳切换时那段渐变/位移动画，Vue 用一个 `<Transition>` 组件实现，靠 CSS 类名在进出两个时刻改样式。
- **View Transitions**：浏览器自己新提供的一套跨页面切换接口，不需要手写 CSS 类名，能力上能衔接不同页面上的两个元素。

## 那条规则的原话长什么样

官方过渡页在讲完 `app.pageTransition` 之后立刻挂了这条 note：

> If you are changing layouts as well as pages, the page transition you set here will not run. Instead, you should set a [layout transition].

我把"改布局时页面过渡不跑"当成一个 bug 排查过半天，其实它是写在这里的默认行为。为什么这么设计，文档在别处给过一句间接解释：182 抄过那句要求布局只有一个根元素的理由——"to allow Nuxt to apply transitions **between layout changes**"。也就是说**同一时刻只有一层过渡在跑**：换外壳时由 `layoutTransition` 负责，不换外壳时才轮到 `pageTransition`。

顺带把另一条更容易踩的补在这儿（119 讲过入口，这条是它的限制）：

> Remember, this page transition cannot be overridden with `definePageMeta` on individual pages.

它指的是 `<NuxtPage :transition="{ name: 'bounce', mode: 'out-in' }" />` 这种在出口上写的过渡——**用这条路设的过渡，页面里的 `definePageMeta` 覆盖不了**。所以别指望"全局用 prop、个别页用 meta"这种组合。

## 硬约束：单根元素，后果不只是没动画

整节最重要的一条，原文（warning）：

> Because Nuxt uses Vue's `<Transition>` component, a page or layout you want to animate must have a **single root element**. A page or layout with multiple root elements (a fragment) cannot be animated, so the transition will not run and navigating between routes may error. Nuxt warns about this in development.

三个层次都在里面：

1. 要求对象**同时**是 page 和 layout（不是我以为的只管页面）。
2. 违反以后先"动画不跑"，再"导航可能报错"。
3. 提示只在开发环境出现——生产构建里你可能只剩下"点了没反应"。

文档给的修法也是一句话的事："Wrap the template in a single root element (for example a `<div>`)."

官方那套布局过渡示例正好把这一点演得很干净，两个布局都是 `<div>` 包一切、`<slot />` 放在里面：

```vue [app/layouts/orange.vue]
<template>
  <div>
    <pre>orange layout</pre>
    <slot />
  </div>
</template>

<style scoped>
div {
  background-color: #eebb90;
  padding: 20px;
  height: 100vh;
}
</style>
```

这里有个我原本会写错的地方：布局里的样式官方用的是 `<style scoped>`，而**过渡用的 CSS 类名写在 `app.vue`**（不写 scoped）。同节 `app.vue` 那段就是这样：

```vue [app/app.vue]
<template>
  <NuxtLayout>
    <NuxtPage />
  </NuxtLayout>
</template>

<style>
.layout-enter-active,
.layout-leave-active {
  transition: all 0.4s;
}
.layout-enter-from,
.layout-leave-to {
  filter: grayscale(1);
}
</style>
```

## 两处入口，与官方那条自定义示例

配置节里 `app.layoutTransition` 与 `app.pageTransition` 的说明一模一样，只有对象不同：

> Default values for layout transitions.
>
> This can be overridden with `definePageMeta` on an individual page. Only JSON-serializable values are allowed.

- **Type**: `boolean | TransitionProps`

那句 **Only JSON-serializable values are allowed** 是这一节里唯一解释"为什么函数形态的 `titleTemplate` 不能放 nuxt.config、而过渡里的函数也放不进去"的同一条理由——它和 190 里 `if (import.meta.server)` 那种"配置层是静态的"说法一致。

页面级覆盖，官方示例把布局和过渡写在同一个 `definePageMeta` 里：

```vue twoslash [pages/about.vue]
<script setup lang="ts">
definePageMeta({
  layout: 'orange',
  layoutTransition: {
    name: 'slide-in',
  },
})
</script>
```

原话给的定位是 "Similar to `pageTransition`, you can apply a custom `layoutTransition` to the page component using `definePageMeta`." ——注意它说的是"用 `definePageMeta` 应用到一个页面上"，也就是**过渡的名字属于页面**，而跑不跑取决于这次导航是否换外壳。

关掉的方法两边对称（Disable Transitions 一节）：页面里 `pageTransition: false, layoutTransition: false`，或者 `nuxt.config` 里两个都设 `false`。改 `name` 的连带义务是同节那条 warning："If you change the `name` property, you also have to rename the CSS classes accordingly."

## JS 钩子：官方只在页面那一侧演示过

过渡页的 JavaScript Hooks 一节，示例写的键是 `pageTransition`：

```vue twoslash [pages/some-page.vue]
<script setup lang="ts">
definePageMeta({
  pageTransition: {
    name: 'custom-flip',
    mode: 'out-in',
    onBeforeEnter: (el) => {
      console.log('Before enter...')
    },
    onEnter: (el, done) => {},
    onAfterEnter: (el) => {},
  },
})
</script>
```

`layoutTransition` 能不能用同一套钩子，**文档没写**。同节那句定位给了线索但不够定论："Both `pageTransition` and `layoutTransition` keys accept [`TransitionProps`](...) as JSON serializable values where you can pass the `name`, `mode` and other valid transition-props"——`TransitionProps` 是 Vue 那边包含钩子的类型，但"JSON serializable values"又和函形态钩子冲突（这正是 118 里我记下的那个矛盾）。我的处理是：真要用 GSAP 一类做外壳切换动画时，先在 `layoutTransition` 上放一个 `onBeforeEnter` 打印一次，跑不跑得只有实测能答。

另外 `mode` 默认值是 `out-in`（示例两处都注着 `mode: 'out-in', // default`）。

## 让位给原生 View Transitions：官方给的配方是全局中间件

这一节我要的东西很具体——浏览器支持原生 API 时把内置 Vue 过渡关掉。官方原文：

> If you are also using Vue transitions like `pageTransition` and `layoutTransition` (see above) to achieve the same result as the new View Transitions API, then you may wish to *disable* Vue transitions if the user's browser supports the newer, native web API. You can do this by creating `~/middleware/disable-vue-transitions.global.ts` with the following contents:

```ts
export default defineNuxtRouteMiddleware((to) => {
  if (import.meta.server || !document.startViewTransition) {
    return
  }

  // Disable built-in Vue transitions
  to.meta.pageTransition = false
  to.meta.layoutTransition = false
})
```

这段同时回答了几个问题：**两个 meta 键要一起改**（只改一个会剩另一半在跑）、判据是 `document.startViewTransition` 这个浏览器能力、而且**服务端直接 return**（`import.meta.server` 那一支）。文件名带 `.global` 是全局中间件的注册方式（128 讲过）。

原生那一侧的开关是 `experimental.viewTransition`，取值三种："The possible values are: `false`, `true`, or `'always'`."，官方解释是：

> If set to true, Nuxt will not apply transitions if the user's browser matches `prefers-reduced-motion: reduce` (recommended). If set to `always`, Nuxt will always apply the transition and it is up to you to respect the user's preference.

默认全局开启、页面级用 `definePageMeta({ viewTransition: false })` 覆盖；`app.viewTransition` 那条配置说明里也写死了前提："This only has an effect when **experimental** support for View Transitions is enabled in your nuxt.config file."

Known Issues 只有一条，但对做布局动画的人来说正好是最该知道的：**在页面 setup 里取数的场景别用**——"By design, View Transitions completely freeze DOM updates whilst they are taking place."（官方接着说他们正考虑把过渡限制在 `<Suspense>` 解析完前的最后一刻，在那之前建议评估自己是否属于这种情形。）

## 自查

1. 换布局时页面过渡为什么不跑，原话是哪句？（答："If you are changing layouts as well as pages, the page transition you set here will not run. Instead, you should set a layout transition."）
2. 多根元素的外壳开过渡，后果是什么？（答：过渡不跑，而且"navigating between routes may error"，开发环境才有警告。）
3. 官方用来在支持原生 API 时关掉内置过渡的文件是什么、改哪两个键？（答：`~/middleware/disable-vue-transitions.global.ts`，同时设 `to.meta.pageTransition = false` 与 `to.meta.layoutTransition = false`。）

## 下一篇

209 - 综合演练：一篇文章页要写齐哪些元数据

## 本文依据

- `getting-started/transitions`：note "If you are changing layouts as well as pages, the page transition you set here will not run."；important 单根元素整段（含 "(a fragment) cannot be animated"、"the transition will not run and navigating between routes may error"、"Nuxt warns about this in development"、"Wrap the template in a single root element (for example a `<div>`)."）；Layout Transitions 的 `app.layoutTransition` 配置块与 `app/app.vue` CSS 示例、`app/layouts/orange.vue` 示例（`<style scoped>`）；"Similar to `pageTransition`, you can apply a custom `layoutTransition` to the page component using `definePageMeta`." 与含 `layout: 'orange'` 的示例；Global Settings 段 "Both `pageTransition` and `layoutTransition` keys accept `TransitionProps` as JSON serializable values..."、warning "If you change the `name` property, you also have to rename the CSS classes accordingly."；Disable Transitions 两处写法；JavaScript Hooks 示例（`onBeforeEnter`/`onEnter`/`onAfterEnter`，键为 `pageTransition`）；Transition with NuxtPage 与 note "Remember, this page transition cannot be overridden with `definePageMeta` on individual pages."；View Transitions 一节（`experimental.viewTransition: true`、三种取值、`prefers-reduced-motion`、`app.viewTransition: false` 全局关、`definePageMeta({ viewTransition: false })` 覆盖、per-page 函数形态、`page:view-transition:start`、disable-vue-transitions 全局中间件全文）；Known Issues 那条冻结 DOM 的原话。
- `api/nuxt-config`（app 节）：`layoutTransition` 与 `pageTransition` 两句说明（"Default values for layout transitions."、"This can be overridden with `definePageMeta` on an individual page. Only JSON-serializable values are allowed."、Type `boolean | TransitionProps`）；`viewTransition` 节 "This only has an effect when **experimental** support for View Transitions is enabled in your nuxt.config file."
- `directory-structure/app/layouts`（182 已引）：单根要求的理由句 "to allow Nuxt to apply transitions **between layout changes**"。
- 与 118/119 的分工：两个全局默认与四组 CSS 类名、`<NuxtPage>` 上的过渡与保活入口在 118/119 讲；本篇只处理布局侧与原生 View Transitions 的互斥关系。
