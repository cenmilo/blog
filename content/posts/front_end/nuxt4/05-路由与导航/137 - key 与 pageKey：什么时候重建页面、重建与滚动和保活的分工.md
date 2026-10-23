---
url: /posts/928edf1259835cd70e77a16c14e750b0/
title: 137 - key 与 pageKey：什么时候重建页面、重建与滚动和保活的分工
date: 2026-11-26T11:00:00+08:00
updated: 2026-11-26T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一格管的是"**同一条路由换了参数时，页面组件要不要重来一遍**"。官方在两个地方给了同一个能力：`definePageMeta` 的 `key`（类型 `false | string | ((route: RouteLocationNormalizedLoaded) => string)`，说明为"**当你需要更多地控制 `<NuxtPage>` 组件何时被重新渲染时，设置一个 `key` 值**"）与 `<NuxtPage>` 的 `pageKey` prop（类型 `string` 或 `function`，说明为"**控制 `NuxtPage` 组件何时被重新渲染**"）。页面目录那一节把两者的等价关系写得很明确："**如果你想要更多地控制 `<NuxtPage>` 组件何时被重新渲染（例如为了过渡），你可以通过 `pageKey` prop 传入一个字符串或函数，或者通过 `definePageMeta` 定义一个 `key` 值。**"——官方随后并列给出两种写法：父页面出口上 `<NuxtPage :page-key="route => route.fullPath" />`，或子页面里 `definePageMeta({ key: route => route.fullPath })`。两条极端情况的官方说法是：「如果你传入一个从不改变的 key，`<NuxtPage>` 组件将只被渲染一次——在它首次挂载时」，示例写作 `<NuxtPage page-key="static" />`；而动态 key 的官方写法是 `<NuxtPage :page-key="route => route.fullPath" />`，紧跟一条 warning：**"不要在这里使用 `$route` 对象，因为它会给 `<NuxtPage>` 用 `<Suspense>` 渲染页面的方式造成问题。"** 另外三件与它相连的事：① 官方把 `key` 的动机之一写成"例如为了过渡"——**不重建就没有可过渡的新元素**（091 的 `<Transition>` 结构）；② `scrollToTop` 官方明确它是独立的："**导航独立于渲染，所以即使页面不重新渲染（例如使用固定的 `key` 时），滚动行为也总会触发。在这种情形下设 `scrollToTop: false` 来禁用滚动。**"；③ 状态保留归 `keepalive`——官方的场景说明正是"**例如在一个拥有动态子路由的父路由里，如果你想跨路由变化保留页面状态**"，并给出父路由专用写法 `<NuxtPage keepalive />`。

categories:
  - 前端开发

tags:
  - nuxt4
  - key
  - pagekey
  - 重渲染
  - keepalive

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


从 `/posts/1` 点到 `/posts/2`，是"同一个组件换个数据"还是"换一个新组件"？官方把这件事交给一个值来定：`key`（或出口上的 `pageKey`）。它一旦定下来，过渡、滚动、状态保留三件事的表现都跟着变。

## 一个字段的两种落点，官方说它们等价

| 位置 | 官方定义 |
| --- | --- |
| `definePageMeta` 的 `key` | "**当你需要更多地控制 `<NuxtPage>` 组件何时被重新渲染时，设置一个 `key` 值。**"类型 `false \| string \| ((route: RouteLocationNormalizedLoaded) => string)` |
| `<NuxtPage>` 的 `pageKey` prop | "**控制 `NuxtPage` 组件何时被重新渲染。**"类型 `string` 或 `function` |

页面目录那一节把等价关系写成一整句：

> 如果你想要更多地控制 `<NuxtPage>` 组件何时被重新渲染（**例如为了过渡**），你可以通过 `pageKey` prop 传入一个字符串或函数，**或者**通过 `definePageMeta` 定义一个 `key` 值。

官方并列的两种写法：

```vue {}[pages/parent.vue]
<template>
  <div>
    <h1>I am the parent view</h1>
    <NuxtPage :page-key="route => route.fullPath" />
  </div>
</template>
```

```vue {}[pages/parent/child.vue]
<script setup lang="ts">
definePageMeta({
  key: route => route.fullPath,
})
</script>
```

两者的差别不是效果，而是**归属方向**：

| 写法 | 谁决定 | 影响范围 |
| --- | --- | --- |
| 出口上的 `:page-key` | **父页面**（出口的所有者） | 这个出口渲染出的全部子页面 |
| 页面里的 `definePageMeta({ key })` | **页面自己** | 只有这一页 |

`key` 的官方说明里还有个 `false` 这个合法取值（类型 `false | string | 函数`）——它在类型层面允许，官方页上没有展开语义，所以**别指望它等于任何你想要的行为**，要用字符串/函数这两种有示例的形态。

## 两个端点：永不改变 vs 随路径改变

官方给的第一端：

> 例如，如果你传入一个**从不改变**的 key，`<NuxtPage>` 组件**将只被渲染一次——在它首次挂载时**。

```vue [app/app.vue]
<template>
  <NuxtPage page-key="static" />
</template>
```

第二端就是上面那个 `route => route.fullPath`。

至于"不设 key 时是哪一端"，官方没有写一句"默认为复用"；但从官方自己给的两段能力说明能定下来：

