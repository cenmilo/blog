---
url: /posts/b8bd1ce45779e712c180f63f80a0f865/
title: 702 - 数据与缓存的九个实验开关：payloadExtraction 的三种取值
date: 2027-06-03T09:00:00+08:00
updated: 2027-06-03T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一组九个开关，管的是"数据放在哪、什么时候清、什么时候重取"。`payloadExtraction` 有三种取值，默认是 `true`，`compatibilityVersion: 5` 时是 `'client'`，而**`ssr: false` 时会被强制成 `false`**。`purgeCachedData` 与 `granularCachedData` 都默认开，关掉后前者不再在导航时清 `useAsyncData` 与 `nuxtApp.static.data` 的缓存，后者不再在刷新时调用 `getCachedData`。`sharedPrerenderData` 打开时官方加了一条硬要求：**每个 key 必须始终解析到同一份数据**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 实验特性
  - 数据获取
  - 缓存
  - 预渲染
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## `payloadExtraction`：三种取值，一个强制规则

它控制的是**预渲染与缓存（ISR/SWR）页面的 payload 数据怎么送达**。三种取值原文逐条：

- `'client'` —— payload 内联在初次服务端渲染的 HTML 里，同时抽成 `_payload.json` 文件供客户端导航使用。
- `true` —— 初次服务端渲染与客户端导航**都**用独立的 `_payload.json` 文件。
- `false` —— 完全不抽；payload 始终内联在 HTML 里，**不生成 `_payload.json`**。

默认值是 `true`，设了 `compatibilityVersion: 5` 时是 `'client'`；**`ssr: false` 时被强制为 `false`**。文档把每种模式的后果指向预渲染那一节的 payload extraction 小节。

## `granularCachedData` 与 `purgeCachedData`：都默认开，关的是两件不同的事

`granularCachedData` 管的是"**刷新时还要不要调用并使用 `getCachedData` 的结果**"——刷新途径包括 `watch`、`refreshNuxtData()` 和手动 `refresh()`。默认开。关掉以后你在 `getCachedData` 里写的自定义缓存逻辑在刷新路径上就不再生效。

`purgeCachedData` 管的是"**导航时要不要清掉 Nuxt 的 static 与 asyncData 缓存**"。原文说明：Nuxt 会自动清除 `useAsyncData` 与 `nuxtApp.static.data` 里的缓存数据，这有助于**防止内存泄漏并确保按需加载新数据**，但可以关掉。默认开。

两者各自挂了一条指向实现细节的 GitHub 链接（分别是 PR #31373 与 #31379）——**这一节没有给这两项的行为表或配置项**，要精确判断只能读那两个 PR。

## `sharedPrerenderData`：收益大，但 key 必须唯一对应同一份数据

这一条讲预渲染。原文：Nuxt 会自动在预渲染的页面之间共享 payload 数据本身，对于**在不同页面抓同一份数据**、用了 `useAsyncData` 或 `useFetch` 的站点，这能带来显著的性能改善；可以关掉。

紧接着那句重要提醒才是重点：

> 开启这个特性时，尤其要确保你数据的每个唯一 key **总是解析到同一份数据**。

文档给了对照写法：

```ts
// This would be unsafe in a dynamic page (e.g. `[slug].vue`) because the route slug makes a difference
// to the data fetched, but Nuxt can't know that because it's not reflected in the key.
const route = useRoute()
const { data } = await useAsyncData(async (_nuxtApp, { signal }) => {
  return await $fetch(`/api/my-page/${route.params.slug}`, { signal })
})
// Instead, you should use a key that uniquely identifies the data fetched.
const { data } = await useAsyncData(route.params.slug, async (_nuxtApp, { signal }) => {
  return await $fetch(`/api/my-page/${route.params.slug}`, { signal })
})
```

不安全的那一段是**没给 key** 的写法：路径 slug 影响返回的数据，但 Nuxt 无法从 key 里知道这件事。安全的写法是把 `route.params.slug` 当 key 传进去。文档另外补了一句：**`useFetch` 应该会自动替你做这件事。**

