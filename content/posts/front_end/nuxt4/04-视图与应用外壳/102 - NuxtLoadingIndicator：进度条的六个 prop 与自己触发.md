---
url: /posts/4d70a6fc5f7e9af804814ffff86f2288/
title: 102 - NuxtLoadingIndicator：进度条的六个 prop 与自己触发
date: 2026-11-15T09:00:00+08:00
updated: 2026-11-15T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：`<NuxtLoadingIndicator>` 的官方用途只有一句——"在页面导航之间显示一个进度条"，放置位置与路由播报组件相同（`app.vue` 或 `app/layouts/`，示例里它与 `<NuxtLayout>` 并列）。它有六个 prop：`color`（进度条颜色，**可以设成 `false` 来关掉显式的颜色样式**）、`errorColor`（当 `error` 被设为 `true` 时进度条的颜色）、`height`（以像素计，默认 `3`）、`duration`（以毫秒计，默认 `2000`）、`throttle`（以毫秒计地节流出现与隐藏，默认 `200`）、`estimatedProgress`（**默认 Nuxt 会在接近 100% 时退避**；你可以提供一个自定义函数来自定进度估算，这个函数接收进度条的 duration 与已耗时间 elapsed，**应当返回 0 到 100 之间的值**）；官方还有一条 tip 解释这条默认行为：**指示条的速度会在到达由 `estimatedProgress` 控制的那个点之后逐渐降低，这个调整让它更准确地反映较长的页面加载时间，并防止指示条过早显示 100% 完成。** 它也有默认插槽可以传入自定义 HTML 或组件，并且官方说明这个组件是**可选**的、可以基于它的源码自己实现一个。底层是 `useLoadingIndicator`：官方说它"返回页面的加载状态，被 `<NuxtLoadingIndicator>` 使用且可控"，通过挂接 `page:loading:start` 与 `page:loading:end` 两个钩子来改变自身状态；它暴露三个只读属性 `isLoading`、`error`、`progress`（后者的范围是 0 到 100）与四个方法 `start()`、`set()`、`finish()`、`clear()`，三个方法都接受 `{ force: true }` 来跳过 interval 立即显示或立即重置，而 `finish()` 额外接受 `{ error: true }` 来改变进度条颜色并把 error 属性设为 true——**`error` 是这个组合式函数上的状态，组件本身只有 `errorColor` 这一个 prop。**

categories:
  - 前端开发

tags:
  - nuxt4
  - 加载指示条
  - useLoadingIndicator
  - 进度
  - 导航

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


换页时顶部那条细线——它不是 CSS 动画，而是一个有状态、有节流、有进度估算函数的组件。这一篇把它的六个 prop 和背后那个可控的组合式函数过完。

## 定位与放置

官方用途一句话："在页面导航之间显示一个进度条。"

放置方式与播报组件完全相同："在你的 `app.vue` 或 `app/layouts/` 里添加 `<NuxtLoadingIndicator/>`。"

```vue [app/app.vue]
<template>
  <NuxtLoadingIndicator />
  <NuxtLayout>
    <NuxtPage />
  </NuxtLayout>
</template>
```

同样是**并列的兄弟节点**：它自己就是一根条，不包裹任何内容。官方还写明这个组件是**可选**的，要完全自定义可以基于它的源码实现自己的一个（Source 指向 `packages/nuxt/src/app/components/nuxt-loading-indicator.ts`）。

## 六个 prop：三个尺寸、两个颜色、一个函数

| prop | 官方说明 | 类型 | 默认 |
| --- | --- | --- | --- |
| `color` | 加载条的颜色；**可以设为 `false` 来关掉显式的颜色样式** | 颜色 / `false` | —— |
| `errorColor` | 当 `error` 被设为 `true` 时加载条的颜色 | 颜色 | —— |
| `height` | 加载条的高度，**以像素计** | `number` | **`3`** |
| `duration` | 加载条的时长，**以毫秒计** | `number` | **`2000`** |
| `throttle` | 对出现与隐藏做节流，以毫秒计 | `number` | **`200`** |
| `estimatedProgress` | 自定义进度估算函数 | `function` | 见下 |

`estimatedProgress` 的官方定义值得单独读：

> 默认情况下，Nuxt 会在接近 100% 时**退避**（back off）。你可以提供一个自定义函数来自定进度估算，它是一个接收加载条的 duration（上面那格）与已耗时间 elapsed 的函数，应当返回 0 到 100 之间的一个值。

配套的 tip 说明了这条默认设计的目的：

> 加载指示条的速度会在到达由 `estimatedProgress` 控制的那个点之后逐渐降低。这个调整提供了对较长页面加载时间更准确的反映，并**防止指示条过早显示 100% 完成**。

**这是一条体验决策，不是技术限制。** 真实进度不可知，所以官方选择让曲线先快后慢；如果你换掉这个函数，就自己承担了"进度条走到 99% 停住不动"这种观感。

