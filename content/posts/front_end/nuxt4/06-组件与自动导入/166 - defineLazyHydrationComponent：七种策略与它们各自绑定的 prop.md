---
url: /posts/549e51b1609418345f1d5e5323e79fae/
title: 166 - defineLazyHydrationComponent：七种策略与它们各自绑定的 prop
date: 2026-12-06T10:00:00+08:00
updated: 2026-12-06T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：官方定位句："**`defineLazyHydrationComponent` is a compiler macro that helps you create a component with a specific lazy hydration strategy. Lazy hydration defers hydration until components become visible or until the browser has completed more critical tasks. This can significantly reduce the initial performance cost, especially for non-essential components.**"。它解决的正是 161 留下的那道缺口——`Lazy` 前只管分包："**Lazy components are great for controlling the chunk sizes in your app, but they don't always enhance runtime performance, as they still load eagerly unless conditionally rendered.**"，而"**In order to optimize your app, you may want to delay the hydration of some components until they're visible, or until the browser is done with more important tasks. Nuxt supports this using lazy (or delayed) hydration, allowing you to control when components become interactive.**"。签名是两个必需参数：`strategy` 的类型是 `'visible' | 'idle' | 'interaction' | 'mediaQuery' | 'if' | 'time' | 'never'`（Required: true），`source` 的类型是 `() => Promise<Component>`（Required: true）。**最容易被写错的是策略名与模板 prop 名的对应关系**，官方七个小节各给一对：`'visible'` → `:hydrate-on-visible="{ rootMargin: '100px' }"`（注释"Hydration will be triggered when the element(s) is **100px away from entering the viewport**"，prop 可选、可传 IntersectionObserver 选项）、`'idle'` → `:hydrate-on-idle="2000"`（"Hydration will be triggered when the browser is idle **or after 2000ms**"，可传正数作最大超时）、`'interaction'` → `hydrate-on-interaction="mouseover"`（不传事件时"**it defaults to hydrating on `pointerenter`, `click`, and `focus`**"）、`'mediaQuery'` → `hydrate-on-media-query="(min-width: 768px)"`、`'time'` → `:hydrate-after="1000"`、`'if'` → `:hydrate-when="isReady"`、`'never'` → 不写 prop（"This component will **never be hydrated by Vue**"）。四个策略官方另注了底层实现（visible/idle/interaction/mediaQuery 分别用 Vue 内置的 `hydrateOnVisible`/`hydrateOnIdle`/`hydrateOnInteraction`/`hydrateOnMediaQuery`）。还有一个事件："**All delayed hydration components emit a `@hydrated` event when they are hydrated.**"

categories:
  - 前端开发

tags:
  - nuxt4
  - 延迟水合
  - 性能
  - 编译器宏
  - Vapor

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


161 结尾我记下一句缺口：`Lazy` 前缀只保证"切成一块"，不保证"晚点加载"。真正控制"什么时候变成可交互"的是这一篇的宏——它把七种触发条件做成一个策略参数，而**每种策略在模板里绑定的 prop 名字都不一样**，这是最容易写错的地方。

## 它解决的是哪个问题

官方第一段就把动机写清了：

> `defineLazyHydrationComponent` is a **compiler macro** that helps you create a component with a specific lazy hydration strategy. Lazy hydration **defers hydration until components become visible or until the browser has completed more critical tasks**. This can significantly reduce the initial performance cost, especially for **non-essential components**.

它与 `Lazy` 前缀的分工，官方在组件目录页那节说得更直白（161 也引过）：

> Lazy components are great for **controlling the chunk sizes** in your app, but they **don't always enhance runtime performance**, as they still load eagerly unless conditionally rendered. ... In order to optimize your app, you may want to **delay the hydration** of some components until they're visible, or until the browser is done with more important tasks.

两句话的对照：

| | `Lazy` 前缀（161） | 延迟水合（这一篇） |
| --- | --- | --- |
| 管什么 | 代码什么时候下载 | 组件什么时候变成可交互 |
| 官方评价 | "great for controlling the chunk sizes" | "can significantly reduce the initial performance cost" |
| 触发方式 | 需要渲染条件（`v-if`）才生效 | 七种策略自己决定 |

## 调用形状：两个必需参数

```ts
const LazyHydrationMyComponent = defineLazyHydrationComponent(
  'visible',
  () => import('./components/MyComponent.vue'),
)
```

官方 Parameters 一节给的定义：

