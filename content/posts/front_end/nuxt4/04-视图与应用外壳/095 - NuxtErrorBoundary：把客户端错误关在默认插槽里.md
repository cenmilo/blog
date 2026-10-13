---
url: /posts/7e88d82c5220cd7b2488f7da2c30c6eb/
title: 095 - NuxtErrorBoundary：把客户端错误关在默认插槽里
date: 2026-11-12T11:00:00+08:00
updated: 2026-11-12T11:00:00+08:00
author: cmdragon

summary:
  摘要：`<NuxtErrorBoundary>` 的官方定义是"处理发生在其默认插槽内的客户端错误"，而它在错误处理那一节的定位说得更实用：**允许你处理应用内部的客户端错误，而不必用错误页替换掉整个站点**。这个组件有三条边界必须一起记住——只管**默认插槽**里发生的错误（官方："这个组件负责处理发生在其默认插槽内的错误。在客户端，它会阻止错误向上冒泡到顶层，转而渲染 `#error` 插槽"）；只管**客户端**错误；底层是 Vue 的 `onErrorCaptured` 钩子（官方 tip 原话）。对外接口只有两个：事件 `@error`（"当该组件的默认插槽抛出错误时发出"）与插槽 `#error`（"指定在发生错误时要显示的后备内容"，可从中拿到 `error` 与 `clearError`）。关于恢复，错误处理页写了两条容易漏的：`#error` 插槽会接收 `error` 作为 prop，**如果你把 `error` 设为 `null`，会触发默认插槽重新渲染——你需要先确保错误真的被解决了，否则错误插槽只会被再渲染一次**；以及一条 tip——**如果你导航到另一个路由，错误会被自动清除**。想在 `<script setup>` 里读 `error` 或调用 `clearError()`，官方的做法是用 `useTemplateRef('errorBoundary')` 拿到实例后走 `errorBoundary.value?.error` 与 `errorBoundary.value?.clearError()`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 错误处理
  - errorboundary
  - 插槽
  - 水合

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


一个组件抛错，默认结局是整站换成错误页。`<NuxtErrorBoundary>` 提供另一个结局：**只把那一片换掉，页面其余部分照常。** 但它的边界比"包起来就安全"窄得多，这一篇把边界画准。

## 官方怎么定位它

| 出处 | 原话 |
| --- | --- |
| 组件页 | "`<NuxtErrorBoundary>` 组件处理发生在其**默认插槽**内的**客户端**错误。" |
| 错误处理页 | "Nuxt 还提供了 `<NuxtErrorBoundary>` 组件，允许你处理应用内部的客户端错误，**而不用你的整个站点被错误页替换**。" |
| 错误处理页 | "这个组件负责处理发生在其默认插槽内的错误。在客户端，它会**阻止错误向上冒泡到顶层**，转而渲染 `#error` 插槽。" |

三条合起来就是这个组件的全部范围：**默认插槽之内、客户端、阻止冒泡**。

## 三条边界各对应一种"包不住"

| 边界 | 包不住的情形 |
| --- | --- |
| 只管默认插槽 | 错误发生在 `#error` 插槽自身的内容里 |
| 只管客户端错误 | 服务端渲染阶段就已经抛出去的错误——那类错误走的是错误页那条路，不是这里 |
| 底层是 `onErrorCaptured` | 官方 tip 明确写着这个组件在底层使用 Vue 的 `onErrorCaptured` 钩子。它继承的是这个钩子的语义：捕获的是**其子树内**组件抛出的错误 |

最常被误用的就是第一条。**边界只圈住它包着的那部分模板**，页面其他地方照样会冒泡到顶层。

## 用法：一层包裹、一个事件、一个插槽

`@error` 事件——官方说明："`@error`：当该组件的默认插槽抛出错误时发出。"

```vue [app/app.vue]
<template>
  <NuxtErrorBoundary @error="logSomeError">
    <!-- ... -->
  </NuxtErrorBoundary>
</template>
```

`#error` 插槽——官方说明："`#error`：指定在发生错误时要显示的后备内容。"

```vue [app/app.vue]
<template>
  <NuxtErrorBoundary>
    <!-- ... -->
    <template #error="{ error, clearError }">
      <p>An error occurred: {{ error }}</p>

      <button @click="clearError">
        Clear error
      </button>
    </template>
  </NuxtErrorBoundary>
</template>
```

错误处理页给的示例是同一件事，只是放在页面里，并把两条注释写清了用途：

```vue [app/pages/index.vue]
<template>
  <!-- some content -->
  <NuxtErrorBoundary @error="someErrorLogger">
    <!-- You use the default slot to render your content -->
    <template #error="{ error, clearError }">
      You can display the error locally here: {{ error }}
      <button @click="clearError">
        This will clear the error.
      </button>
    </template>
  </NuxtErrorBoundary>
</template>
```

`@error` 挂的是"记录一次"，`#error` 挂的是"给用户看什么、以及怎么回到正常内容"。这两件事分开，是因为它们的时机不同。

## 恢复：`clearError` 之后发生什么

这是这一节唯一要背下来的因果，官方写得很完整：**`#error` 插槽会接收 `error` 作为一个 prop。（如果你把 `error` 设置为 `null`，会触发默认插槽重新渲染；你需要先确保错误已经被完全解决，否则错误插槽只会被再渲染一次。）**