## `throttle` 与 `duration` 各自管什么

两者容易混，但官方的措辞很清楚：

| 格 | 单位 | 管的是 |
| --- | --- | --- |
| `duration` | 毫秒 | 加载条本身的**时长** |
| `throttle` | 毫秒 | 出现与隐藏的**节流**间隔 |

`throttle` 存在的意义是防抖：**一次很快的导航不应该让条闪一下**。这就是它默认 200 毫秒的原因——快于这个间隔的起止被吞掉。

## 默认插槽

官方说明："你可以通过加载指示条的默认插槽传入自定义 HTML 或组件。"

这条给了一个不用改 CSS 的替换路径：条的容器还是它，视觉换成你自己的元素。

## 底层可控：`useLoadingIndicator`

组件页那条 note 指向这里：**你可以用 `useLoadingIndicator` 这个组合式函数钩入底层指示条实例，它将允许你自己触发 start/finish 事件。**

组合式函数页给的定位是：

> 一个返回页面加载状态的组合式函数。**被 `<NuxtLoadingIndicator>` 使用，且可控。** 它挂接 `page:loading:start` 与 `page:loading:end`（两个都是应用运行时钩子）来改变自己的状态。

这三句话解释了架构：**换页钩子驱动状态，状态驱动组件，而你也能直接驱动同一个状态。**

### 参数、属性、方法

参数三格与组件的 prop 同名同默认值：`duration`（默认 `2000`）、`throttle`（默认 `200`）、`estimatedProgress`（默认在接近 100% 时退避，接收 duration 与 elapsed、返回 0 到 100）。

| 属性 | 类型 | 官方说明 |
| --- | --- | --- |
| `isLoading` | `Readonly<ShallowRef<boolean>>` | 加载状态 |
| `error` | `Readonly<ShallowRef<boolean>>` | 错误状态 |
| `progress` | `Readonly<ShallowRef<number>>` | 进度状态，从 `0` 到 `100` |

| 方法 | 官方说明 |
| --- | --- |
| `start()` | 把 `isLoading` 设为 true 并开始增加 `progress`。接受 `{ force: true }` 以跳过 interval、立即显示加载状态 |
| `set()` | 把 `progress` 设为一个指定值。同样接受 `{ force: true }` |
| `finish()` | 把 `progress` 设为 `100`，停止所有计时器与 interval，然后在 **`500` 毫秒后**重置加载状态。接受 `{ force: true }` 跳过重置前的 interval；接受 `{ error: true }` 来改变加载条颜色并把 error 属性设为 true |
| `clear()` | 被 `finish()` 使用。清除该组合式函数使用的所有计时器与 interval |

两个细节别漏：`finish()` 的重置有 500 毫秒延迟（所以条不会瞬间消失）；`clear()` 不是给用户调的第四个开关，它是 `finish()` 内部那一步。

### 官方的默认估算函数

官方在示例里把默认实现直接写了出来，注释是"This is how progress is calculated by default"：

```vue
<script setup lang="ts">
const { progress, isLoading, start, finish, clear } = useLoadingIndicator({
  duration: 2000,
  throttle: 200,
  // This is how progress is calculated by default
  estimatedProgress: (duration, elapsed) => (2 / Math.PI * 100) * Math.atan(elapsed / duration * 100 / 50),
})
</script>
```

这条 `atan` 曲线就是"退避"的数学形式：增长先快后慢，永远趋近但不到 100。**要覆盖它，你的函数同样得返回 0 到 100 之间的值。**

### `force` 的等价写法

官方第二个示例点明了 `start({ force: true })` 的语义：

```vue
<script setup lang="ts">
const { start, set } = useLoadingIndicator()
// same as set(0, { force: true })
// set the progress to 0, and show loading immediately
start({ force: true })
</script>
```

也就是说，**`start({ force: true })` 等价于 `set(0, { force: true })`：把进度归零并立即显示加载状态**，绕过 `throttle` 那层间隔。

## `error` 是状态，不是组件 prop

这是最容易找错地方的一个词：

| 名字 | 在哪 | 是什么 |
| --- | --- | --- |
| `errorColor` | 组件的 **prop** | 当 `error` 为 true 时条的颜色 |
| `error` | `useLoadingIndicator` 返回的**属性**（`Readonly<ShallowRef<boolean>>`） | 错误状态本身 |
| `finish({ error: true })` | 方法参数 | 同时改颜色并把 error 属性设为 true |

所以模板里写 `<NuxtLoadingIndicator :error="true">` 是无效的——组件没有这个 prop；要让条变红，走 `finish({ error: true })`。

## 自查

