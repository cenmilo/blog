---
url: /posts/7b0ec7de2aacbaf521cf661b12ff5ec3/
title: useLoadingIndicator：三个状态、四个方法与 force 的三种含义
date: 2026-11-29T11:00:00+08:00
updated: 2026-11-29T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：102 讲的那个组件只是壳，真正的控制器是这个 composable。官方定位句："**This composable gives you access to the loading state of the app page.**"，机制说明："A composable which returns the loading state of the page. **Used by `<NuxtLoadingIndicator>` and controllable. It hooks into `page:loading:start` and `page:loading:end` to change its state.**"。它暴露**三个状态**：`isLoading`、`error`、`progress`，类型全是 `Readonly<ShallowRef<…>>`（`boolean` / `boolean` / `number`，progress 官方注明 "From `0` to `100`"）；**四个方法**：`start()`、`set()`、`finish()`、`clear()`。`force` 这个选项在三个方法上含义不同，官方逐条写明：`start` 与 `set` 的 `{ force: true }` 是 "**to skip the interval and show the loading state immediately**"，而 `finish` 的 `{ force: true }` 是 "**to skip the interval before the state is reset**"，`finish` 还有 `{ error: true }`——"**to change the loading bar color and set the error property to true**"。`finish()` 的正常路径官方写成一句话："Set the `progress` value to `100`, stop all timers and intervals **then reset the loading state `500` ms later**."，而 `clear()` 是 "**Used by `finish()`.** Clear all timers and intervals used by the composable."。三个参数（`duration` 默认 2000、`throttle` 默认 200、`estimatedProgress`）与组件那一页的三个同名 prop 描述逐字一致；官方对退速那段还有一条 tip："The loading indicator's speed gradually decreases after reaching a specific point controlled by `estimatedProgress`. This adjustment provides a more accurate reflection of longer page loading times and prevents the indicator from **prematurely showing 100% completion**."。至于它凭什么自动动，官方钩子表里那两行是关键，其中 `page:loading:start` 还带了一句容易踩的说明："Called when a route navigation begins (before resolution) or when the page key changes. **May fire without the page component's `setup()` re-running if the page is reused (e.g. with a static `key` in `definePageMeta`).**"

categories:
  - 前端开发

tags:
  - nuxt4
  - 加载指示器
  - composables
  - 钩子
  - 进度条

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


我给项目加了 `<NuxtLoadingIndicator/>` 之后，遇到的下一个问题是："发起一个不是路由跳转的请求时，能不能也用这条进度条？"能——进度条的状态本来就是一个 composable 在持有，组件只是把它画出来。

## 关系先摆正：组件是壳，composable 是状态源

官方两页把方向说得很清楚：

- `useLoadingIndicator` 页："A composable which returns the loading state of the page. **Used by `<NuxtLoadingIndicator>` and controllable.**"
- `<NuxtLoadingIndicator>` 页："You can hook into the underlying indicator instance using **the `useLoadingIndicator` composable**, which will allow you to **trigger start/finish events yourself**."

再加上这一句，它和导航的绑定关系就不是魔法：

> It hooks into `page:loading:start` and `page:loading:end` to change its state.

**它自己不动，是这两个钩子推它动。** 所以进度条看起来"自动"，本质是：导航开始 → `page:loading:start` → `isLoading` 变 true；导航结束 → `page:loading:end` → 收尾。

## 三个状态：全是只读的

| 属性 | 官方类型 | 官方说明 |
| --- | --- | --- |
| `isLoading` | `Readonly<ShallowRef<boolean>>` | The loading state |
| `error` | `Readonly<ShallowRef<boolean>>` | The error state |
| `progress` | `Readonly<ShallowRef<number>>` | The progress state. From `0` to `100`. |

类型是 `Readonly<…>`，意味着**你不能直接写它**——要改状态只能走方法。这一点很实用：想在别处显示"当前进度百分比"，读 `progress` 就行；想让它跳到某个值，必须 `set(...)`。

`error` 这一格有个容易被误读的地方，需要和组件那一页对照：

