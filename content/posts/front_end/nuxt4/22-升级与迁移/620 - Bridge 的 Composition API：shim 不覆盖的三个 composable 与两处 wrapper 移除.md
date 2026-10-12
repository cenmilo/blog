---
url: /posts/3f15b4f0ecfb075322da93157c2ae358/
title: Bridge 的 Composition API：shim 不覆盖的三个 composable 与两处 wrapper 移除
date: 2027-05-06T11:00:00+08:00
updated: 2027-05-06T11:00:00+08:00
author: cmdragon

summary:
  摘要：`bridge/bridge-composition-api`（页面标题是 **Legacy Composition API**）的前提是：**Bridge 自带的 Composition API 实现是照 Nuxt 3 对齐的，所以以前用过 Composition API 的项目要多做几步**。第一步是删两个包：**`@vue/composition-api`** 与 **`@nuxtjs/composition-api`（连同 `nuxt.config` 里 modules 的那份）**。之后分两条路：只用过 `@vue/composition-api` 的，**只需删掉手动注册那几行，Bridge 会替你处理**；从 `@nuxtjs/composition-api` 迁的，**Bridge 会自动提供一个 shim 覆盖"大多数"现有导入**，但有三个例外没有替代——**`withContext` 已移除**、**`useStatic` 已移除且目前没有替代**、**`reqRef`／`reqSsrRef` 曾废弃后已彻底移除**。另外两个 type-helper 桩函数 **`defineNuxtMiddleware` 与 `defineNuxtPlugin` 被移除**（改用 `@nuxt/types` 的类型断言），而 `useRoute` 的关键差别是**它不再返回 computed property**（`route.value.path` 要写成 `route.path`）。

categories:
  - 前端开发

tags:
  - nuxt4
  - bridge
  - composition-api
  - vue-demi
  - 升级迁移
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

在 Nuxt 2 里用过 Composition API 的项目，装 Bridge 之后不是"直接就能跑"，而是要先把旧的 API 层拆掉。这一页拆成两种情况，破坏性集中在第二种。

## 前提：Bridge 的实现是照 Nuxt 3 写的

> Nuxt Bridge provides access to Composition API syntax. It is specifically designed to be aligned with Nuxt 3. Because of this, there are a few extra steps to take when enabling Nuxt Bridge, if you have been using the Composition API previously.

"额外几步"具体是什么，`## Remove Modules` 先给两条：

- Remove `@vue/composition-api` from your dependencies.
- Remove `@nuxtjs/composition-api` from your dependencies (**and from your modules in `nuxt.config`**).

**第二条要求删两处**：依赖里，以及配置里 `modules` 数组中那一项。

## 情况一：只用过 @vue/composition-api

官方说这条路"very straightforward"，实质动作只有一条：

```diff
- import Vue from 'vue'
- import VueCompositionApi from '@vue/composition-api'
-
- Vue.use(VueCompositionApi)
```

**把手动注册 Composition API 的那个插件删掉，因为 Bridge 会替你处理这件事。** 第二条是可选的：**其余部分你什么都不用做；但如果你愿意，也可以删掉从 `@vue/composition-api` 的显式 import，改依赖 Bridge 的自动导入。**

## 情况二：从 @nuxtjs/composition-api 迁

> Nuxt Bridge implements the Composition API slightly differently from `@nuxtjs/composition-api` and provides different composables (designed to be aligned with the composables that Nuxt 3 provides).
>
> Because some composables have been removed and don't yet have a replacement, this will be a slightly more complicated process.

难点就在最后一句：**有些 composable 被删掉且暂时没有替代**。

### shim 覆盖大多数，三个例外

第一个动作是**把 `@nuxtjs/composition-api/module` 从 `buildModules` 里移除**，而官方特别说明：**不必立刻改导入**，因为 Bridge 会自动提供一层 shim：

> You don't have to immediately update your imports yet - Nuxt Bridge will automatically provide a 'shim' for most imports you currently have, to give you time to migrate to the new, Nuxt 3-compatible composables, **with the following exceptions:**

三个例外逐个列在下面：

| 例外 | 官方说明 | 去处 |
| --- | --- | --- |
| `withContext` | has been removed | 见 nuxt3-compatible 那页的 "useContext and withContext" 一节 |
| `useStatic` | has been removed，**"There is no current replacement."** | 有实际需求可以自己开讨论 |
| `reqRef`／`reqSsrRef` | **which were deprecated, have now been removed entirely** | 按 nuxt3-compatible 那页的 "ssrRef and shallowSsrRef" 一节替换 |

这三行是 shim 帮不了你的地方——**装了 Bridge 之后仍报某个 composable 不存在，先回来对这张表**。措辞上还能读出一层信息：`reqRef`/`reqSsrRef` 是"先废弃、如今彻底移除"，说明它们跨了两个阶段；而 `useStatic` 是"移除且目前没有替代"。

### 开关联动：bridge.capi