1. `height`、`duration`、`throttle` 三格的单位和默认值分别是什么？
2. `color` 有什么别的颜色 prop 没有的特殊取值？官方怎么说它的用途？
3. 默认的 `estimatedProgress` 行为是什么？官方给的理由是哪一句？
4. `useLoadingIndicator` 靠哪两个钩子改变自己的状态？
5. `finish()` 除了把进度设成 100 还做了哪两件事？延迟多久重置？
6. 想让进度条变红，该改哪个 prop、调哪个方法？为什么 `:error="true"` 加在组件上没用？

## 下一篇

103 讲应用外壳这个概念：一趟导航里哪些区域是固定的、哪些会随路由更换，以及 090 那五层怎么组合成一个可持续的外壳——这是把前面这些出口组件串起来的收束篇。

## 本文依据

- `https://nuxt.com/docs/4.x/api/components/nuxt-loading-indicator`
  - 原文（描述与首句）："Display a progress bar between page navigations."
  - 原文（Usage）："Add `<NuxtLoadingIndicator/>` in your `app.vue` or `app/layouts/`."；示例 `<NuxtLoadingIndicator />` 与 `<NuxtLayout><NuxtPage /></NuxtLayout>` 并列
  - 原文（Slots）："You can pass custom HTML or components through the loading indicator's default slot."
  - 原文（Props）："`color`: The color of the loading bar. It can be set to `false` to turn off explicit color styling."；"`errorColor`: The color of the loading bar when `error` is set to `true`."；"`height`: Height of the loading bar, in pixels (default `3`)."；"`duration`: Duration of the loading bar, in milliseconds (default `2000`)."；"`throttle`: Throttle the appearing and hiding, in milliseconds (default `200`)."；"`estimatedProgress`: By default Nuxt will back off as it approaches 100%. You can provide a custom function to customize the progress estimation, which is a function that receives the duration of the loading bar (above) and the elapsed time. It should return a value between 0 and 100."
  - 原文（note）："This component is optional. To achieve full customization, you can implement your own one based on its source code."
  - 原文（note）："You can hook into the underlying indicator instance using the `useLoadingIndicator` composable, which will allow you to trigger start/finish events yourself."
  - 原文（tip）："The loading indicator's speed gradually decreases after reaching a specific point controlled by `estimatedProgress`. This adjustment provides a more accurate reflection of longer page loading times and prevents the indicator from prematurely showing 100% completion."
  - 原文 link-example 指向 `https://nuxt.com/docs/examples/routing/pages`；Source 指向 `packages/nuxt/src/app/components/nuxt-loading-indicator.ts`
- `https://nuxt.com/docs/4.x/api/composables/use-loading-indicator`
  - 原文（描述）："This composable gives you access to the loading state of the app page."
  - 原文（Description）："A composable which returns the loading state of the page. Used by `<NuxtLoadingIndicator>` and controllable. It hooks into `page:loading:start` and `page:loading:end` to change its state."
  - 原文（Parameters）："`duration`: Duration of the loading bar, in milliseconds (default `2000`)."；"`throttle`: Throttle the appearing and hiding, in milliseconds (default `200`)."；"`estimatedProgress`: By default Nuxt will back off as it approaches 100%. ... It should return a value between 0 and 100."
  - 原文（Properties）：`isLoading` — type `Readonly<ShallowRef<boolean>>`、"The loading state"；`error` — type `Readonly<ShallowRef<boolean>>`、"The error state"；`progress` — type `Readonly<ShallowRef<number>>`、"The progress state. From `0` to `100`."
  - 原文（Methods）：`start()` — "Set `isLoading` to true and start to increase the `progress` value. `start` accepts a `{ force: true }` option to skip the interval and show the loading state immediately."；`set()` — "Set the `progress` value to a specific value. `set` accepts a `{ force: true }` option to skip the interval and show the loading state immediately."；`finish()` — "Set the `progress` value to `100`, stop all timers and intervals then reset the loading state `500` ms later. `finish` accepts a `{ force: true }` option to skip the interval before the state is reset, and `{ error: true }` to change the loading bar color and set the error property to true."；`clear()` — "Used by `finish()`. Clear all timers and intervals used by the composable."
  - 原文（Example）：`useLoadingIndicator({ duration: 2000, throttle: 200, estimatedProgress: (duration, elapsed) => (2 / Math.PI * 100) * Math.atan(elapsed / duration * 100 / 50) })`，注释 `// This is how progress is calculated by default`
  - 原文（Example 2）：`const { start, set } = useLoadingIndicator()`、`// same as set(0, { force: true })`、`// set the progress to 0, and show loading immediately`、`start({ force: true })`
  - 原文 Source：`packages/nuxt/src/app/composables/loading-indicator.ts`
- `https://nuxt.com/docs/4.x/api/components/nuxt-route-announcer`
  - 对照用（见 101）：放置位置同为 `app.vue` 或 `app/layouts/`，同样可选、同样可基于源码自实现