- `pageKey` 存在的目的是"**更多地控制**何时被重新渲染"——如果默认就是"永不重建"，`key` 这个字段与 `keepalive` 都没有存在必要。
- 官方对 `keepalive` 的场景说明是"**在一个拥有动态子路由的父路由里，如果你想跨路由变化保留页面状态**"——**要保留，说明默认不保留**。

**结论按官方语义读：想固定就传一个不变的 key（`page-key="static"` 的写法），想随参数重来就传 `route => route.fullPath`。**

## 一条 warning：不要用 `$route` 对象

官方在动态 key 那行之后紧跟着写：

> **不要在这里使用 `$route` 对象，因为它会给 `<NuxtPage>` 用 `<Suspense>` 渲染页面的方式造成问题。**

对照官方给的两种正确写法就能看出差别：

| 写法 | 合规 |
| --- | --- |
| `<NuxtPage :page-key="route => route.fullPath" />` | ✔ 用回调**参数**里的 `route` |
| `definePageMeta({ key: route => route.fullPath })` | ✔ 同上 |
| `<NuxtPage :page-key="$route.fullPath" />` | ✘ 官方点名的 `$route` 对象 |

原因回指 091 那张结构图：`<Suspense>` 包在页面外面，key 参与的是"什么时候重建这个 Suspense 内容"，直接引用那个全局响应式对象会绕开这条链。

## 三件被 `key` 牵动的事

### ① 过渡：不重建就没有可过渡的对象

官方给 `key` 的动机原文是"**例如为了过渡**"。原因在 091 那张结构图里：`<Transition>` 包的是被 `<Suspense>` 承载的那个页面组件——**Vue 的过渡发生在元素被替换/插入时**。key 不变→组件不重建→过渡不跑，这时你看到的症状是"CSS 类都写对了但页面切换没有动画"。

所以排查顺序应该反过来：**过渡不跑，先确认这次导航到底有没有触发重建**，再看 CSS 前缀（118）。

### ② 滚动：`scrollToTop` 与它无关，别指望用 key 控制

这一格官方写得非常明确，就是为消除这个误解：

> 告诉 Nuxt 在渲染页面之前是否滚动到顶部。**导航独立于渲染，所以即使页面不重新渲染（例如使用固定的 `key` 时），滚动行为也总会触发。**在这种情形下**设 `scrollToTop: false` 来禁用滚动**。

**也就是说：用固定 `key` 复用页面时，滚动仍然会跳顶部。** 要换参数时不跳顶部，得显式 `scrollToTop: false`，或者按官方另一句去 `~/router.options.ts` 里覆盖默认滚动行为（138）。

### ③ 状态：`keepalive` 管的是"重建时留不留"

| 字段 | 管什么 |
| --- | --- |
| `key` / `pageKey` | **要不要重建**（092、137） |
| `keepalive` | 组件被切换时**状态留不留**；官方场景："在一个拥有动态子路由的父路由里，如果你想跨路由变化保留页面状态" |
| `<NuxtPage keepalive />` | 官方给的**父路由专用写法**："When your goal is to preserve state for parent routes use this syntax" |

三个凑在一起时的实际含义：**固定 key + keepalive** 才是"这个页面别重做、状态留着"；**动态 key + keepalive** 则是"每次换参数都重建，但被换掉的那些实例的状态仍被 `<KeepAlive>` 缓存着"（细粒度控制用 `keepalive: { exclude: [...] }`，119）。

## 该选哪条路：重建 or 不重建

| 你要的行为 | 官方给的组合 |
| --- | --- |
| 换参数就重新走一遍页面逻辑（`setup` 重跑、`onMounted` 再来） | `key: route => route.fullPath` |
| 换参数**不**重来，只让数据更新 | **不设动态 key**，改用响应式取数：把 `route.params`/ref 交给取数选项，或 `watch: [page]`，或用 computed 派生 key（122 的三条路径） |
| 换参数要有过渡动画 | 动态 key（不重建就没有过渡）+ 单根元素 + CSS 前缀对上 `name` |
| 换参数不要跳顶部 | `scrollToTop: false`（**不是**靠固定 key） |
| 换子路由时父页面状态要留 | `<NuxtPage keepalive />` 或 `keepalive: true` |

**最容易搞错的是第二行**：很多人先写了 `watch(route, ...)` 手动同步，而 122 说官方给的正解是让取数选项本身响应式。**先决定要不要重建，再决定用什么同步数据**——顺序反了会同时用两套机制互相打架。

## 自查

1. `key` 与 `pageKey` 分别写在哪？官方用哪句话说明它们等价？
2. 传一个永不改变的 key 会怎样？官方的示例写法是什么？
3. 想按路径重建，两种写法分别是什么？官方禁止在这里使用哪个对象、理由是什么？
4. 过渡不跑时，为什么第一步是确认"有没有重建"？
5. 固定 key 时滚动还会不会跳顶部？官方给的禁用方式是什么？
6. `key`、`keepalive`、`<NuxtPage keepalive />` 三者分别管什么？
7. "换参数不重来、只更新数据"这条路线，官方的正解是什么（而不是手写 watch）？

## 下一篇

138 讲滚动这一层：`scrollToTop` 为什么与渲染无关、`router.options.ts` 与 `scrollBehaviorType` 分别管什么、`hashMode` 那条"URL 从不发给服务器且不支持 SSR"的限制。
