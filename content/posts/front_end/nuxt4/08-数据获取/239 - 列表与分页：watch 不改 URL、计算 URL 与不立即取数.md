---
url: /posts/7e1a755131c1afff3b7ed1e72d27b494/
title: 239 - 列表与分页：watch 不改 URL、计算 URL 与不立即取数
date: 2026-12-30T11:00:00+08:00
updated: 2026-12-30T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇把列表页/搜索页最常用的三段官方写法排在一起，起点是官方那句最容易让人困惑的提醒：**"Note that *watching a reactive value won't change the URL fetched*. For example, this will keep fetching the same initial ID of the user because **the URL is constructed at the moment the function is invoked**."**——所以 `useFetch(\`/api/users/${id.value}\`, { watch: [id] })` 这种写法**会反复取同一个 ID**（反例在文档里直接给了）。官方给的三条正解分别是：① **把参数挂成响应式值**："Instead of juggling your way around, you can **attach each param as a reactive value. Nuxt will automatically use the reactive value and re-fetch each time it changes.**"（`useLazyFetch('/api/user', { query: { user_id: id } })`）；② **URL 用回调算**："In the case of more complex URL construction, you may use a callback as a computed getter that returns the URL string. **Every time a dependency changes, the data will be fetched using the newly constructed URL.**"（`useLazyFetch(() => \`/api/users/${id.value}\`, { immediate: false })`）；③ 需要"等用户操作才取"就用 `immediate: false` 配 `execute`："The `useFetch` composable will start fetching data the moment it is invoked. You may prevent this by setting `immediate: false`... With that, you will need both the `status` to handle the fetch lifecycle, and `execute` to start the data fetch."。`status` 四值在这一节被逐条定义（idle 是 "when the fetch hasn't started"），这也是为什么官方示例模板里要单独给 "Type a user ID" 那一支。

categories:
  - 前端开发

tags:
  - nuxt4
  - 分页
  - watch
  - execute
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


列表页的三种需求——翻页、搜索、按需加载——在文档里对应三段不同的写法，混用的典型症状是**点了下一页但请求还是第 1 页**，或者**每敲一个字符就打一次接口**。

## 先看那个陷阱：watch 不会重算 URL

官方给的 Watch 一节定义与反例（原样抄，反例的解释句是重点）：

```vue
<script setup lang="ts">
const id = ref(1)

const { data, error, refresh } = await useFetch(`/api/users/${id.value}`, {
  watch: [id],
})
</script>
```

> Note that **watching a reactive value won't change the URL fetched**. For example, this will keep fetching the same initial ID of the user because the URL is constructed at the moment the function is invoked.

这句 "the URL is constructed at the moment the function is invoked" 值得单独背下来：**模板字符串里的 `id.value` 在调用那一刻就被求值成常量了**，`watch` 只是让它"再跑一次同一个请求"。

官方接着给方向："If you need to change the URL based on a reactive value, you may want to use a [computed URL] instead."

顺带另一条同族提醒，`watch: false` 关掉的是**选项自动监听**：

```ts
// Won't automatically refetch when id changes
const { data, execute } = await useFetch('/api/users', {
  query: { id }, // id is watched by default
  watch: false, // disables automatic watching of id
})

// doesn't trigger refetch
id.value = 2
```

注释里的 "`id` is watched by default" 说明**默认状态是"带 ref 就会被监听"**，`watch: false` 是退出键（216 讲过这条只在 `useFetch` 页写明）。

## 正解一：参数交给 query，别拼进路径

官方的 Computed URL 一节开头给了理由与做法：

> Sometimes you may need to compute a URL from reactive values, and refresh the data each time these change. Instead of juggling your way around, you can attach each param as a reactive value. Nuxt will automatically use the reactive value and re-fetch each time it changes.

```vue
<script setup lang="ts">
const id = ref(null)

const { data, status } = useLazyFetch('/api/user', {
  query: {
    user_id: id,
  },
})
</script>
```

这一段的形状：**URL 固定、变化的部分放进 `query` 并把 ref 原样写进去**。翻页就自然是这个样子：

```ts
const page = ref(1)
const { data } = await useFetch('/api/articles', { query: { page } })
```

**这个 `page` 版本是我按上面那条规则套的**（官方给的例子是 `user_id`），但它的等价性来自 `query` 那一行原文——"Query/search params to append to the URL"——加上 221 那句"响应式选项会被自动监听"。

这一版的代价要清楚：**每次 `page` 变化都会发一次新请求**，所以搜索框直接绑 `query.q` 会出现"输入即请求"（防抖方案文档没写，221 里我标过是自己加的）。

## 正解二：URL 真的会变时，用回调

