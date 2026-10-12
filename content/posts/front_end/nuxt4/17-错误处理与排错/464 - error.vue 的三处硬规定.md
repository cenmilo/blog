---
url: /posts/bc89e81d0c8dd6befe0badaa8cd90dc3/
title: error.vue 的三处硬规定
date: 2027-03-15T11:00:00+08:00
updated: 2027-03-15T11:00:00+08:00
author: cmdragon

summary:
  摘要：`error.vue` 用来覆盖框架默认的错误页，位置在**应用源码目录里、和 `app.vue` 同级**。官方对它有三条写得明明白白的规定：第一，它虽然叫"错误页"，但**不是路由**，不该放进 `~/pages`，同一理由也**不该在里面用 `definePageMeta()`**；第二，这个文件**只有一个 prop**，就是 `error`，模板里那句 `{{ error.status }}` 读的是它的字段；第三，错误对象的字段是按类型收敛的——**如果你有自定义字段，它们会丢失，应该放进 `data`**。复用布局是被明确允许的：用 `NuxtLayout` 并指定布局名即可。

categories:
  - 前端开发

tags:
  - nuxt4
  - error.vue
  - 错误页
  - NuxtError
  - 目录结构
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

接口 404 的时候，用户看到的是一页灰底蓝字的默认错误信息。你想换成自家样式，于是建了 `error.vue`，顺手在里面写 `definePageMeta({ layout: 'app' })`——**这一行是官方明确禁止的**。

这一篇把 `error.vue` 的边界讲清，三处规定都来自同一页原文。

## 它是什么、放在哪

官方定性：

> During the lifespan of your application, some errors may appear unexpectedly at runtime. In such case, we can use the `error.vue` file to override the default error files and display the error nicely.

**作用是覆盖默认的错误文件**。位置在错误处理那一页写得具体：

> Customize the default error page by adding `~/error.vue` in the source directory of your application, alongside `app.vue`.

**和 `app.vue` 同级**，在应用源码目录里。

## 规定一：它不是路由

官方用一条 `::note` 说了三件事：

> Although it is called an 'error page' it's not a route and shouldn't be placed in your `~/pages` directory. For the same reason, you shouldn't use `definePageMeta` within this page. That being said, you can still use layouts in the error file, by utilizing the `NuxtLayout` component and specifying the name of the layout.

拆开：

| 说法 | 内容 |
| --- | --- |
| 不是路由 | 尽管名字叫"错误页"，它不是 route |
| 不放 `pages/` | 不该放进 `~/pages` 目录 |
| 不用 `definePageMeta` | **同一理由**：不该在这个文件里用 `definePageMeta` |
| 可以用布局 | 用 `NuxtLayout` 并**指定布局名**即可复用布局 |

第三条的逻辑要讲清：`definePageMeta` 是**页面**的元数据（057 那批讲过它由路由系统读取并静态提取），而 `error.vue` 根本不在路由表里，写了也没人去读它。想套布局，官方给的替代就是最后那句——直接在模板里用 `<NuxtLayout name="...">`。

## 规定二：只有一个 prop

官方原话一句：

> The error page has a single prop - `error` which contains an error for you to handle.

官方给的完整文件长这样：

```vue [error.vue]
<script setup lang="ts">
import type { NuxtError } from '#app'

const props = defineProps<{ error: NuxtError }>()
</script>

<template>
  <div>
    <h1>{{ error.status }}</h1>
    <NuxtLink to="/">Go back home</NuxtLink>
  </div>
</template>
```

三处细节值得照抄：**类型从 `#app` 导入**（`import type { NuxtError } from '#app'`）、模板里读的是 `error.status`（**不是 `statusCode`**）、以及"回首页"用的是 `<NuxtLink>`。

两个页面的 props 写法不一样，这点要留意——`error.vue` 那一页用的是泛型形式 `defineProps<{ error: NuxtError }>()`，而错误处理那一页给的是带断言的形式：