- 组件侧的 prop 是 `errorColor`："The color of the loading bar **when `error` is set to `true`**."
- composable 侧只有 `finish()` 的一个选项能设它：`{ error: true }` —— "**to change the loading bar color and set the error property to true**".

**组件的 props 表里没有 `error` 这一项**（六个 prop 是 `color`、`errorColor`、`height`、`duration`、`throttle`、`estimatedProgress`，见 102）。也就是说：想让用户看到"这次导航出错了"的红条，唯一入口是调用侧的 `finish({ error: true })`，不是给组件传 prop。

## 三个参数：和组件的同名 prop 是一回事

composable 的参数（官方原文）：

| 参数 | 官方说明 |
| --- | --- |
| `duration` | Duration of the loading bar, in milliseconds (default `2000`). |
| `throttle` | Throttle the appearing and hiding, in milliseconds (default `200`). |
| `estimatedProgress` | By default Nuxt will back off as it approaches 100%. You can provide a custom function to customize the progress estimation, which is a function that receives the duration of the loading bar (above) and the elapsed time. **It should return a value between 0 and 100.** |

这三行文字与组件那一页对应三个 prop 的描述**逐字相同**，`duration: 2000` / `throttle: 200` 两个默认值也一致。官方给的示例是"把三个参数都写出来 + 把默认曲线函数原样贴回来"：

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

官方注释 "This is how progress is calculated by default" 就是那条默认曲线：入参是 `duration` 与 `elapsed`，返回 0-100 的一个数。为什么要这么绕？组件那一页的 tip 说了目的：

> The loading indicator's speed gradually decreases after reaching a specific point controlled by `estimatedProgress`. This adjustment provides a more accurate reflection of longer page loading times and prevents the indicator from prematurely showing 100% completion.

拆成两句人话：**先快后慢**；慢下来是为了"不至于还没好就显示成 100%"。

## 四个方法，和 force 的三种含义

官方原文逐条：

| 方法 | 官方定义 |
| --- | --- |
| `start()` | Set `isLoading` to true and start to increase the `progress` value. `start` accepts a `{ force: true }` option **to skip the interval and show the loading state immediately**. |
| `set()` | Set the `progress` value to a specific value. `set` accepts a `{ force: true }` option **to skip the interval and show the loading state immediately**. |
| `finish()` | Set the `progress` value to `100`, stop all timers and intervals **then reset the loading state `500` ms later**. `finish` accepts a `{ force: true }` option **to skip the interval before the state is reset**, and `{ error: true }` **to change the loading bar color and set the error property to true**. |
| `clear()` | **Used by `finish()`.** Clear all timers and intervals used by the composable. |

`force` 在两处的落点完全不同，这是最容易记混的：

- `start({ force: true })` / `set(..., { force: true })`：跳过 interval，**立刻把加载态显示出来**（绕开的正是 `throttle` 那段"太快的导航不值得闪一下"）。
- `finish({ force: true })`：跳过的是**状态重置之前的那段 interval**，也就是不等那 500 ms。

官方还有一个等价写法值得抄：

```vue
<script setup lang="ts">
const { start, set } = useLoadingIndicator()
// same as set(0, { force: true })
// set the progress to 0, and show loading immediately
start({ force: true })
</script>
```

那两行注释把 `start({ force: true })` 与 `set(0, { force: true })` 划了等号——**"强制立即显示"的实际动作就是"把进度归零并立刻可见"**。

`clear()` 的定位也在这张表里：它是 `finish()` 内部使用的那一步（"Used by `finish()`"），单独调用就是"把所有 timer 与 interval 清掉"，不改 `progress` 到 100、也不安排那次重置。

## 它凭什么跟导航同步：把四个钩子排成一列

`useLoadingIndicator` 只挂两个钩子，但 App Hooks 表里跟"页面加载"相关的有五个，全在 Client 侧：