> In the case of more complex URL construction, you may use a callback as a computed getter that returns the URL string.
>
> Every time a dependency changes, the data will be fetched using the newly constructed URL. Combine this with not-immediate, and you can wait until the reactive element changes before fetching.

```vue
<script setup lang="ts">
const id = ref(null)

const { data, status } = useLazyFetch(() => `/api/users/${id.value}`, {
  immediate: false,
})
</script>

<template>
  <div>
    <!-- disable the input while fetching -->
    <input
      v-model="id"
      type="number"
      :disabled="status === 'pending'"
    >

    <div v-if="status === 'idle'">
      Type a user ID
    </div>

    <div v-else-if="status === 'pending'">
      Loading ...
    </div>

    <div v-else>
      {{ data }}
    </div>
  </div>
</template>
```

这段我觉得是整页最值得抄的形状，三件事同时做对了：

1. **URL 是函数**，所以每次求值都用新 `id`（对比陷阱那节的模板串）。
2. **`immediate: false`**：`id` 初始是 `null`，不关掉就会立刻打一次无意义请求。
3. **模板按 `status` 分三支**，`idle` 那一支给的是"请输入"而不是转圈——**这正是 214 那条"四个状态要分开处理"的实践**。顺带 `:disabled="status === 'pending'"` 是官方自己加的防重复输入。

`v-model="id"` 绑到 `type="number"` 的 input，所以 `id` 可能变成字符串——**`toValue`/类型一致性问题文档没在这里提**，我按 214 的 ref 语义使用，不额外假设。

最后一句官方留的口子："If you need to force a refresh when other reactive values change, you can also [watch other values]."——也就是三种触发（响应式选项、响应式 URL、`watch` 数组）可以叠。

## 正解三：点了按钮才取数

Not immediate 一节原文：

> The `useFetch` composable will start fetching data the moment it is invoked. You may prevent this by setting `immediate: false`, for example, to wait for user interaction. With that, you will need both the `status` to handle the fetch lifecycle, and `execute` to start the data fetch.

```vue
<script setup lang="ts">
const { data, error, execute, status } = await useLazyFetch('/api/comments', {
  immediate: false,
})
</script>

<template>
  <div v-if="status === 'idle'">
    <button @click="execute">
      Get data
    </button>
  </div>

  <div v-else-if="status === 'pending'">
    Loading comments...
  </div>

  <div v-else>
    {{ data }}
  </div>
</template>
```

官方在这一节末尾把四值重新列了一遍（措辞比 API 页更口语）：

- `idle` when the fetch hasn't started
- `pending` when a fetch has started but not yet completed
- `error` when the fetch fails
- `success` when the fetch is completed successfully

**`execute` 与 `refresh` 的分工在这一页也被写清了**（214 只说"别名"）：

> The `execute` function is an alias for `refresh` that works in exactly the same way but is **more semantic for cases when the fetch is not immediate**.

所以我的写法纪律是：`immediate: false` 的场景一律解构 `execute`，翻页/重取的场景用 `refresh`——**行为相同，读代码的人更容易看懂**。

## 列表页的两条补充

**只要模板用到的字段**：官方给 `pick` 的定位是"minimize the payload size stored in your HTML document"（235 引过），列表接口往往字段最多，收益最直接：

```ts
const { data: mountain } = await useFetch('/api/mountains/everest', {
  pick: ['title', 'description'],
})
```

**多个接口一次等**：官方 Recipes 一节给的是 `Promise.all` 包在 `useAsyncData` 里，并且**每个 `$fetch` 都把 signal 传进去**：

```ts
const { data } = await useAsyncData((_nuxtApp, { signal }) => {
  return Promise.all([
    $fetch('/api/comments/', { signal }),
    $fetch('/api/author/12', { signal }),
  ])
})

const comments = computed(() => data.value?.[0])
const author = computed(() => data.value?.[1])
```

（useAsyncData 那页另给过一个"购物车两个优惠接口"的同款例子，注释是 `discounts.value.coupons` / `.offers`。）

如果两个请求有依赖关系（先拿 ID 再拿详情），官方没有给链条写法——**那就是两格 key 串起来的问题（215/228），不是并行**。

## 自查

1. `useFetch(\`/api/x/${id.value}\`, { watch: [id] })` 会取到什么？为什么？（答：反复取同一个初始 ID；官方说 "the URL is constructed at the moment the function is invoked"。）
2. `immediate: false` 时必须同时用到哪两样？（答：`status`（判断 `idle`/`pending`/`error`/`success`）与 `execute`（真正发起那次取数）。）
3. `execute` 和 `refresh` 有行为差别吗，官方推荐怎么区分使用？（答：没有，"works in exactly the same way"，但 `execute` 在非立即取数的场景语义更贴切。）

## 下一篇

240 - 提交表单之后该调谁：refresh、execute 与 clear
