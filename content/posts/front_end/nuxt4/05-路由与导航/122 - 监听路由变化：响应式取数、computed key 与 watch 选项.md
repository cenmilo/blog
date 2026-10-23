---
url: /posts/b5904ba8d60c85c40a3e4ff82ad217e9/
title: 122 - 监听路由变化：响应式取数、computed key 与 watch 选项
date: 2026-11-21T11:00:00+08:00
updated: 2026-11-21T11:00:00+08:00
author: cmdragon

summary: >-
  摘要："路由变了要重新取数"这件事，官方给的正解不是 `watch(route, ...)`，而是**把路由值做成响应式来源交给取数接口**。`useFetch` 那一节的原文是："**取数选项可以是响应式的，支持 `computed`、`ref` 与 computed getters。当一个响应式取数选项被更新时，它会使用更新后解析出的响应式值触发一次重新取数（refetch）。**"官方示例把这条写成最直观的形式：`useFetch('/api/search', { query: { q: searchQuery } })`，然后 `searchQuery.value = 'new search'` 后面注释写着 **`// triggers a refetch: /api/search?q=new%20search`**；反过来"**如果需要的话，你可以使用 `watch: false` 退出这一行为**"，示例里同样的赋值配上是 `// does not trigger a refetch`。`useAsyncData` 那一节还给了一个内置选项："**内置的 `watch` 选项允许在检测到任何变化时自动重跑取数函数**"，示例是把 sources 显式列成数组 `watch: [page]`，参数表的定义是 `MultiWatchSources`、"**要监听并自动刷新的响应式来源数组**"。第三条路更巧妙，是**把 key 本身做成响应式**："你可以使用一个 computed ref、普通 ref 或 getter 函数作为 key，从而允许在 key 变化时自动更新的动态取数"，官方示例正是从路由参数派生：`const route = useRoute()`、`const userId = computed(() => \`user-${route.params.id}\`)`、注释写明 **"当路由变化且 userId 更新时，数据会被自动重新取回"**。这三种触发在数据层是可区分的——`AsyncDataRequestContext` 里那个 `cause` 字段的取值是 `'initial' | 'refresh:manual' | 'refresh:hook' | 'watch'`。要控制代价，官方给了三个开关：`dedupe` 类型 `'cancel' | 'defer'`、默认 **`'cancel'`**，定义是"**同一时刻触发多次执行时的策略**"；`handler` 第二参数里的 `signal` 可以让请求可取消，官方说"**这在请求不再需要时取消它很有用，例如用户从页面导航离开**"；`enabled` 官方描述是一道屏障，"**在为 `false` 期间每一次执行都会被阻塞（初次取数、`execute`/`refresh` 以及 watch 触发）**"。最后还有一条组件侧的守卫可用：`onBeforeRouteUpdate` 的官方定义是"**允许在一个组件内注册一个路由守卫**"，官方页只给这一句并把语义交给 vue-router 文档（它与 `useRoute` 的 source 同在 `packages/nuxt/src/app/composables/router.ts`）。

categories:
  - 前端开发

tags:
  - nuxt4
  - 路由监听
  - 响应式取数
  - watch
  - useFetch

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


同一个页面换了参数（`/users/1` → `/users/2`），组件默认**不会重建**，于是"重新取数"这件事必须有人负责。官方的答案是：交给响应式系统，而不是手写一堆 `watch`。

## 先分清两件事

| 问题 | 由谁决定 |
| --- | --- |
| 换参数时，页面组件要不要**重建**？ | `key` / `pageKey`（092、116）：官方的写法是 `definePageMeta({ key: route => route.fullPath })` 或 `<NuxtPage :page-key="route => route.fullPath" />` |
| 组件不重建时，数据要不要**跟着换**？ | 本篇的三条响应式路径 |

这两条只要想反，就会出现两种相反的事故：设了随路径变化的 `key` 却指望 `watch` 保住状态（组件已经重建，`watch` 也重新注册了）；或者组件复用却不给任何响应式来源（`route.params` 变了但数据仍是旧的）。