| Hook | Arguments | Description（原文） |
| --- | --- | --- |
| `page:loading:start` | - | Called when a route navigation begins (before resolution) or **when the page key changes**. May fire without the page component's `setup()` re-running if the page is reused (e.g. with a static `key` in `definePageMeta`). |
| `page:start` | `pageComponent?` | Called on Suspense inside of `NuxtPage` pending event. |
| `page:finish` | `pageComponent?` | Called on Suspense inside of `NuxtPage` resolved event. |
| `page:loading:end` | - | Called after `page:finish` |
| `page:transition:finish` | `pageComponent?` | After page transition onAfterLeave event. |

两件事直接影响进度条的表现：

1. **`page:loading:start` 的触发条件不止"导航"**：官方写了"or when the page key changes"（137 讲的 `key`/`pageKey`）。所以换 key 导致重建页面时，进度条也会走一遍。
2. **进度条动了，不代表 `setup()` 重跑了**：同一条官方说明的下一句就是"**May fire without the page component's `setup()` re-running if the page is reused**（例如 `definePageMeta` 里给了静态 `key`）"。这条对排障很值钱——**别用"进度条闪了"来判断"数据重新取了"**。

而 `page:loading:end` 只有一句 "Called after `page:finish`"：收尾发生在 Suspense 解析完之后，不是在点击那一刻。

## 用它的三种真实场景

| 场景 | 写法 | 依据 |
| --- | --- | --- |
| 给一次非路由的长请求挂同一条进度条 | 请求前 `start({ force: true })`，`finally` 里 `finish()` | 组件页："which will allow you to trigger start/finish events yourself" |
| 请求失败要让条变红 | `finish({ error: true })` | `finish` 的 `{ error: true }` 定义；组件侧 `errorColor` 生效条件 |
| 不用官方组件、自己画一个 | 读 `isLoading` / `progress` 自己渲染，甚至照源码自研 | "This component is optional. To achieve full customization, you can implement your own one based on its source code."（102 讲的默认插槽同样可用） |

一个可复用的小形状：

```vue
<script setup lang="ts">
const { start, finish } = useLoadingIndicator()

async function heavyAction () {
  start({ force: true })
  try {
    await $fetch('/api/slow')
    finish()
  } catch {
    finish({ error: true })
  }
}
</script>
```

这是我按上面几条定义拼的写法，不是官方示例；官方示例只演示了解构与 `start({ force: true })`。

## 与 102 的分工

| | 102（组件页） | 本篇（composable 页） |
| --- | --- | --- |
| 讲什么 | 放哪儿、六个 prop、默认插槽、"组件是可选的" | 三个状态、四个方法、force 的落点、钩子时序 |
| 你在这里找 | `color` / `height` / `errorColor` 这类外观控制 | `start` / `finish` / `progress` 这类行为控制 |

## 自查

1. `useLoadingIndicator` 与 `<NuxtLoadingIndicator>` 谁用谁？它是靠哪两个钩子改状态的？
2. 三个属性分别是什么类型？为什么"不能直接写 `progress`"？
3. 用户看到红色进度条的条件是什么？组件的六个 prop 里有 `error` 吗？
4. `start({ force: true })` 与 `finish({ force: true })` 各自跳过的是什么？
5. `finish()` 正常路径做哪三件事？那个 500 ms 用在哪一步？
6. 默认的 `estimatedProgress` 是哪两个入参、返回什么范围？官方给这条曲线的目的是什么？
7. 进度条闪了一下，能推出页面 `setup()` 重跑了吗？官方在哪一行说明了原因？
8. 想让一次非路由请求复用这条进度条，你会怎么调？

## 下一篇

147 把 `<NuxtLink>` 的 props 一次摊平：RouterLink 组、NuxtLink 组、Anchor 组分别有哪些，`href` 与 `to` 同时写谁生效，以及"内部链接还是外部链接"是怎么判定的。

## 本文依据