| 参数 | 类型 | 必需 |
| --- | --- | --- |
| `strategy` | `'visible' \| 'idle' \| 'interaction' \| 'mediaQuery' \| 'if' \| 'time' \| 'never'` | `true` |
| `source` | `() => Promise<Component>` | `true` |

策略七值的官方描述表：

| Strategy | Description（原文） |
| --- | --- |
| `visible` | Hydrates when the component becomes visible in the viewport. |
| `idle` | Hydrates when the browser is idle or after a delay. |
| `interaction` | Hydrates upon user interaction (e.g., click, hover). |
| `mediaQuery` | Hydrates when the specified media query condition is met. |
| `if` | Hydrates when a specified boolean condition is met. |
| `time` | Hydrates after a specified time delay. |
| `never` | Prevents Vue from hydrating the component. |

**注意第二参数必须是 `source` 形态**：官方类型写的是 `() => Promise<Component>`，也就是**动态 import 的函数**，不是组件本身。这一点与 167 那条"不能用外部变量"的限制合起来看，基本能确定这宏是靠读参数文本工作的——167 展开。

## 七种策略与各自的 prop

这一节是本篇真正的用处：**策略名与模板 prop 名不是一套拼写**。官方七个小节各给一对。

### `visible`

```vue
<script setup lang="ts">
const LazyHydrationMyComponent = defineLazyHydrationComponent(
  'visible',
  () => import('./components/MyComponent.vue'),
)
</script>

<template>
  <div>
    <!--
      Hydration will be triggered when
      the element(s) is 100px away from entering the viewport.
    -->
    <LazyHydrationMyComponent :hydrate-on-visible="{ rootMargin: '100px' }" />
  </div>
</template>
```

> The `hydrateOnVisible` prop is optional. **You can pass an object to customize the behavior of the `IntersectionObserver` under the hood.**

底层："Under the hood, this uses Vue's built-in `hydrateOnVisible` strategy."

### `idle`

```vue
<template>
  <div>
    <!-- Hydration will be triggered when the browser is idle or after 2000ms. -->
    <LazyHydrationMyComponent :hydrate-on-idle="2000" />
  </div>
</template>
```

> The `hydrateOnIdle` prop is optional. **You can pass a positive number to specify the maximum timeout.**
>
> Idle strategy is for components that can be hydrated when the browser is idle.

底层：`hydrateOnIdle`。官方对这一策略的定位句是"This is suitable if you need the component to load as soon as possible, **but not block the critical rendering path**."

### `interaction`

```vue
<template>
  <div>
    <!--
      Hydration will be triggered when
      the element(s) is hovered over by the pointer.
    -->
    <LazyHydrationMyComponent hydrate-on-interaction="mouseover" />
  </div>
</template>
```

> The `hydrateOnInteraction` prop is optional. **If you do not pass an event or a list of events, it defaults to hydrating on `pointerenter`, `click`, and `focus`.**

底层：`hydrateOnInteraction`。

### `mediaQuery`

```vue
<template>
  <div>
    <!--
      Hydration will be triggered when
      the window width is greater than or equal to 768px.
    -->
    <LazyHydrationMyComponent hydrate-on-media-query="(min-width: 768px)" />
  </div>
</template>
```

底层：`hydrateOnMediaQuery`。

### `time`

```vue
<template>
  <div>
    <!-- Hydration is triggered after 1000ms. -->
    <LazyHydrationMyComponent :hydrate-after="1000" />
  </div>
</template>
```

> Hydrates the component after a specified delay (in milliseconds).
>
> Time strategy is for components that can wait a specific amount of time.

**这一格是拼写陷阱**：策略叫 `time`，prop 却叫 `hydrate-after`（组件目录页那版写法也是 `:hydrate-after="2000"`，两处一致）。而且**它是七个里少数没有"under the hood 用 Vue 内置策略"注记的**。

### `if`

```vue
<script setup lang="ts">
const LazyHydrationMyComponent = defineLazyHydrationComponent(
  'if',
  () => import('./components/MyComponent.vue'),
)

const isReady = ref(false)

function myFunction () {
  // Trigger custom hydration strategy...
  isReady.value = true
}
</script>

<template>
  <div>
    <!-- Hydration is triggered when isReady becomes true. -->
    <LazyHydrationMyComponent :hydrate-when="isReady" />
  </div>
</template>
```

> If strategy is best for components that might not always need to be hydrated.

策略叫 `if`，prop 叫 `hydrate-when`——**第二个拼写陷阱**。

### `never`

