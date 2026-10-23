---
url: /posts/010520266c6ab44654b0811fb6dd6dfe/
title: 378 - 延迟水合策略：defineLazyHydrationComponent 实战
date: 2027-02-15T09:00:00+08:00
updated: 2027-02-15T09:00:00+08:00
author: cmdragon

summary:
  摘要：377 的 ClientOnly/Fallback 都在"要不要服务端渲染"上想办法；延迟水合是第三条路——服务端照常把 HTML 渲染出来，只是把**水合**推迟到一个更划算的时刻，专门削非关键组件的首屏 JS 成本。`defineLazyHydrationComponent(strategy, source)` 是个编译器宏，七种策略各有对应的 `hydrate-*` 属性：`visible`（IntersectionObserver，可传 rootMargin）、`idle`（浏览器空闲或超时毫秒）、`interaction`（点击/hover，默认 pointerenter/click/focus）、`mediaQuery`、`if`（布尔 ref）、`time`（延时毫秒）、`never`（永不水合）。所有延迟组件在水合后发 `@hydrated`。一条硬提醒：宏参数别用外部变量，否则编译器认不出。

categories:
  - 前端开发

tags:
  - nuxt4
  - 延迟水合
  - defineLazyHydrationComponent
  - 性能
  - 编译器宏
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


373 说水合是"浏览器再把 JS 跑一遍挂事件"。对首屏看不见、又不急着交互的组件（页脚、轮播、折叠区），这一刻就跑它的水合是浪费——它抢关键渲染路径的 CPU。延迟水合就是把这部分开销往后挪。

## 它延的是"水合"，不是"渲染"

官方定位：

> `defineLazyHydrationComponent` is a compiler macro that helps you create a component with a specific lazy hydration strategy. Lazy hydration defers hydration until components become visible or until the browser has completed more critical tasks. This can significantly reduce the initial performance cost, especially for non-essential components.

三关键词：**编译器宏**、**延的是 hydration**、**为非关键组件削首屏成本**。注意它和 376/377 的差别：ClientOnly 让服务端**不渲染**这块；延迟水合是服务端**照常渲染出 HTML**，只是把"何时水合"推后——所以首屏仍是完整内容，SEO 也不受影响。

## 七种策略

宏签名 `defineLazyHydrationComponent(strategy, source)`，`strategy` 类型是 `'visible' | 'idle' | 'interaction' | 'mediaQuery' | 'if' | 'time' | 'never'`（必填），`source` 是 `() => Promise<Component>`（必填，一个动态 import）。策略表照文档：

| 策略 | 何时水合 | 配套属性 |
| --- | --- | --- |
| `visible` | 组件进入视口时 | `hydrate-on-visible`（IntersectionObserver 选项对象，如 `{ rootMargin: '100px' }`） |
| `idle` | 浏览器空闲或超时后 | `hydrate-on-idle`（正数=最大超时毫秒，如 `2000`） |
| `interaction` | 用户交互后 | `hydrate-on-interaction`（事件名或列表；不传默认 `pointerenter`、`click`、`focus`） |
| `mediaQuery` | 命中媒体查询时 | `hydrate-on-media-query`（查询串，如 `"(min-width: 768px)"`） |
| `if` | 布尔条件为真时 | `hydrate-when`（一个布尔 ref） |
| `time` | 指定延时之后 | `hydrate-after`（毫秒，如 `1000`） |
| `never` | 永不水合 | （无） |

`idle` 官方点明适合"要尽快加载、但别挡关键渲染路径"的组件；`time` 适合"能等一小段时间"的；`if` 适合"不一定需要水合"的；`never` 让 Vue 永不水合该组件。

底层：`visible`/`idle`/`interaction`/`mediaQuery` 官方注明分别走 Vue 内置的 `hydrateOnVisible`/`hydrateOnIdle`/`hydrateOnInteraction`/`hydrateOnMediaQuery` 策略。

## 用 visible 举例

```vue
<script setup lang="ts">
const LazyHydrationMyComponent = defineLazyHydrationComponent(
  'visible',
  () => import('./components/MyComponent.vue'),
)
</script>

<template>
  <!-- 元素距离进入视口 100px 时触发水合 -->
  <LazyHydrationMyComponent :hydrate-on-visible="{ rootMargin: '100px' }" />
</template>
```

`hydrateOnVisible` 属性可选，传对象自定义 `IntersectionObserver` 行为。`idle`/`interaction` 等同理，换个策略名加对应 `hydrate-*` 属性即可。`if` 策略则把 `hydrate-when` 绑到一个 ref，改那个 ref 就是手动触发水合。

## 用 never 固定纯静态块

```vue
<script setup lang="ts">
const LazyHydrationMyComponent = defineLazyHydrationComponent(
  'never',
  () => import('./components/MyComponent.vue'),
)
</script>

<template>
  <!-- 这个组件永远不会被 Vue 水合 -->
  <LazyHydrationMyComponent />
</template>
```

内容型站点上这个很省：HTML 服务端出、但 JS 永不执行。（381 会看到静态站配方大量用它。）

## 听水合事件

所有延迟水合组件在水合完成时发 `@hydrated`：

```vue
<LazyHydrationMyComponent
  :hydrate-on-visible="{ rootMargin: '100px' }"
  @hydrated="onHydrated"
/>
```

## 一条会静默失效的坑

`::warning` 说得很硬：为了让编译器正确识别这个宏，**别用外部变量**：

```vue
<script setup lang="ts">
const strategy = 'visible'
const source = () => import('./components/MyComponent.vue')
const LazyHydrationMyComponent = defineLazyHydrationComponent(strategy, source) // 这样宏认不出来
</script>
```

策略字符串和 import 工厂都得**内联**写进宏调用，不能提成变量——因为它是编译器宏，在编译期静态解析参数，变量它看不到。

## 自查

- 延迟水合＝服务端照常渲染 HTML，只把**水合**推后；和 ClientOnly（不渲染）路线不同，首屏与 SEO 不受影响。
- `defineLazyHydrationComponent` 是编译器宏，参数必须内联（提成变量编译期认不出）。
- 七策略：visible/idle/interaction/mediaQuery/if/time/never，各有 `hydrate-*` 属性（`interaction` 默认 pointerenter/click/focus）。
- 全部延迟组件水合后发 `@hydrated`；`never` 让 Vue 永不水合，静态站常用。

## 下一篇

下一篇 379 转入预渲染：`nuxt generate` / `nitro.static` 用 Nitro 爬虫在构建期把页面拼成静态 HTML，`prerender.routes`、`ignore` 与 `crawlLinks` 怎么手动指定与发现。