把这段拆成执行顺序：

1. 点按钮 → `clearError` 把 `error` 置空
2. `error` 为空 → 边界重新渲染**默认插槽**
3. 如果错误的根源没解决，默认插槽里的代码又抛一次错 → 又渲染 `#error`

所以"点了按钮页面闪回错误态"不是 `clearError` 坏了，而是第 3 步按官方描述正常发生。想让"重试"真的成立，得在触发 `clearError` 之前先改变导致错误的那个条件。

另一条更省事的退出机制，官方写成 tip：**如果你导航到另一个路由，错误会被自动清除。**

## 想在脚本里读错误：走 `useTemplateRef`

官方给的示例是"在组件脚本中访问 `error` 和 `clearError`"：

```vue [app/app.vue]
<template>
  <NuxtErrorBoundary ref="errorBoundary">
    <!-- ... -->
  </NuxtErrorBoundary>
</template>

<script setup lang="ts">
const errorBoundary = useTemplateRef('errorBoundary')

// errorBoundary.value?.error
// errorBoundary.value?.clearError()
</script>
```

两点注意：`?.` 不能省（`value` 在拿到实例前是空的）；这里取的是组件实例上的 `error` / `clearError`，与 `#error` 插槽里解构出来的那两个是同一份状态的两个入口。

## 它和 `error.vue` 的分工

| | `error.vue` | `<NuxtErrorBoundary>` |
| --- | --- | --- |
| 官方用途 | 你应用的错误页 | 处理应用内部的客户端错误，而不整站替换 |
| 影响范围 | 整个站点换成这一页 | 只有默认插槽那一片 |
| 能覆盖的错误 | 渲染链上冒到顶层的错误 | 只在默认插槽内、且是客户端错误 |
| 恢复手段 | 走 `clearError` 那类工具或跳转 | `clearError`，或导航到别的路由（自动清除） |
| 典型场景 | 404、500 | 某个会失败的挂件、某块第三方内容 |

这两层是叠加关系，不是二选一：边界接住的错误不会走到错误页，边界**没**接住的（服务端阶段的、插槽外的）才会。

## 自查

1. `<NuxtErrorBoundary>` 处理哪一类错误、管哪一段模板？它的底层钩子是什么？
2. `@error` 事件在什么时候发出？`#error` 插槽能拿到什么？
3. 调 `clearError` 之后渲染顺序是怎样的？为什么"点了又回到错误态"可能是正常行为？
4. 还有什么情况会清除错误，不需要你手写代码？
5. 服务端渲染阶段就抛出去的错误，会被这个组件接住吗？该走哪一层？

## 下一篇

096 讲 `<ClientOnly>`：默认插槽的内容如何被从服务端构建里摇掉、`#fallback` 插槽与 `fallback`/`placeholder` prop 两种占位写法的区别，以及为什么它显示占位直到挂载完成。

## 本文依据

- `https://nuxt.com/docs/4.x/api/components/nuxt-error-boundary`
  - 原文："The \<NuxtErrorBoundary> component handles client-side errors happening in its default slot."
  - 原文（tip）："The `<NuxtErrorBoundary>` uses Vue's `onErrorCaptured` hook under the hood."
  - 原文（Events）："`@error`: Event emitted when the default slot of the component throws an error."；示例 `<NuxtErrorBoundary @error="logSomeError">`（文件标注 `app/app.vue`）
  - 原文（Slots）："`#error`: Specify a fallback content to display in case of error."；示例 `<template #error="{ error, clearError }"><p>An error occurred: {{ error }}</p><button @click="clearError">Clear error</button></template>`
  - 原文（Example - Accessing `error` and `clearError` in Script）："You can access `error` and `clearError` properties within the component's script as below:"；示例 `const errorBoundary = useTemplateRef('errorBoundary')`、`// errorBoundary.value?.error`、`// errorBoundary.value?.clearError()`
  - 原文 Source：`packages/nuxt/src/app/components/nuxt-error-boundary.vue`
- `https://nuxt.com/docs/4.x/getting-started/error-handling`
  - 原文（Render Error in Component）："Nuxt also provides a `<NuxtErrorBoundary>` component that allows you to handle client-side errors within your app, without replacing your entire site with an error page."
  - 原文："This component is responsible for handling errors that occur within its default slot. On client-side, it will prevent the error from bubbling up to the top level, and will render the `#error` slot instead."
  - 原文："The `#error` slot will receive `error` as a prop. (If you set `error = null` it will trigger re-rendering the default slot; you'll need to ensure that the error is fully resolved first or the error slot will just be rendered a second time.)"
  - 原文（tip）："If you navigate to another route, the error will be cleared automatically."
  - 原文示例（文件标注 `app/pages/index.vue`）：`<!-- some content -->`、`<NuxtErrorBoundary @error="someErrorLogger">`、`<!-- You use the default slot to render your content -->`、`<template #error="{ error, clearError }">`、`You can display the error locally here: {{ error }}`、`<button @click="clearError">This will clear the error.</button>`
  - 原文 link-example 指向 `https://nuxt.com/docs/examples/advanced/error-handling`