```vue
<template>
  <div>
    <!-- This component will never be hydrated by Vue. -->
    <LazyHydrationMyComponent />
  </div>
</template>
```

**这一格最反直觉**：策略写 `'never'`，模板里**什么都不用传**。

### 对照表（我会贴在显示器边上那种）

| 策略 | 模板 prop | 参数形态 | 有底层 Vue 策略注记 |
| --- | --- | --- | --- |
| `visible` | `hydrate-on-visible` | IntersectionObserver 选项对象 | ✅ `hydrateOnVisible` |
| `idle` | `hydrate-on-idle` | 正数=最大超时 | ✅ `hydrateOnIdle` |
| `interaction` | `hydrate-on-interaction` | 事件名或列表，默认 `pointerenter`/`click`/`focus` | ✅ `hydrateOnInteraction` |
| `mediaQuery` | `hydrate-on-media-query` | 媒体查询字符串 | ✅ `hydrateOnMediaQuery` |
| `time` | `hydrate-after` | 毫秒数 | ❌ |
| `if` | `hydrate-when` | 布尔响应式值 | ❌ |
| `never` | （无） | — | ❌ |

## 知道什么时候真的水合完：`@hydrated`

> All delayed hydration components emit a `@hydrated` event when they are hydrated.

```vue
<script setup lang="ts">
const LazyHydrationMyComponent = defineLazyHydrationComponent(
  'visible',
  () => import('./components/MyComponent.vue'),
)

function onHydrate () {
  console.log('Component has been hydrated!')
}
</script>

<template>
  <div>
    <LazyHydrationMyComponent
      :hydrate-on-visible="{ rootMargin: '100px' }"
      @hydrated="onHydrated"
    />
  </div>
</template>
```

**这段官方示例自己有个前后不一致**：`<script setup>` 里定义的是 `onHydrate`，模板里绑的是 `@hydrated="onHydrated"`。直接照抄会得到一个未定义变量的问题——**抄的时候把两处名字对齐即可**（这是文档笔误，不是某种隐式别名）。

这个事件的实用价值在验证上：想知道某个策略到底什么时候触发，`console.log` 一句比猜时间可靠得多。

## 变量名为什么都叫 LazyHydration 开头

七个示例的变量名统一是 `LazyHydrationMyComponent`。这是**示例的命名选择，不是宏要求的语法**——宏返回的是一个普通组件变量，你叫什么都能用。但**它和 161 的 `Lazy` 前缀是两回事**：

| | 谁决定名字 | 是否影响加载 |
| --- | --- | --- |
| `Lazy` 前缀 | 你在模板里写的标签名，Nuxt 解析时匹配 | ✅ 触发动态导入 |
| `LazyHydrationXxx` 变量 | 你自己 `const` 出来的局部变量 | 由宏的策略参数决定 |

所以一个组件可以同时是懒加载且延迟水合，但那是**两件配置**，不是同一个前缀的两种说法。

## 与"模板 prop 版"的关系

组件目录页还给了另一种写法：不定义变量，直接给 `Lazy` 前缀组件加 hydrate prop——

```vue [app/pages/index.vue]
<template>
  <div>
    <LazyMyComponent hydrate-on-visible />
  </div>
</template>
```

两条路的策略集合是一致的（`hydrate-on-visible`/`hydrate-on-idle`/`hydrate-on-interaction`/`hydrate-on-media-query`/`hydrate-after`/`hydrate-when`/`hydrate-never`），差别在**谁来产生那个组件**：模板 prop 版依赖自动导入的 `Lazy` 前缀，宏版是你自己显式建一个组件变量。它们共享同一批限制（SFC、prop 写法、不能用于 `#components` 直接导入），这些限制与"prop 变更会立刻触发水合"那条一起，我放在 167 讲。

（另注：`nuxt-client` 的选择性水合是岛屿那套机制里的另一个东西，与这一篇的七种策略无关，见 171。）

## 自查

1. 官方给这个宏的定位句里，"延迟到什么条件"是用哪两个短语描述的？它适合哪类组件？
2. `Lazy` 前缀与延迟水合分别管哪件事？官方对前者的评价句是哪一句？
3. 两个必需参数的类型分别是什么？第二参数为什么不能直接传组件？
4. `time` 与 `if` 两个策略各自对应的模板 prop 名是什么？`never` 需要传 prop 吗？
5. 七个策略里有几个官方注明用了 Vue 内置策略？分别是哪几个？
6. `interaction` 不传事件时的默认三个是什么？`idle` 传数字表示什么？
7. `@hydrated` 那句官方定义是什么？它那段示例里有什么笔误，抄的时候要怎么处理？
8. 宏版与模板 prop 版的差别在哪一点上？它们共享的那批限制去哪一篇看？

