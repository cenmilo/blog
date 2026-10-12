---
url: /posts/39e988cc831b4a53fa4a81f74a19f9cb/
title: 路由与类型的八个实验开关：strictRouteTypes 必须配 routeTypedFetch
date: 2027-06-03T10:00:00+08:00
updated: 2027-06-03T10:00:00+08:00
author: cmdragon

summary:
  摘要：这一组的依赖关系最密：`strictRouteTypes` 一节第一句就写着它**需要 `routeTypedFetch`**；`extractSerializablePageMeta` 在 `scanPageMeta` 为 `false` 时**完全无效**；`extraPageMetaExtractionKeys` 只在开了 `scanPageMeta` 之后才有意义。类型侧的收益也写得具体——`routeTypedFetch` 打开后 handler 返回的 `Date` 在调用点是 `string`、校验过的 `body` 不能省略也不能给错形状、路由不响应的方法会被拒绝、`params` 不再接受。

categories:
  - 前端开发

tags:
  - nuxt4
  - 实验特性
  - 类型安全
  - 路由
  - page metadata
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## `typedPages`：类型化路由的入口

一句话是"启用新的实验性类型路由"。开了以后，文档点名的可直接类型化用法包括 `navigateTo`、`<NuxtLink>`、`router.push()` 等等；还额外给了一条页面内的用法：`const route = useRoute('route-name')` 能拿到带类型的参数。**这一节没有列出它不覆盖的 API**，也没有说与其他类型开关的关系。

## `routeTypedFetch`：类型来源换了地方

它的定位原文：让 `$fetch` 与 `useFetch` 的类型来自**你的服务端构建器报告的路由**，而不是来自 nitro 通过 `InternalApi` 接口贡献给 `ServerRoutes` 的那些响应。

生成的路由集里带着 handler 校验过的 `body`、`query`、`headers` 以及响应，**并按数据上线时的形状来定类型**。文档列出的六条后果：

- handler 返回的 `Date` 在调用点是 `string`
- handler 校验过的 `body` 不能省略，也不能给错的形状
- 该路由不响应的方法会被拒绝
- `params` 不再被接受，因为它从来只是 `query` 的别名
- 手工增强 `ServerRoutes` 声明的路由**需要重写**，因为两种对路由的描述不同（指向升级指南）
- 用 `Endpoint`（从 `nuxt/app` 导入）按重写后的形状声明的路由，**在这个开关开着和关着时都会被正确定型**，所以可以提前改

它是 opt-in 的，而且 **`future.compatibilityVersion: 5` 会启用它**。显式设置的用途是"无论兼容版本如何都选定型这一边"，让你能在迁到 Nuxt 5 之前先适应；反过来说也可以带着 `routeTypedFetch: false` 升到 Nuxt 5、之后再单独接。

那一条 note 交代了寿命：**这个选项只在两套定型并存期间存在；从 Nuxt 5 起生成的路由集是唯一来源，选项就没了。**

## `strictRouteTypes`：三种取值，各自拒绝什么

第一句是依赖声明：拒绝那些没有服务端路由应答的请求；**需要 `routeTypedFetch`**。取值三种：

- `false`（默认）—— Nuxt 认不出的路径被接受并解析成 `unknown`，任何方法都不拒绝。理由写得很清楚：**Nuxt 看不见所有可能被应答的方式**，nitro 中间件、`routeRules` 代理、catch-all handler 都能应答一个没有任何路由描述的路径。
- `true` —— 只接受你的服务端构建器报告的路由，且只接受它们应答的方法。
- `'isomorphic'` —— 同 `true`，但把你的页面也算作返回 `string` 的 GET 路由，于是 `$fetch('/about')` 由 Vue 路由要服务的那个路由来定型，而不是被拒绝。

开了之后拼错的路径和不支持的方法都是错误，文档给的例子带注释：

```ts
// server/api/todos.post.ts answers POST only
await $fetch('/api/todos', { method: 'POST' })
await $fetch('/api/todo', { method: 'POST' })
//           ^ no POST route matches '/api/todo'
await $fetch('/api/todos')
//           ^ no GET route matches '/api/todos'
```

三类例外也讲明了：**绝对 URL**（`$fetch('https://example.com')`）与**运行时拼出来的路径**仍然被接受（后者解析成 `unknown`），因为那不是 Nuxt 能描述的；**具名响应类型**（`await $fetch<{ id: string }>(...)`）等于放弃推断、也就放弃检查。`baseURL` 会先解析进路径再匹配，所以经由它发出的请求是按它实际要的那个路径来检查的。

还有一条单独 note 很容易踩：**`public/` 目录里的文件不是服务端构建器报告的路由**，所以开了 `strictRouteTypes` 之后请求它们会报错——要么带显式响应类型去取，要么别开。

