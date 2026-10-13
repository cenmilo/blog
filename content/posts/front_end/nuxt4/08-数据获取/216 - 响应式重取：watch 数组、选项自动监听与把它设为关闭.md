---
url: /posts/0ca0dded81fc7ad73dfdfa0647b74569/
title: 216 - 响应式重取：watch 数组、选项自动监听与把它设为关闭
date: 2026-12-23T09:00:00+08:00
updated: 2026-12-23T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：让数据跟着状态重取，文档里其实给了三条不同的路，而且它们的关闭方式不一样，我先把三条摆出来再讲开关。① 显式 `watch` 数组：note 是 "The built-in `watch` option allows automatically rerunning the fetcher function when any changes are detected."，示例是 `{ watch: [page] }`，类型 `MultiWatchSources`，说明 "Array of reactive sources to watch and auto-refresh."。② **选项本身给成响应式就会自动重取**：`useFetch` 的 Reactive Fetch Options 一节写 "Fetch options can be provided as reactive, supporting `computed`, `ref` and computed getters. When a reactive fetch option is updated it will trigger a refetch using the updated resolved reactive value."，例子是 `query: { q: searchQuery }`，改一次值就打一次 `/api/search?q=new%20search`；两节表格后面各有一条同义 note。③ key 或 URL 用 computed：215 讲过 key 的生成，212 引的 Reactive Keys 那句是 "When the route changes and userId updates, the data will be automatically refetched"。**关掉的方式只有两处写明**：`useFetch` 给了 `watch: false`（"If needed, you can opt out of this behavior using `watch: false`"，选项表补 "`false` disables watching."），`enabled: false` 则把三种执行一起挡（"While `false`, every execution is blocked (initial fetch, `execute`/`refresh`, and **watch triggers**)"）。至于 `useAsyncData` 那一节的 `watch` 行**类型里没有 `false`**，这是两节之间的一处不对称，我没有替它补齐。

categories:
  - 前端开发

tags:
  - nuxt4
  - watch
  - 响应式
  - useFetch
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先解释一句"响应式重取"：某个 ref 变了，框架自动再跑一次取数，不用你手写 `watch`。**先想清是哪一种"变"在驱动**，因为文档给的三条路触发条件不同。

## 路一：显式 watch 数组

`useAsyncData` 那一节 Watch Parameters 小节的定义与例子：

> The built-in `watch` option allows automatically rerunning the fetcher function when any changes are detected.

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

选项表里这一行是：类型 `MultiWatchSources`，默认 `-`，说明 "Array of reactive sources to watch and auto-refresh."。

这个例子的形状值得记：**handler 里读的是 `page.value`，`watch` 里放的是 `page` 本身**（不带 `.value`）。前者是取值，后者是给框架一个可以被订阅的对象——写反了就不会触发。

## 路二：选项给成响应式，不用写 watch

这条是我最晚才注意到的，因为它藏在一句很平的话里。`useFetch` 的 Reactive Fetch Options：

> Fetch options can be provided as reactive, supporting `computed`, `ref` and computed getters. **When a reactive fetch option is updated it will trigger a refetch** using the updated resolved reactive value.

```ts [app/pages/index.vue]
const searchQuery = ref('initial')
const { data } = await useFetch('/api/search', {
  query: { q: searchQuery },
})
// triggers a refetch: /api/search?q=new%20search
searchQuery.value = 'new search'
```

`useAsyncData` 那一节表格后也有一条同义 note：

> All options can be given a `computed` or `ref` value. These will be watched and new requests made automatically with any new values if they are updated.

（`useFetch` 那条多了一个括号条件："...**unless `watch` is set to `false**`"——这个差别见下面"开关"那节。）

所以 `query: { q: someRef }` 这种写法**不需要再配 `watch: [someRef]`**，它自己就会被监听。这也解释了为什么官方文档里的搜索例子多半只写 `query`。

## 路三：key 或 URL 变成响应式

前两条是"数据参数变了重取"，这一条是"格子换了重取"。`useAsyncData` 的 Reactive Keys：

```vue [app/pages/[id].vue]
<script setup lang="ts">
const route = useRoute()
const userId = computed(() => `user-${route.params.id}`)

// When the route changes and userId updates, the data will be automatically refetched
const { data: user } = useAsyncData(userId, () => fetchUserById(route.params.id))
</script>
```

`useFetch` 对应的是把 URL 写成函数：

```vue [app/pages/[id].vue]
<script setup lang="ts">
const route = useRoute()
const id = computed(() => route.params.id)

// When the route changes and id updates, the data will be automatically refetched
const { data: post } = await useFetch(() => `/api/posts/${id.value}`)
</script>
```