## `renderJsonPayloads` 与 `parseErrorData`：两个默认开的开关

- `renderJsonPayloads`：允许以 JSON 渲染 payload，**支持把复杂类型复原（revivify）**。默认开，可以关。关掉意味着 payload 的复杂类型还原能力变了——这一节没有展开关之后的具体差异。
- `parseErrorData`：**渲染服务端错误页时要不要解析 `error.data`**。默认开。这决定了错误页里能不能拿到抛错时附带的 `data`。

## `pendingWhenIdle`：`pending` 这个 ref 什么时候为真

它控制 `useAsyncData` 与 `useFetch` 返回的 `pending`。默认 `false` 时的行为原文写得很具体：**`pending` 只在请求进行中为 `true`，与 `status === 'pending'` 一致；`status` 是 `idle` 时 `pending` 保持 `false`**——你能在 `{ immediate: false }` 或渲染期的 `{ server: false }` 上看到这个现象。

设成 `true` 的效果是：`status` 为 `idle` 且没有可用缓存数据时 `pending` 也为 `true`。文档给的选择标准是：**需要区分 `idle` 与"正在请求"的加载界面，就用 `status`**。

## `alwaysRunFetchOnKeyChange`：key 变了但还没触发过要不要跑

默认关。它管的是一个很窄的场景：`useFetch` 设了 `immediate: false` 且尚未被触发时，**key 变化要不要照样运行**。原文同时给出了既有规则：`immediate: true` 或已经触发过时，`useFetch` 与 `useAsyncData` 在 key 变化时总会运行。

## `extractAsyncDataHandlers`：把 handler 抽成独立 chunk

它把 `useAsyncData` 和 `useLazyAsyncData` 调用里的 handler 函数抽到单独的 chunk，用于改善代码分割与缓存效率。文档用 before/after 展示了变换的形状——内联的 `async () => { return await $fetch('/api/user') }` 变成 `() => import('/generated-chunk.js').then(r => r.default())`，收益表述是"能把数据获取逻辑拆出来，同时在需要时仍允许加载这段代码"。

它带了一条重要提示划定适用范围：

> 这个特性**只推荐用于启用了 payload 提取的静态构建**，且数据不需要在运行时重新获取的场景。

## 九个开关的默认状态一览

| 开关 | 默认 | 关掉/打开会改变什么 |
| --- | --- | --- |
| `payloadExtraction` | `true`（compatibilityVersion 5 时 `'client'`；`ssr: false` 强制 `false`） | payload 是内联还是抽成 `_payload.json` |
| `granularCachedData` | 开 | 刷新路径上还不调用 `getCachedData` |
| `purgeCachedData` | 开 | 导航时还清不清 static 与 asyncData 缓存 |
| `sharedPrerenderData` | 未声明 | 预渲染页面之间是否共享 payload 数据；开时 key 必须唯一对应同一份数据 |
| `renderJsonPayloads` | 开 | 是否以 JSON 渲染 payload 并复原复杂类型 |
| `parseErrorData` | 开 | 服务端错误页是否解析 `error.data` |
| `pendingWhenIdle` | `false` | `status` 为 `idle` 时 `pending` 是否为真 |
| `alwaysRunFetchOnKeyChange` | 关 | `immediate: false` 且未触发时，key 变化是否运行 |
| `extractAsyncDataHandlers` | 未声明 | handler 是否被抽成动态导入的 chunk；仅推荐用于静态构建＋payload 提取 |

同组的 `stripNeverHydratedData` 与水合策略联动（对 `hydrate-never` 懒水合的调用默认施加 `serialize: false`，把数据留在 `__NUXT_DATA__` 之外），`restoreState` 与 chunk 错误联动，这两条分别在组件篇与渲染篇里处理。

下一篇看路由与类型那一组八个：`typedPages`、`strictRouteTypes` 为什么必须成对开、`scanPageMeta` 只能读静态值这件事。

## 下一篇

下一篇 703：路由与类型的八个实验开关：strictRouteTypes 必须配 routeTypedFetch

## 本文依据

- https://nuxt.com/docs/4.x/guide/going-further/experimental-features
