---
url: /posts/85f85a38cd98a3036f8486907ddd747b/
title: viewTransition 实验特性
date: 2027-02-17T09:00:00+08:00
updated: 2027-02-17T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：383 的页面动画走的是 Vue `<Transition>`；Nuxt 还接了另一条更现代的路——浏览器原生 View Transitions API，用 `experimental.viewTransition` 开启。三档值：`false`、`true`（用户浏览器匹配 `prefers-reduced-motion: reduce` 时不应用，官方推荐）、`'always'`（总是应用，是否尊重用户偏好你自己管）。默认对所有页开启，可 `app.viewTransition: false` 全局关再逐页开；`definePageMeta` 的 `viewTransition` 能逐页覆盖（但只有开了实验选项才生效）。v4.4 起支持 view transition types（`types`/`toTypes`/`fromTypes`，可按前进/后退给不同动画、支持函数动态判定）。一条已知问题：它会在水合期间冻结 DOM 更新，页面 setup 里取数的要慎用。

categories:
  - 前端开发

tags:
  - nuxt4
  - viewTransition
  - 实验特性
  - 页面动画
  - 无障碍
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

383 是 Vue 层的过渡。384 讲另一条路线：让浏览器自己接管切页动画——View Transitions API。experimental-features 那页给它一句话："Enables View Transition API integration with client-side router."（把 View Transition API 接到客户端路由上）。

## 先开开关

在配置里打开：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    viewTransition: true,
  },
})
```

它和其它实验特性一样：**没开这个 `experimental.viewTransition`，逐页覆盖都不起作用**（官方 `::warning` 写明 per-page 覆盖只在实验选项开启后生效）。

## 三档值

transitions 那页把取值讲清：可能值是 `false`、`true`、`'always'`。

- `true`：浏览器匹配 `prefers-reduced-motion: reduce` 时**不**应用过渡——官方标注这是"recommended"（尊重用户"减少动效"的系统偏好）。
- `'always'`：总是应用，是否尊重用户偏好由你自己负责。

## 全局默认与逐页开关

默认对所有页面启用。想反过来（全局关、个别页开）：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  app: {
    // 全局关，再逐页开
    viewTransition: false,
  },
})
```

单页用 `definePageMeta` 覆盖：

```vue
<script setup lang="ts">
definePageMeta({
  viewTransition: false,
})
</script>
```

注意这里有**两个** `viewTransition`：`experimental.viewTransition` 是总开关（`false|true|'always'`），`app.viewTransition` / `definePageMeta({ viewTransition })` 是按页/全局的布尔启用位。

## Types：按导航方向给不同动画（v4.4）

View transition types 让你按导航类型给不同 CSS 动画——做前进/后退不对称的效果。类型设在 `ViewTransition` 对象上，CSS 里用 `:active-view-transition-type()` 选择器定位。

全局默认：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  app: {
    viewTransition: {
      enabled: true,
      types: ['slide'],
    },
  },
})
```

逐页更细，支持 `types` / `toTypes` / `fromTypes`（进/出分别用），还能用函数按路由动态判定：

```vue
<script setup lang="ts">
definePageMeta({
  viewTransition: {
    enabled: true,
    toTypes: (to, from) => {
      return Number(to.params.id) > Number(from.params.id)
        ? ['slide-left']
        : ['slide-right']
    },
  },
})
</script>
```

官方 `::note` 限定：`types`/`toTypes`/`fromTypes` 的**函数形态只在 `definePageMeta` 里有效**，`nuxt.config.ts` 里只支持静态 `string[]`。运行期还能在 `page:view-transition:start` 钩子里读写 `transition.types`（371 表里那个钩子）。

## 和 Vue 过渡二选一

如果你已经用 View Transitions，就不需要 Vue 那套 `pageTransition` 重复做一遍。官方给的做法：建一个 `~/middleware/disable-vue-transitions.global.ts`，浏览器支持原生 API 时把 Vue 过渡关掉：

```ts
export default defineNuxtRouteMiddleware((to) => {
  if (import.meta.server || !document.startViewTransition) {
    return
  }
  to.meta.pageTransition = false
  to.meta.layoutTransition = false
})
```

## 一条必须先掂量的已知问题

Known Issues 官方自己先劝退了取数场景：

> If you perform data fetching within your page setup functions, you may wish to reconsider using this feature for the moment. (By design, View Transitions completely freeze DOM updates whilst they are taking place.) We're looking at restricting the View Transition to the final moments before `<Suspense>` resolves...

View Transitions **在进行期间会完全冻结 DOM 更新**。如果你在页面 setup 里取数（372 那套 `useFetch`/`useAsyncData` 靠 Suspense 等待），过渡期间这些更新会被冻住。官方说正在考虑把 View Transition 限制到 `<Suspense>` 解析前的最后时刻，但在那之前，页面里做取数的要谨慎采用。

## 自查

- `experimental.viewTransition` 是总开关：`false`/`true`（尊重 prefers-reduced-motion，推荐）/`'always'`（自己管偏好）。
- 默认全页启用；`app.viewTransition: false` 全局关、`definePageMeta({ viewTransition })` 逐页——都要先开实验选项才生效。
- v4.4 types：`types`/`toTypes`/`fromTypes` 做前进后退不对称动画，函数形态只在 definePageMeta 有效；`page:view-transition:start` 可运行时读写 types。
- 和 Vue `pageTransition` 二选一（用全局中间件在有原生 API 时关掉 Vue 过渡）。
- 已知问题：View Transitions 期间冻结 DOM 更新，页面 setup 取数的慎用。

## 下一篇

下一篇 385 讲另外两个导航/首屏体验开关：`navigationRepaint`（导航前等一帧让浏览器重绘、降 INP）和 `ssrStreaming`（先发抖壳流式渲染、降 TTFB），两者优化的是不同指标。

## 本文依据

- experimental-features `## viewTransition`（"Enables View Transition API integration with client-side router"、布尔与对象 `{ enabled, types }` 两形态）；transitions 页 View Transitions 节：三档值与 `true` 尊重 `prefers-reduced-motion`、`app.viewTransition` 全局默认、`definePageMeta` 逐页覆盖的 `::warning`、View Transition Types（v4.4、types/toTypes/fromTypes、函数只在 definePageMeta、`:active-view-transition-type()`、`page:view-transition:start`）、disable-vue-transitions 中间件、Known Issues "completely freeze DOM updates"（`guide/going-further/experimental-features`、`getting-started/transitions`）。