```ts
const props = defineProps({
  error: Object as () => NuxtError,
})
```

后者是 430/056 那批解释过的"非原始类型 prop 需要 `as` 断言"的写法。两种都在官方示例里出现，功能上等价，选一种保持一致即可。

## 规定三：字段按类型收敛

这一页给的错误对象形状是八个字段：

```ts
interface NuxtError {
  status: number
  fatal: boolean
  unhandled: boolean
  statusText?: string
  data?: unknown
  cause?: unknown
  // legacy/deprecated equivalent of `status`
  statusCode: number
  // legacy/deprecated equivalent of `statusText`
  statusMessage?: string
}
```

其中两个字段被官方**注释成 legacy/deprecated 等价物**：`statusCode` 对应 `status`、`statusMessage` 对应 `statusText`。**新代码要读 `status` 与 `statusText`**——这也和 461 记录的三处不一致对上了：只有这一页列了 `unhandled` 和这两个旧字段，也只有这一页**没有列 `message`**。

紧跟着那句是第三条规定的正文：

> If you have an error with custom fields they will be lost; you should assign them to `data` instead:

```ts
throw createError({
  status: 404,
  statusText: 'Page Not Found',
  data: {
    myCustomField: true,
  },
})
```

**自定义字段会丢，要放进 `data`**。这条和 462 记的接口侧规则连起来才是完整的：`data` 是能穿过错误响应的容器，客户端用 `useFetch` 处理时在 `error.value.data.data` 取到你放的对象。

## 什么时候它会被渲染

这一页只说了"覆盖默认错误页"，没说触发条件。触发条件在别处，正好接上前两篇：

- 服务端任何未处理错误，或客户端 `fatal: true` 创建的错误 → 触发全屏错误页（460 的 fatal 定义）。
- `showError` 会触发全屏错误页，`createError` 在服务端 throw 也会（462）。
- 清掉它用 `clearError`，可以带 `redirect`（463）。
- 渲染错误页是一次独立加载，全站中间件会再跑一遍（461）。

## 自查

- 位置：应用源码目录、与 `app.vue` 同级；不是路由，所以不进 `pages/`、不用 `definePageMeta()`。
- 想带布局就在模板里用 `NuxtLayout` 并指定布局名，这是官方给的唯一合法路径。
- 只有 `error` 一个 prop；类型从 `#app` 导入，读字段用 `status`/`statusText`（`statusCode`/`statusMessage` 是官方标注的旧等价物）。
- 自定义字段会丢，必须塞进 `data`。
- 它何时出现由 fatal 规则决定，不是由这个文件决定。

## 下一篇

下一篇 465 把"错误页里到底能用什么"这一层摊开：官方**没有给可用组合式函数的清单**，只写明了两条限制（依赖插件的东西要先检查、中间件会再跑一遍），加上从两个示例里能直接确认的三样（`<NuxtLink>`、`NuxtLayout`、`clearError`）。其余要靠自己在 dev 里试一遍。

## 本文依据

- `error.vue` 那一页：定性句（override the default error files、display the error nicely）、完整示例（`import type { NuxtError } from '#app'`、`defineProps<{ error: NuxtError }>()`、`error.status`、`<NuxtLink to="/">`）、`::note` 三条（不是路由且不放 `~/pages`、同一理由不用 `definePageMeta`、可用 `NuxtLayout` 指定布局名）、"single prop - `error`"、八字段 `NuxtError` 接口含两个标注 legacy/deprecated 的等价字段、"自定义字段会丢失、应放进 `data`"及 `createError({ data: { myCustomField: true } })` 示例（`directory-structure/app/error`）
- 错误处理页：`~/error.vue` 与 `app.vue` 同级的位置句、`defineProps({ error: Object as () => NuxtError })` 形式的示例与 `clearError({ redirect: '/' })`、fatal 与全屏错误页的 `::note`（`getting-started/error-handling`）
- `createError` 的参数与 `data`、`error.value.data.data`（`api/utils/create-error`）