```ts [nuxt.config.ts]
import { defineNuxtConfig } from '@nuxt/bridge'

export default defineNuxtConfig({
  bridge: {
    capi: true,
    nitro: false, // If migration to Nitro is complete, set to true
  },
})
```

与上一节同款：`nitro` 那行是迁移进度指示。**开关之后官方还有一句**："For each other composable you are using from `@nuxtjs/composition-api`, follow the steps below." ——**下面的每一条都是按"你在用哪个 composable"来挑的**。

### 逐个 composable 的改法

**`useFetch`**：`$fetchState` 与 `$fetch` 这一对被移除，换成 `fetch` 与 `fetchState`：

```diff
const {
- $fetch,
- $fetchState,
+ fetch,
+ fetchState,
} = useFetch(() => { posts.value = await $fetch('/api/posts') })
```

**`defineNuxtMiddleware`** 与 **`defineNuxtPlugin`**：两个都是**类型辅助桩函数，现在已被移除**。去掉包装即可：

```diff
- import { defineNuxtMiddleware } from '@nuxtjs/composition-api`
- export default defineNuxtMiddleware((ctx) => {})
+ export default (ctx) => {}
```

```diff
- import { defineNuxtPlugin } from '@nuxtjs/composition-api'
- export default defineNuxtPlugin((ctx, inject) => {})
+ export default (ctx, inject) => {}
```

类型支持改由 **`@nuxt/types`** 提供，用断言写法：`export default <Middleware> function (ctx) { }` 与 `export default <Plugin> function (ctx, inject) {}`。插件那一条还配了一句提醒：**"While this example is valid, Nuxt 3 introduces a new defineNuxtPlugin function that has a slightly different signature."** ——**这里去掉包装后的写法是 Nuxt 2 形状，和 3 的那个同名函数不是一回事**。

**`useRouter` 与 `useRoute`**：Bridge 提供直接替代，唯一关键差别是**`useRoute` 不再返回 computed property**：

```diff
- import { useRouter, useRoute } from '@nuxtjs/composition-api'

  const router = useRouter()
  const route = useRoute()

- console.log(route.value.path)
+ console.log(route.path)
```

**这一处最容易改漏**：`.value` 去掉之后，凡是把 route 当 ref 用的代码都要扫一遍。

## 自查

- 先删两个包：`@vue/composition-api`、`@nuxtjs/composition-api`（后者**连 `modules` 里那项一起删**）。
- 情况一只删手动注册那几行；显式 import 可以留给自动导入（可选）。
- 情况二：**shim 覆盖大多数导入，但 `withContext`、`useStatic`、`reqRef`/`reqSsrRef` 三个例外帮不了**（`useStatic` 明确"目前没有替代"）。
- `$fetch`/`$fetchState` → `fetch`/`fetchState`；`defineNuxtMiddleware`/`defineNuxtPlugin` 两个桩函数移除，类型改用 `@nuxt/types` 断言。
- 去掉包装后的 `defineNuxtPlugin((ctx, inject) => {})` 是 Nuxt 2 形状，**与 Nuxt 3 的同名函数签名不同**。
- `useRoute` 不再返回 computed property，**`route.value.path` 要改成 `route.path`**。

## 下一篇

下一篇 621 读 Bridge 下的插件与中间件：插件**只接一个参数 `nuxtApp`**，但有一条例外必须记住——**`defineNuxtRouteMiddleware` 在 `app/middleware` 目录之外不被支持**；以及两条硬性警告：**用 `nuxtApp.vueApp` 注册插件/指令/mixins/组件要自己保证不会被安装多次，否则可能内存泄漏**；`definePageMeta` 需要先开 `macros.pageMeta`，**且在 Bridge 下只支持 `middleware` 与 `layout` 两个键**。

## 本文依据

- https://nuxt.com/docs/4.x/bridge/bridge-composition-api（页标题 "Legacy Composition API" 与"与 Nuxt 3 对齐所以要额外几步"的开场段；`## Remove Modules` 两条（含"以及 `nuxt.config` 里的 modules"）；`## Using @vue/composition-api` 的 "very straightforward"、删手动注册的 diff、"there is nothing you need to do"与可选改用自动导入那句；`## Migrating from @nuxtjs/composition-api` 两句定性；`### Remove @nuxtjs/composition-api/module from your buildModules` 的 shim 段与三个例外（`withContext`、`useStatic` 无替代、`reqRef`/`reqSsrRef` 废弃后彻底移除）及两处指向 nuxt3-compatible 那页的链接；`### Set bridge.capi` 代码块与"按你在用的 composable 逐条处理"句；`### useFetch` 的 diff；`### defineNuxtMiddleware` 与 `### defineNuxtPlugin` 两段"桩函数已移除"、两处 diff、`@nuxt/types` 的 `<Middleware>`／`<Plugin>` 写法与那条 `::warning`；`### useRouter and useRoute` 段与 `route.value.path`→`route.path` 的 diff）