## 下一篇

167 讲这套宏的两类限制：为什么策略与来源都不能用外部变量、为什么它只对 SFC 生效、以及"prop 一变就立刻水合"这条会让人意外（包括 `hydrate-never` 也会因此失效）。

## 本文依据

- `https://nuxt.com/docs/4.x/api/utils/define-lazy-hydration-component`
  - 原文（定位）："Define a lazy hydration component with a specific strategy."；"`defineLazyHydrationComponent` is a compiler macro that helps you create a component with a specific lazy hydration strategy. Lazy hydration defers hydration until components become visible or until the browser has completed more critical tasks. This can significantly reduce the initial performance cost, especially for non-essential components."
  - 原文（七个策略小节与示例）：`'visible'` + `:hydrate-on-visible="{ rootMargin: '100px' }"` 与注释 "Hydration will be triggered when the element(s) is 100px away from entering the viewport."、"The `hydrateOnVisible` prop is optional. You can pass an object to customize the behavior of the `IntersectionObserver` under the hood."、note "Under the hood, this uses Vue's built-in `hydrateOnVisible` strategy."；`'idle'` + `:hydrate-on-idle="2000"` 与 "Hydration will be triggered when the browser is idle or after 2000ms."、"The `hydrateOnIdle` prop is optional. You can pass a positive number to specify the maximum timeout."、"Idle strategy is for components that can be hydrated when the browser is idle."、"This is suitable if you need the component to load as soon as possible, but not block the critical rendering path."、note `hydrateOnIdle`；`'interaction'` + `hydrate-on-interaction="mouseover"` 与 "Hydration will be triggered when the element(s) is hovered over by the pointer."、"The `hydrateOnInteraction` prop is optional. If you do not pass an event or a list of events, it defaults to hydrating on `pointerenter`, `click`, and `focus`."、note `hydrateOnInteraction`；`'mediaQuery'` + `hydrate-on-media-query="(min-width: 768px)"` 与 "Hydration will be triggered when the window width is greater than or equal to 768px."、note `hydrateOnMediaQuery`；`'time'` + `:hydrate-after="1000"` 与 "Hydration is triggered after 1000ms."、"Hydrates the component after a specified delay (in milliseconds)."、"Time strategy is for components that can wait a specific amount of time."；`'if'` + `:hydrate-when="isReady"` 与 `const isReady = ref(false)`、`function myFunction () { // Trigger custom hydration strategy... \n isReady.value = true }`、"Hydration is triggered when isReady becomes true."、"If strategy is best for components that might not always need to be hydrated."；`'never'` + 无 prop 与 "This component will never be hydrated by Vue."、"Never hydrates the component."
  - 原文（事件）："All delayed hydration components emit a `@hydrated` event when they are hydrated." + 示例（`function onHydrate ()` 与 `@hydrated="onHydrated"` 两处名字不一致、`:hydrate-on-visible="{ rootMargin: '100px' }"`）
  - 原文（Parameters）：`strategy` — "**Type**: `'visible' | 'idle' | 'interaction' | 'mediaQuery' | 'if' | 'time' | 'never'`" / "**Required**: `true`" / 七行描述表；`source` — "**Type**: `() => Promise<Component>`" / "**Required**: `true`"
  - 原文（Source）：`packages/nuxt/src/components/plugins/lazy-hydration-macro-transform.ts`
- `https://nuxt.com/docs/4.x/directory-structure/app/components`
  - 原文（分工与模板 prop 版）："Lazy components are great for controlling the chunk sizes in your app, but they don't always enhance runtime performance, as they still load eagerly unless conditionally rendered. In real-world applications, some pages may include a lot of content and a lot of components, and most of the time not all of them need to be interactive as soon as the page is loaded. Having them all load eagerly can negatively impact performance."；"In order to optimize your app, you may want to delay the hydration of some components until they're visible, or until the browser is done with more important tasks. Nuxt supports this using lazy (or delayed) hydration, allowing you to control when components become interactive."；`<LazyMyComponent hydrate-on-visible />` 等六个策略小节与 "Under the hood..." 三条注记
- `https://nuxt.com/docs/4.x/guide/concepts/vuejs-development`
  - 对照用（见 164）：Vapor 组件不支持 Options API；`<script setup vapor>` 与本篇宏的兼容性官方未提及
