---
url: /posts/508484c14bcc22fc30a90307737d5375/
title: 228 - useNuxtData 实战：乐观更新与失败回滚的官方形状
date: 2026-12-27T09:00:00+08:00
updated: 2026-12-27T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：215 用 `useNuxtData` 讲过"读缓存当占位值"，这一篇讲它真正的用途：在不重新渲染整棵树的前提下改那一格数据。官方定位是 "Access the current cached value of data fetching composables."，note 列出适用范围："gives you access to the current cached value of `useAsyncData`, `useLazyAsyncData`, `useFetch` and `useLazyFetch` with **explicitly provided key**"，签名只有一个成员——`export function useNuxtData<DataT = any> (key: string): { data: Ref<DataT | undefined> }`。两条返回值描述我特别在意第二条：**"This `Ref` automatically updates if the cached data changes, allowing seamless reactivity in your components."**——它给的是同一格数据的引用，不是你复制出来的快照。官方给的用例三件（reusing already-fetched data、Optimistic Updates、cascading data updates），并把前提写死："To use `useNuxtData`, ensure that the data-fetching composable has been called with an explicitly provided key."。**乐观更新那段示例是这一族里最值得抄的一次交互**：`onRequest` 里先存 `previousTodos` 再把新项 push 进 `todos.value`；`onResponseError` 里 `todos.value = previousTodos` 回滚；`onResponse` 里 `await refreshNuxtData('todos')` 在后台用服务端真值覆盖——三个钩子刚好是 221 那四个拦截器里的三个。

categories:
  - 前端开发

tags:
  - nuxt4
  - useNuxtData
  - 乐观更新
  - 缓存
  - 状态

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先说这一族的整体形状：`useAsyncData` 那一头把数据存进一格，`useNuxtData` 这一头**按 key 拿到同一格**。215 讲的是"读它"，这一篇讲的是"改它"。

## 签名和它唯一的成员

```ts [Signature]
export function useNuxtData<DataT = any> (key: string): { data: Ref<DataT | undefined> }
```

| 项目 | 官方原话 |
| --- | --- |
| 参数 | "The unique key that identifies the cached data. This key should match the one used during the original data fetch." |
| 返回 | "A reactive reference to the cached data associated with the provided key. **If no cached data exists, the value will be `undefined`.** This `Ref` automatically updates if the cached data changes, allowing seamless reactivity in your components." |

三条约束都在这里了：**key 是字符串且必须对上**、**泛型默认 `any`**（所以要自己标，否则 `todos.value.find(...)` 这种调用完全没有类型保护）、**没有缓存时是 `undefined` 而不是空数组**。

`undefined` 这一条对官方自己的示例都算一个隐患：`useNuxtData('todos')` 在缓存还没建立时（比如直接刷新到那个子组件），`todos.value` 是 `undefined`，`[...todos.value, newTodo.value]` 会直接抛错。所以我照抄时加了兜底：

```ts
const { data: todos } = useNuxtData('todos')

function addOptimistically (item: string) {
  const base = todos.value ?? []
  todos.value = [...base, item]
}
```

**这个 `?? []` 是我加的**，官方示例没有（它假设列表页已经取过数）。

## 官方给乐观更新下的定义

这一句我觉得是整节里最值得背的，因为它同时说明了适用场景与失败分支：

> Optimistic Updates is a technique where the user interface is updated immediately, assuming a server operation will succeed. If the operation eventually fails, the UI is rolled back to its previous state.

然后是完整示例。列表页只有一行取数：

```vue [app/pages/todos.vue]
<script setup lang="ts">
// We can access same data later using 'todos' key
const { data } = await useAsyncData('todos', (_nuxtApp, { signal }) => $fetch('/api/todos', { signal }))
</script>
```

组件那一头是三钩子协作：

```vue [app/components/NewTodo.vue]
<script setup lang="ts">
const newTodo = ref('')
let previousTodos = []

// Access to the cached value of useAsyncData in todos.vue
const { data: todos } = useNuxtData('todos')

async function addTodo () {
  await $fetch('/api/addTodo', {
    method: 'post',
    body: {
      todo: newTodo.value,
    },
    onRequest () {
      // Store the previously cached value to restore if fetch fails.
      previousTodos = todos.value

      // Optimistically update the todos.
      todos.value = [...todos.value, newTodo.value]
    },
    onResponseError () {
      // Rollback the data if the request failed.
      todos.value = previousTodos
    },
    async onResponse () {
      // Invalidate todos in the background if the request succeeded.
      await refreshNuxtData('todos')
    },
  })
}
</script>
```

我把它的分工拆开，因为这正是"为什么不用 `useState` 手写一份"的答案：

| 阶段 | 钩子 | 做的事 |
| --- | --- | --- |
| 发出前 | `onRequest` | 先把当前值存进局部变量，再就地改 `todos.value`（列表页立刻看到新项） |
| 失败 | `onResponseError` | 用存下来的旧值整体回滚 |
| 成功 | `onResponse` | 调 `refreshNuxtData('todos')`，让服务端真值覆盖乐观值 |

三步都在同一格数据上操作，**列表页不需要接收事件、不需要重新挂载**——`useNuxtData` 返回的是那个 ref 本身，改了它，读它的地方跟着变（这正是那句 "automatically updates" 的含义）。

三个钩子的名字与来源是 221 那张拦截器表里的 `onRequest` / `onResponseError` / `onResponse`（第四个 `onRequestError` 这里没用）。

## 两处我认为必须自己确认的地方

第一处，**`previousTodos = todos.value` 存的是引用而不是副本**。如果后面有代码就地改了同一个数组（`push`、`splice`），回滚时拿到的可能是被改过的那一份。示例里用的是"整体替换 `todos.value = [...todos.value, new]`"这种不可变写法，所以暂时没露问题。**我倾向显式复制一份**（`[...todos.value]`），但这是我自己加的规则，文档没写。

第二处，这段逻辑要跑在服务端渲染阶段吗？`onRequest` 里读 `localStorage` 之类的肯定不行（199/221 都提过），但这里只是改 ref，问题不大。真正要小心的是**触发时机**：这段代码写在点击处理里，只在浏览器发生，所以天然绕开了 SSR 双跑（210）那一类问题。**如果把它挪进 `<script setup>` 顶层，就会变成两侧各跑一次。**

## 它和 useState 的分工

官方在 100 那篇给过 `useHydration` 与 `useState` 的分界（要全局响应式状态用 `useState`）。这里同理：**`useNuxtData` 不是状态管理工具，它读的是"某次取数的结果"**，key 必须对应上一次真实取数。列表页没取过、key 写错了，拿到的就是 `undefined`——它不会帮你把请求发出去。

## 自查

1. 用 `useNuxtData` 的前提是什么？（答：那次取数用了**显式 key**，且 key 要对上。）
2. 缓存不存在时 `data` 是什么？我的示例代码为什么会因此出问题？（答：`undefined`；官方示例直接 `[...todos.value, ...]` 会抛，需要 `?? []` 兜底。）
3. 乐观更新三步各自的钩子与目的？（答：`onRequest` 存旧值并立刻改；`onResponseError` 回滚；`onResponse` 用 `refreshNuxtData('todos')` 让服务端真值覆盖。）

## 下一篇

229 - 两种清理粒度：refreshNuxtData 与 clearNuxtData