两条注释几乎一字不差，但机制不同：**key 换了等于换一格新数据**（旧 key 的数据还在缓存里，215 的 `useNuxtData` 还能读到）；而路二是在同一格里重取。签名上也支持：`useFetch` 的 `url` 类型是 `string | Request | Ref<string | Request> | (() => string | Request)`。

## 两个写明的开关

**`watch: false`（只在 `useFetch` 那一节写明）**：

> If needed, you can opt out of this behavior using `watch: false`:

```ts [app/pages/index.vue]
const searchQuery = ref('initial')
const { data } = await useFetch('/api/search', {
  query: { q: searchQuery },
  watch: false,
})
// does not trigger a refetch
searchQuery.value = 'new search'
```

选项表里也补了类型与语义：`MultiWatchSources | false`，"Array of reactive sources to watch and auto-refresh. `false` disables watching."

对照 `useAsyncData` 那一节：它的 `watch` 行类型是 `MultiWatchSources`（**没有 `| false`**），note 也没有那个 `unless` 括号。**两节这里不一致，文档没有解释**。处理办法是：需要"带 ref 但不重取"时用 `useFetch` 那条写明路径；写 `useAsyncData` 时不去猜 `false` 的语义。要确认只能实测：给它 `watch: false` 看类型报不报错、以及改 ref 时是否仍重发请求。

**`enabled: false`（把三种执行一起挡）**：

> Barrier that gates whether the `handler` may run. While `false`, every execution is blocked (initial fetch, `execute`/`refresh`, and **watch triggers**), and switching `true` → `false` cancels any in-flight request without clearing `data`. Re-enabling does not refetch on its own.

这条的价值在于它管的是**触发源**而不是单次调用：首次、手动刷新、watch 触发的都挡住。而"关掉不清空 `data`、重新打开不会自动重取"这两句决定了它是"暂停键"而不是"重置键"。

## 怎么知道这次请求是谁触发的

`AsyncDataRequestContext` 里那个 `cause` 就是干这个的：

```ts
type AsyncDataRequestContext = {
  /** The reason for this data request */
  cause: 'initial' | 'refresh:manual' | 'refresh:hook' | 'watch'
}
```

四个值对应四种来源：**首次执行**、**手动 `refresh`/`execute`**、**钩子触发的重取**、**watch 触发**。它作为第三个参数进 `getCachedData`：`(key, nuxtApp, ctx) => DataT | undefined`（213 那条默认实现就用了 `nuxtApp.isHydrating`，而这里给了按原因区分缓存策略的位置）。`middleware` 与 `refresh` 的上下文里也有 signal，但**`cause` 我只在 `getCachedData` 的签名里见过**——`middleware` 收到的 ctx 只写了 `{ signal }`。

所以排查"谁又发了一次请求"，我用的办法是在自定义 `getCachedData` 里打印 `ctx.cause`，比在网络面板里数请求更直接。

## 自查

1. `query: { q: ref }` 要不要再写 `watch: [ref]`？（答：不要，两处 note 都写了响应式选项会被自动监听并重发请求。）
2. 要"带 ref 但不重取"，文档写明的写法是什么？（答：`useFetch` 里 `watch: false`；`useAsyncData` 那一节没有这一档。）
3. `watch` 触发和 `enabled: false` 有什么关系？（答：`enabled` 为 `false` 时 watch 触发的执行也会被挡住，和首次、手动 refresh 一起挡。）

## 下一篇

217 - 不服务端取数那一档：server 关掉之后 status 与 data 会怎样

## 本文依据

- `api/composables/use-async-data`：Watch Parameters 小节定义句与 `page` 示例；选项表 `watch` 行（`MultiWatchSources`）；表后 note（computed/ref 自动重发）；`enabled` 长说明；`AsyncDataRequestContext` 的四个 `cause`；`getCachedData` 类型与默认实现；Reactive Keys 一节（`userId` computed 示例与注释）；`AsyncDataOptions` 定义。
- `api/composables/use-fetch`：Reactive Fetch Options 一节两句与 `searchQuery` 示例、`watch: false` 示例与 "If needed, you can opt out of this behavior using `watch: false`."；表后 note（含 "unless `watch` is set to `false`"）；选项表 `watch` 行（`MultiWatchSources | false`、"`false` disables watching."）；Reactive Keys and Shared State 一节（函数 URL 示例与注释）；Type 一节 `url` 参数类型与 `UseFetchOptions` 定义；`AsyncDataRequestContext` 同页重复定义。
