---
url: /posts/f01791267de5812698fb8a7043d5ea65/
title: error.vue：它不是路由、却渲染服务端致命错误
date: 2027-01-29T10:00:00+08:00
updated: 2027-01-29T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：服务端一个致命错误（未处理的、或客户端 fatal:true 的）会让 Nuxt 渲染一整页错误页，那个可自定义的页面就是 error.vue。它有几个反直觉的点：名字叫"页"但不是路由、不能放 pages 目录、不能用 definePageMeta；只有一个 prop 叫 error（类型 NuxtError）；能用布局但得自己套 NuxtLayout 并给 name；自定义字段不会保留、要放进 error 的 data。这一篇把这些讲清，并对齐服务端错误的渲染规则：Accept: application/json 或（cv5 下）服务端路由错误回 JSON，其余触发全屏错误页；清除用 clearError 可带 redirect。

categories:
  - 前端开发

tags:
  - nuxt4
  - error-vue
  - NuxtError
  - 致命错误
  - clearError
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

前面几篇讲服务端怎么抛错（createError、requireUserSession）。这一篇讲错误**抛上去之后由谁渲染**——`error.vue`。

## 它是什么、什么时候出现

官方定位：

> During the lifespan of your application, some errors may appear unexpectedly at runtime. In such case, we can use the `error.vue` file to override the default error files and display the error nicely.

它在**致命错误**时被渲染。什么算致命、会渲染它还是回 JSON，error-handling 那页定了规则：

> When Nuxt encounters a fatal error (any unhandled error on the server, or an error created with `fatal: true` on the client) it will either render a JSON response (if requested with `Accept: application/json` header) or trigger a full-screen error page. With `future.compatibilityVersion: 5`, a page error always triggers the error page, and an error thrown in a server route or server middleware returns a JSON response.

三条：

- 服务端任何未处理错误、或客户端 `fatal: true` 创建的错 → 致命。
- 带 `Accept: application/json` 请求头 → 回 JSON；否则触发全屏错误页。
- 开了 `compatibilityVersion: 5`：**页面**错误总触发错误页、**服务端路由/中间件**抛的错总回 JSON 响应（服务端接口不再顺带渲染一页 HTML）。

## 最反直觉的三点：它不是路由

官方 note 一次点了三个"像路由但都不是"：

> Although it is called an 'error page' it's not a route and shouldn't placed in your `~/pages` directory. For the same reason, you shouldn't use `definePageMeta` within this page. That being said, you can still use layouts in the error file, by utilizing the `NuxtLayout` component and specifying the name of the layout.

- 叫"error page"，但**不是路由**、不放 `app/pages/`（放在和 app.vue 同级的源目录根）。
- 因为不是路由，**不能用 `definePageMeta`**（那是页面元数据）。
- 想用它**得手动套 `NuxtLayout` 并给 name**（它不自动走布局解析）。

## 唯一的 prop：error

官方："The error page has a single prop - `error` which contains an error for you to handle." 最小写法：

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

`error` 的类型是 `NuxtError`（从 `#app` 导入），官方给了它的全部字段：

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

注意最后两个 `statusCode`/`statusMessage` 是旧版遗留（deprecated），新代码用 `status`/`statusText`。`cause` 只在开发暴露（234、308）。

## 自定义字段会丢：放进 data

这条最容易栽。官方：

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

NuxtError 只有上面那张表里列的字段。你直接在错误对象上挂自定义键（`{ status: 404, myCustomField: true }`），到 error.vue 里**读不到**——它不在 NuxtError 形状里、会被丢。要把自定义信息带过来，塞进 `data`（`error.data.myCustomField`）。这也和 309 那条"API 错误自定义数据走 data、前端在 `error.value.data.data` 取"呼应。

## 清掉错误页

error-handling 那页：错误页 ready 移除时调用 `clearError`，它接受一个可选的跳转路径：

> When you are ready to remove the error page, you can call the `clearError` helper function, which takes an optional path to redirect to (for example, if you want to navigate to a 'safe' page).

官方 error.vue 示例里的用法就是 `clearError({ redirect: '/' })`。另外 285 提过那条坑——插件抛错后不清错误就不会重跑，所以错误态下别默认依赖插件的 `$route`/`useRouter`。

## 自查

- error.vue 不是路由：别放 `app/pages/`、别用 `definePageMeta`；要用布局就手动套 `NuxtLayout` + `name`。
- 只有一个 prop `error`（`NuxtError` 从 `#app` 导）；新代码用 status/statusText，statusCode/statusMessage 是遗留。
- 自定义字段会丢，必须塞进 `data`。
- 渲染还是回 JSON：`Accept: application/json` 或（cv5）服务端路由错误回 JSON，其余全屏错误页；清除用 `clearError({ redirect })`。

## 下一篇

下一篇 329 讲 serverHandlers 与 devServerHandlers：怎么在 nuxt.config 里手动登记服务端处理程序（含 route 的 rou3 约定、method、middleware、lazy），以及它和目录自动注册的关系。

## 本文依据

- error.vue（`directory-structure/app/error`）：定位句；note（"it's not a route and shouldn't placed in your ~/pages directory ... you shouldn't use definePageMeta ... you can still use layouts ... utilizing the NuxtLayout component and specifying the name"）；"The error page has a single prop - error"；`error.vue` 示例（`import type { NuxtError } from '#app'`、`defineProps<{ error: NuxtError }>()`、`error.status`）；NuxtError 字段表（status/fatal/unhandled/statusText/data/cause + legacy statusCode/statusMessage）；"custom fields ... will be lost; you should assign them to data instead" + `throw createError({ status, statusText, data: { myCustomField } })`。
- Error Handling（`getting-started/error-handling`）：fatal 错误 → JSON 或全屏错误页、cv5 页面错误总触发错误页、服务端路由/中间件错误回 JSON（note）；clearError 可带 redirect。