- `https://nuxt.com/docs/4.x/api/composables/use-loading-indicator`
  - 原文（描述）："This composable gives you access to the loading state of the app page."；"A composable which returns the loading state of the page. Used by `<NuxtLoadingIndicator>` and controllable. It hooks into `page:loading:start` and `page:loading:end` to change its state."
  - 原文（Parameters）："`duration`: Duration of the loading bar, in milliseconds (default `2000`)."；"`throttle`: Throttle the appearing and hiding, in milliseconds (default `200`)."；"`estimatedProgress`: By default Nuxt will back off as it approaches 100%. You can provide a custom function to customize the progress estimation, which is a function that receives the duration of the loading bar (above) and the elapsed time. It should return a value between 0 and 100."
  - 原文（Properties）：`isLoading` — "**type**: `Readonly<ShallowRef<boolean>>`" / "The loading state"；`error` — "**type**: `Readonly<ShallowRef<boolean>>`" / "The error state"；`progress` — "**type**: `Readonly<ShallowRef<number>>`" / "The progress state. From `0` to `100`."
  - 原文（Methods）："`start()` Set `isLoading` to true and start to increase the `progress` value. `start` accepts a `{ force: true }` option to skip the interval and show the loading state immediately."；"`set()` Set the `progress` value to a specific value. `set` accepts a `{ force: true }` option to skip the interval and show the loading state immediately."；"`finish()` Set the `progress` value to `100`, stop all timers and intervals then reset the loading state `500` ms later. `finish` accepts a `{ force: true }` option to skip the interval before the state is reset, and `{ error: true }` to change the loading bar color and set the error property to true."；"`clear()` Used by `finish()`. Clear all timers and intervals used by the composable."
  - 原文（Example 一）：`const { progress, isLoading, start, finish, clear } = useLoadingIndicator({ duration: 2000, throttle: 200, // This is how progress is calculated by default \n estimatedProgress: (duration, elapsed) => (2 / Math.PI * 100) * Math.atan(elapsed / duration * 100 / 50) })`
  - 原文（Example 二）：`const { start, set } = useLoadingIndicator()` + 注释 "// same as set(0, { force: true })" + "// set the progress to 0, and show loading immediately" + `start({ force: true })`
  - 原文（Source）：`packages/nuxt/src/app/composables/loading-indicator.ts`
- `https://nuxt.com/docs/4.x/api/components/nuxt-loading-indicator`
  - 原文（六 props）："`color`: The color of the loading bar. It can be set to `false` to turn off explicit color styling."；"`errorColor`: The color of the loading bar when `error` is set to `true`."；"`height`: Height of the loading bar, in pixels (default `3`)."；"`duration`: Duration of the loading bar, in milliseconds (default `2000`)."；"`throttle`: Throttle the appearing and hiding, in milliseconds (default `200`)."；"`estimatedProgress`: By default Nuxt will back off as it approaches 100%. ... It should return a value between 0 and 100."
  - 原文（三条 note/tip）："This component is optional. To achieve full customization, you can implement your own one based on its source code."；"You can hook into the underlying indicator instance using the `useLoadingIndicator` composable, which will allow you to trigger start/finish events yourself."；"The loading indicator's speed gradually decreases after reaching a specific point controlled by `estimatedProgress`. This adjustment provides a more accurate reflection of longer page loading times and prevents the indicator from prematurely showing 100% completion."
  - 原文（Slots）："You can pass custom HTML or components through the loading indicator's default slot."
- `https://nuxt.com/docs/4.x/api/advanced/hooks`
  - 原文（App Hooks 表相关五行）："`page:loading:start` | - | Client | Called when a route navigation begins (before resolution) or when the page key changes. May fire without the page component's `setup()` re-running if the page is reused (e.g. with a static `key` in `definePageMeta`)."；"`page:start` | `pageComponent?` | Client | Called on Suspense inside of `NuxtPage` pending event."；"`page:finish` | `pageComponent?` | Client | Called on Suspense inside of `NuxtPage` resolved event."；"`page:loading:end` | - | Client | Called after `page:finish`"；"`page:transition:finish` | `pageComponent?` | Client | After page transition onAfterLeave event."
- `https://nuxt.com/docs/4.x/api/utils/define-page-meta` 与 `https://nuxt.com/docs/4.x/api/components/nuxt-page`
  - 对照用（见 092、137）：`key` / `pageKey` 与页面复用、`Suspense` 在 `NuxtPage` 内部的位置
