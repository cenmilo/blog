---
url: /posts/ed10f2cbc66570b6b0e153f4b15a3635/
title: 缓存的两把新开关：granularCachedData 与 purgeCachedData
date: 2026-12-29T11:00:00+08:00
updated: 2026-12-29T11:00:00+08:00
author: cmdragon

summary:
  摘要：215 讲过 key 决定"数据归谁"，这一篇讲"数据活多久"，靠两个实验开关。**`purgeCachedData`**：定位句 "Whether to clean up Nuxt static and asyncData caches on route navigation."，解释句 "Nuxt will automatically purge cached data from `useAsyncData` and `nuxtApp.static.data`. This helps prevent memory leaks and ensures fresh data is loaded when needed, but it is possible to disable it."，**默认开启**，官方附 PR #31379 作为实现细节出处。**`granularCachedData`**：定位句 "Whether to call and use the result from `getCachedData` when refreshing data for `useAsyncData` and `useFetch` (whether by `watch`, `refreshNuxtData()`, or a manual `refresh()` call."，同样**默认开启**，PR #31373。这两个开关一起出现的地方最有价值：升级指南里那条 **"Update `getCachedData` implementations"** 给了迁移写法——第三个参数 `ctx` 带 `cause`（`'initial' | 'refresh:hook' | 'refresh:manual' | 'watch'`），官方案例是 `if (ctx.cause === 'refresh:manual') return undefined`（"Example: Don't use cache on manual refresh"），并补了一句 "**Alternatively, for now, you can disable this behaviour with** `granularCachedData: false, purgeCachedData: false`"。另外两处相关：响应式 key 换掉时官方写了清理条件（"the old data will be cleaned up if no other components use it"）；`useState` 的 `resetOnClear` 决定 `clearNuxtState` 是回到初值还是 `undefined`，**`compatibilityVersion: 5` 下默认 `true`**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 缓存
  - getCachedData
  - 实验特性
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先说这一对开关为什么会咬人：它们都**默认开启**，也就是说升级之后行为变了，而你的 `getCachedData` 可能还是按老规矩写的。**症状是"手动刷新拿不到新数据"**——缓存被当成有效结果返回了。

## 两把开关各自管什么

| 开关 | 官方定位句 | 默认 |
| --- | --- | --- |
| `experimental.purgeCachedData` | "Whether to clean up Nuxt static and asyncData caches on route navigation." | 开启 |
| `experimental.granularCachedData` | "Whether to call and use the result from `getCachedData` when refreshing data for `useAsyncData` and `useFetch` (whether by `watch`, `refreshNuxtData()`, or a manual `refresh()` call." | 开启 |

`purgeCachedData` 那节的解释句给了目的（两个）：

> Nuxt will automatically purge cached data from `useAsyncData` and `nuxtApp.static.data`. This helps prevent memory leaks and ensures fresh data is loaded when needed, but it is possible to disable it.

关掉它怎么写，官方只给了一个配置块：

```ts twoslash [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    purgeCachedData: false,
  },
})
```

注意它清理的两个地方：`useAsyncData` 那一格与 `nuxtApp.static.data`（213 那条默认实现里"非水合期读的那个对象"正是它）。**换句话说，路由一跳，`static.data` 里的旧条目就没了**——如果你的"跨页复用缓存"逻辑依赖它，就会被这把开关影响。

`granularCachedData` 管的是**时机**：`watch`、`refreshNuxtData()`、手动 `refresh()` 这三种刷新发生时，还叫不叫 `getCachedData`、用不用它的结果。

## 升级指南里那段才是重点

这两把开关在文档里一起出现的地方是升级页，标题就叫 "Update `getCachedData` implementations"，给的迁移写法（照抄）：

```diff
 useAsyncData('key', fetchFunction, {
-  getCachedData: (key, nuxtApp) => {
-    return cachedData[key]
-  }
+  getCachedData: (key, nuxtApp, ctx) => {
+    // ctx.cause - can be 'initial' | 'refresh:hook' | 'refresh:manual' | 'watch'
+    
+    // Example: Don't use cache on manual refresh
+    if (ctx.cause === 'refresh:manual') return undefined
+    
+    return cachedData[key]
+  }
 })
```

这段我读到的三条信息：

1. **第三个参数 `ctx` 是这次请求的原因**，四个取值和 216 那张表一致。
2. 官方推荐的写法是**用 `cause` 判断要不要吃缓存**——`refresh:manual` 时返回 `undefined`，"没缓存"就等于强制走 handler。这正好补上 214 那句"refresh 用来手动刷新数据"在缓存层的落地。
3. 不想改代码的退路官方也写了：

> Alternatively, for now, you can disable this behaviour with:

```ts twoslash [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    granularCachedData: false,
    purgeCachedData: false,
  },
})
```

**"for now" 这个词出现在官方原话里**，所以我的判断是：这是临时兼容档，别当长期方案——两把都关掉意味着缓存既不按原因区分、也不随路由清理，时间越长越容易攒下过期数据。

## 默认实现长什么样，为什么它会"读不到"

213 抄过的那段仍然适用：