最后一句是适用边界：**只在你的路由可枚举的地方设置它**。有一个 catch-all 页面的应用配 `'isomorphic'` 会匹配所有路径，结果正确，但这个选项也就约束不了任何东西。

## `scanPageMeta`：构建期只读六个键，而且只读静态值

原文说 Nuxt 会在构建期把 `definePageMeta` 里的部分路由元数据暴露给模块，**具体是 `alias`、`name`、`path`、`redirect`、`props`、`middleware` 六个**。两条限制：

- **只对静态值或字符串/数组生效**，变量与条件赋值不行，并指向原始 issue #24770。
- 时机：默认在**所有路由于 `pages:extend` 注册完之后**才扫描，然后调用 `pages:resolved` 钩子。

这也解释了为什么第 17-19 行的 `pages:resolved` 那行在构建期钩子表里——想读扫描结果要在它之后。可以关掉（`scanPageMeta: false`），文档给的关闭理由是"如果它给你的项目带来问题"。

## `extraPageMetaExtractionKeys`：把额外的键一起抽出来

`definePageMeta()` 是收集页面构建期元数据的手段，而 **Nuxt 自己提供的是一个固定键集合**，用来驱动重定向、页面别名、自定义路径这些内部特性。这个选项让你**在使用 `scanPageMeta` 时额外抽取指定的键**：

```vue
<script lang="ts" setup>
definePageMeta({
  foo: 'bar',
})
</script>
```

```ts
export default defineNuxtConfig({
  experimental: {
    extraPageMetaExtractionKeys: ['foo'],
  },
  hooks: {
    'pages:resolved' (ctx) {
      // ✅ foo is available
    },
  },
})
```

这段是官方原样，两条信息藏在里面：读取点是 `pages:resolved` 这个 `hooks` 映射；如果你是在模块里用它，**文档建议同时给自己的键增强 `NuxtPage` 的类型**。

## `extractSerializablePageMeta`：每个页面少一个模块

默认在 `future.compatibilityVersion` 为 `5` 或更高时开启。机制是：默认情况下 Nuxt 构建期只读固定列表里的 `definePageMeta()` 键（加上 `extraPageMetaExtractionKeys` 里的），**其余都在运行时从一个"每页一个"的独立模块解析**；开了这个选项，Nuxt 会把**每个可 JSON 序列化的属性直接写进生成的路由记录**。

收益那句很具体：当一个页面的 `definePageMeta()` 每个属性都能静态解析时，**该路由就不再 import 这个页面的 meta 模块**，于是 dev 的模块图里每页少一个模块。

两个边界：函数、变量引用、展开、计算键这些**无法序列化的属性不受影响**，仍然在运行时解析；**当 `scanPageMeta` 为 `false` 时这个选项完全无效**，因为那种情况下路由记录不会覆盖 meta 模块。

## `inlineRouteRules` 与 `normalizePageNames`

`inlineRouteRules` 用 `defineRouteRules` **在页面级别定义路由规则**，原文一句关键：匹配出来的路由规则**是基于页面的 `path` 生成的**。

`normalizePageNames` 解决的是一个很小的具体问题：默认 Vue 按文件名给组件命名，于是 `pages/foo/index.vue` 与 `pages/bar/index.vue` **组件名都叫 `index`**，这让按名字的 `<KeepAlive>` 过滤不可靠。开启后页面组件按路由命名（`foo`、`bar`），于是可以直接用 `<KeepAlive>` 的 `include`/`exclude` 而不必给每个页面手写 `defineOptions({ name: '...' })`：

```vue [app.vue]
<template>
  <NuxtPage :keepalive="{ include: ['foo'] }" />
</template>
```

它在 `future.compatibilityVersion` 设为 `5` 或更高时启用，也可以显式关掉。

## 依赖关系一览

| 选项 | 依赖或被依赖 |
| --- | --- |
| `strictRouteTypes` | 需要 `routeTypedFetch` |
| `extraPageMetaExtractionKeys` | 只在 `scanPageMeta` 生效时有意义 |
| `extractSerializablePageMeta` | `scanPageMeta: false` 时无效；`compatibilityVersion: 5` 时默认开 |
| `routeTypedFetch` | `future.compatibilityVersion: 5` 会启用它；Nuxt 5 起选项消失 |
| `normalizePageNames` | `compatibilityVersion: 5` 时启用 |
| `typedPages` | 页面内类型化参数用 `useRoute('route-name')` |

这一页里**没有任何一处说明 `typedPages` 与 `routeTypedFetch` 之间的关系**，一个管导航类型、一个管 fetch 类型，能不能同开文档未说明。

下一篇看组件与水合那一组七个：`componentIslands`、`clientFallback`、`lazyHydration`、`asyncEntry`，以及 `templateImportResolution` 那条"关掉以后可能修好解析问题"的用法。

## 本文依据

- https://nuxt.com/docs/4.x/guide/going-further/experimental-features