121 说过 `useRoute()` 提供的字段本身**就是计算引用**——这是所有响应式写法能成立的前提。

## 路径一：把 ref 直接塞进取数选项

官方在 `useFetch` 那一节写的规则：

> **取数选项可以是响应式的，支持 `computed`、`ref` 与 computed getters。当一个响应式取数选项被更新时，它会使用更新后解析出的响应式值触发一次重新取数。**

```ts [app/pages/index.vue]
const searchQuery = ref('initial')
const { data } = await useFetch('/api/search', {
  query: { q: searchQuery },
})
// triggers a refetch: /api/search?q=new%20search
searchQuery.value = 'new search'
```

**`query` 的值是 ref，官方注释就是那行 "triggers a refetch"**。同样的写法可以直接换成路由参数（120 的官方示例已经把 `route.params.slug` 放进 URL）：

```ts
const route = useRoute()
const { data } = await useFetch('/api/search', {
  query: { q: route.params.keyword },
})
```

想退出这个行为，官方给了明确的开关：

> **如果需要的话，你可以使用 `watch: false` 退出这一行为。**

```ts [app/pages/index.vue]
const searchQuery = ref('initial')
const { data } = await useFetch('/api/search', {
  query: { q: searchQuery },
  watch: false,
})
// does not trigger a refetch
searchQuery.value = 'new search'
```

**这一格是"我只在初次取，之后别自己动"的正解**——不需要为了阻止自动重取而绕写法。

## 路径二：显式列出要监听的 sources

`useAsyncData` 那一节给的是同一个能力的另一种写法：

> **内置的 `watch` 选项允许在检测到任何变化时自动重跑取数函数。**

```vue [app/pages/index.vue]
<script setup lang="ts">
const page = ref(1)
const { data: posts } = await useAsyncData(
  'posts',
  (_nuxtApp, { signal }) => $fetch('https://fakeApi.com/posts', {
    params: {
      page: page.value,
    },
    signal,
  }), {
    watch: [page],
  },
)
</script>
```

参数表里的定义是：`watch` — `MultiWatchSources` — "**要监听并自动刷新的响应式来源数组**"。

与路径一的分工很清楚：**路径一是"选项里嵌了响应式值"，系统顺带就监听了；路径二是"handler 里读的是别的源，必须自己点名"**（这个例子里 `page.value` 出现在 handler 内部，所以必须写 `watch: [page]`）。

## 路径三：让 key 自己跟着路由变

官方在 Reactive Keys 一节给的做法，恰好是"路由变化自动重取"最短的一条：

> **你可以使用一个 computed ref、普通 ref 或 getter 函数作为 key，从而允许在 key 变化时自动更新的动态取数。**

```vue [app/pages/[id].vue]
<script setup lang="ts">
const route = useRoute()
const userId = computed(() => `user-${route.params.id}`)

// When the route changes and userId updates, the data will be automatically refetched
const { data: user } = useAsyncData(
  userId,
  () => fetchUserById(route.params.id),
)
</script>
```

那条官方注释值得复读：**"当路由变化且 userId 更新时，数据会被自动重新取回。"**

这条为什么值得单独列：**key 变了意味着这是另一份数据**（不同缓存条目），而不只是"同一份数据刷新一次"。`/users/1` 与 `/users/2` 各占一个 key，来回切还能命中缓存。

## 三种触发是能被区分的

`AsyncDataRequestContext` 里的 `cause` 字段（官方类型）：

```ts
type AsyncDataRequestContext = {
  /** The reason for this data request */
  cause: 'initial' | 'refresh:manual' | 'refresh:hook' | 'watch'
}
```

| 取值 | 对应这次执行是 |
| --- | --- |
| `'initial'` | 初次 |
| `'refresh:manual'` | 你调了 `refresh`/`execute` |
| `'refresh:hook'` | 由钩子触发 |
| `'watch'` | **由监听触发**（本篇三条路径里凡是走 watch 的都在这一格） |