```ts
const getDefaultCachedData = (key, nuxtApp, ctx) => nuxtApp.isHydrating
  ? nuxtApp.payload.data[key]
  : nuxtApp.static.data[key]
```

> This only caches data when `experimental.payloadExtraction` in `nuxt.config` is enabled.

把三件事叠起来才是完整图：**默认实现在水合期读 payload、其它时候读 `static.data`；payload 那条通道由 `payloadExtraction` 决定；而 `static.data` 会被 `purgeCachedData` 在路由跳转时清掉。**所以"我以为缓存会跨页留着"这个期待，在默认配置下是有保质期的。

## 顺带：key 换掉之后旧数据去哪了

响应式 key 那段（官方示例注释里藏了一句清理规则）：

```ts
// Using a computed property as a key
const userId = ref('123')
const { data: user } = useAsyncData(
  computed(() => `user-${userId.value}`),
  () => fetchUser(userId.value),
)

// When userId changes, the data will be automatically refetched
// and the old data will be cleaned up if no other components use it
userId.value = '456'
```

那句 "**if no other components use it**" 我认为很关键：旧 key 的数据不是无条件删，**还有别的组件在用就不删**。这跟 215 的"共享"是一体两面——共享省了请求，但会让清理延后。

## 状态那一侧的同类开关

紧邻的另有一条，讲 `useState` 的清理语义：

> The `useState.resetOnClear` option controls whether `clearNuxtState` resets state to its initial value (provided by the `init` function of `useState`) instead of setting it to `undefined`. This defaults to `true` with `compatibilityVersion: 5`.

也就是说"清空状态"有两种含义：回到初值 或 变成 `undefined`，而默认值随兼容版本变化。**这一条和 `clearNuxtData`（229）不是同一个函数管的**，但症状很像（"清空后组件读到空"），所以我把它记在同一节里防混。

## 两张清单不一致：同 key 共享时能差什么

215 用的是 `useAsyncData` 那页的清单（必须一致 7 项 / 可以不同 7 项）。data-fetching 那页同题重写了一遍，**条目却不一样**：

| 页 | 必须一致 | 可以不同 |
| --- | --- | --- |
| `api/composables/use-async-data` | handler、`deep`、`transform`、`pick`、`getCachedData`、`default`、**`middleware`** | `server`、`lazy`、`immediate`、`dedupe`、`watch`、**`enabled`**、**`serialize`** |
| `getting-started/data-fetching` | handler、`deep`、`transform`、`pick`、`getCachedData`、`default`（**无 `middleware`**） | `server`、`lazy`、`immediate`、`dedupe`、`watch`（**无 `enabled`/`serialize`**） |

后一页是概览页，前一页是 API 参考页；`middleware`/`enabled`/`serialize` 这三个都带较新的版本徽标（v4.5/v4.6）。**我按 API 参考页为准**，但这是我自己选的判断，文档没有说哪份更新。另外 data-fetching 那页还多给了一条实用建议：

> If you need independent instances, use different keys:

```ts
// These are completely independent instances
const { data: users1 } = useAsyncData('users-1', (_nuxtApp, { signal }) => $fetch('/api/users', { signal }))
const { data: users2 } = useAsyncData('users-2', (_nuxtApp, { signal }) => $fetch('/api/users', { signal }))
```

## 自查

1. 两把开关分别管什么，默认都是什么状态？（答：`granularCachedData` 管刷新时叫不叫 `getCachedData`；`purgeCachedData` 管路由跳转时清不清 `useAsyncData` 与 `nuxtApp.static.data`；两者默认开启。）
2. 官方推荐的"手动刷新不吃缓存"写法是什么？（答：`getCachedData(key, nuxtApp, ctx)` 里 `if (ctx.cause === 'refresh:manual') return undefined`。）
3. 默认缓存实现在哪三种情况下可能读不到值？（答：`payloadExtraction` 关闭（含 `ssr: false` 强制关）、路由跳转后 `static.data` 被 purge、key 变化后旧条目在无其它使用者时被清理。）

## 下一篇

237 - 缓存命中时取数还跑吗：swr、isr 与 payload 文件

## 本文依据

- `guide/going-further/experimental-features`：`purgeCachedData` 一节（定位句、解释句、默认开启、配置块、PR #31379 read-more）；`granularCachedData` 一节（定位句、默认开启、配置块、PR #31373 read-more）；`payloadExtraction` 一节（235 已逐句引）；`useState.resetOnClear` 那句。
- `getting-started/upgrade`：「Update `getCachedData` implementations」整段 diff（含 `ctx.cause` 四值注释、"Don't use cache on manual refresh" 判例）与 "Alternatively, for now, you can disable this behaviour with:" 及两键配置块。
- `api/composables/use-async-data`：默认 `getCachedData` 实现与 `payloadExtraction` 限定句、`enabled`/`serialize`/`middleware` 三行原文与版本徽标、Shared State 两张清单（213/215 已逐句引）。
- `getting-started/data-fetching`：Reactive Keys 示例与"old data cleaned up if no other components use it"注释、Refresh and execute 段、Shared State 两张清单与"if you need independent instances, use different keys"及代码块（215/216 已逐句引）。