需要按"这次为什么要取"分岔逻辑（例如 watch 触发时要清掉分页状态），`cause` 就是官方的入口——它出现在 `getCachedData(key, nuxtApp, ctx)` 与中间件这类回调的上下文里。

## 三个控制代价的开关

| 开关 | 官方定义 | 默认 |
| --- | --- | --- |
| `dedupe` | "**同一时刻触发多次执行时的策略**"，类型 `'cancel' \| 'defer'` | **`'cancel'`** |
| `signal` | handler 第二参数提供，让请求可中断；官方："**这在请求不再需要时取消它很有用，例如用户从页面导航离开**" | —— |
| `enabled` | "**一道屏障，用于控制 handler 是否允许运行。在为 `false` 期间，每一次执行都会被阻塞（初次取数、`execute`/`refresh` 以及 watch 触发）**"，从 `true` 切到 `false` 会取消进行中的请求但不清空 `data`；重新启用本身不会重取 | `true` |

`dedupe` 在监听场景里最实用：输入框里连续改 `searchQuery`，每一次都会触发一次 refetch——**默认的 `'cancel'` 会把前一次取消掉**，官方那个 signal 例子也配了注释 `refresh() // will actually cancel the $fetch request (if dedupe: cancel)`。

```ts
const { data } = await useFetch('/api/search', {
  query: { q: searchQuery },
  dedupe: 'cancel',
})
```

配合可取消的 handler：

```ts [app/pages/index.vue]
const { data, error } = await useAsyncData(
  'users',
  (_nuxtApp, { signal }) => $fetch('/api/users', { signal }),
)
```

## 组件侧：注册一个路由守卫

如果你要的不是"重新取数"而是"离开/更新前拦一道"，官方给的组件内接口是这两个（`onBeforeRouteLeave` 与 `onBeforeRouteUpdate`）。`onBeforeRouteUpdate` 那一节的定义只有一句：

> **`onBeforeRouteUpdate` 组合式函数允许在一个组件内注册一个路由守卫。**

具体参数与语义官方直接交给 vue-router 文档（这一节只有一个 read-more 卡片，指向 `onbeforerouteupdate`）。值得注意的是它的 Source 与 `useRoute` **同在 `packages/nuxt/src/app/composables/router.ts`**——也就是说它是 Nuxt 提供、而非你从 `vue-router` 自己导入的那一份。

## 官方从不这样写：几点提醒

| 你可能想写的 | 建议 |
| --- | --- |
| `watch(route, ...)` 深监听整个路由对象 | 官方三处示例监听的都是**具体来源**（`[page]`、`searchQuery`、computed `userId`），而不是整个 `route`。范围越具体，触发越少 |
| `watch(() => route.fullPath, ...)` 然后据此改渲染 | 120 那条水合陷阱仍然成立：**`fullPath` 包含 hash，而浏览器请求不带片段**。当 key、做比较没问题，别拿它去驱动模板文本 |
| 在中间件里 `watch` 路由 | 120 的官方限制：**中间件里没有"当前路由"的概念**，`useRoute()` 只应在组件 setup 或插件里用；中间件要的是参数 `to`/`from` |
| 组件复用时用 `watch` 补状态 | 先决定"要不要重建"：重建归 `key`（092），不重建的响应归本篇三条路径 |

## 自查

1. "页面要不要重建"与"数据要不要跟着换"分别由什么决定？
2. 把 ref 放进 `useFetch` 的 `query` 里会发生什么？官方用什么注释说明？怎么退出？
3. `watch: [page]` 这种显式 sources 的写法，解决的是哪一类场景？
4. 用 computed 作为 key 的好处是什么？官方那条注释怎么写的？
5. `cause` 有哪四个取值？怎么区分"由监听触发的取数"？
6. 输入框连续变化时，哪个默认值帮你把多余请求取消掉？`enabled: false` 会阻塞哪几种执行？

## 下一篇

123 讲 `useRouter()`：它返回什么、什么时候会是空、`push`/`replace` 与官方推荐的 `navigateTo` 之间怎么分工。
