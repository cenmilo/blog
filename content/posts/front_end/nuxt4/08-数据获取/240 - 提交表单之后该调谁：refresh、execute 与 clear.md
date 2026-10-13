---
url: /posts/1b03280402b2233f746d0771c2b35fdf/
title: 240 - 提交表单之后该调谁：refresh、execute 与 clear
date: 2026-12-31T09:00:00+08:00
updated: 2026-12-31T09:00:00+08:00
author: cmdragon

summary:
  摘要：这一篇解决一个具体时刻：**POST 成功了，屏幕上的数据还是旧的，我该调哪个？**文档给的四个候选各有明确边界。① 实例上的 `refresh`（或 `execute`）："If you want to fetch or refresh data manually, use the `execute` or `refresh` function provided by the composables."，它自带一条并发保护："By default, Nuxt waits until a `refresh` is finished before it can be executed again."，并接受 `AsyncDataExecuteOptions`（`dedupe`、`timeout`、`signal` 三键）。② 全局的 `refreshNuxtData(keys?)`——官方在 useNuxtData 的乐观更新示例里就是在 `onResponse` 里 `await refreshNuxtData('todos')`（注释 "Invalidate todos in the background if the request succeeded."），并且有一条 tip 把两者并列："To globally refetch or invalidate cached data, see `clearNuxtData` and `refreshNuxtData`."。③ `clear()`——**当你在组件里、不想去猜 key 时用它**："If you want to clear the data provided, for whatever reason, without needing to know the specific key to pass to `clearNuxtData`, you can use the `clear` function provided by the composables."，官方示例是 `watch` 路由变化时 `clear()`。④ 直接改那一格数据（乐观更新），前提是那次取数用了显式 key 并用 `useNuxtData('todos')` 拿到同一个 ref。最后一个容易被忽略的点：如果你写了自定义 `getCachedData`，手动刷新可能仍吃缓存——升级指南给的解法是 `if (ctx.cause === 'refresh:manual') return undefined`。

categories:
  - 前端开发

tags:
  - nuxt4
  - refresh
  - 表单
  - clear
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


场景固定一下：页面上有列表（`useFetch` 取的），下面一个新增表单（POST）。提交成功后界面要么显示新数据要么回滚。**"提交后要干什么"这一步，文档给了四个不同层次的做法。**

## 先分清两件事：发请求与重取数

官方 210 那个例子的分工在这里直接可用——**提交动作本身用 `$fetch`，取回来的列表才归 `useFetch`/`useAsyncData` 管**：

```vue [app/app.vue]
<script setup lang="ts">
const { data } = await useFetch('/api/data')

async function handleFormSubmit () {
  const res = await $fetch('/api/submit', {
    method: 'POST',
    body: {
      // My form data
    },
  })
}
</script>
```

配套那句解释（210 抄过）："`$fetch` has no such mechanism and is a better option to use when the request is solely made from the browser."。所以**别把 POST 包进 `useFetch`**——它没有"重取"的语义，包进去只会让 payload 里多存一份提交响应。

发完之后要不要重取，才是这一篇要选的。

## 四个候选，按作用域从小到大

| 我要的效果 | 调用 | 官方依据句 |
| --- | --- | --- |
| 重取这一次取数的数据 | `await refresh()` / `await execute()` | "use the `execute` or `refresh` function provided by the composables" |
| 按 key 重取（可在别处触发） | `await refreshNuxtData('todos')` 或数组 | 乐观更新示例里就在 `onResponse` 调用 |
| 重取所有 key | `await refreshNuxtData()` | "All `useAsyncData` and `useFetch` keys are re-fetched when no `keys` are explicitly specified." |
| 丢掉数据、回到 idle | `clear()` | "without needing to know the specific key to pass to `clearNuxtData`" |
| 立即改界面（不等服务端） | 直接给那一格 ref 赋值 | 官方乐观更新示例 `todos.value = [...todos.value, newTodo.value]` |

官方把后两条列成一对：

> To globally refetch or invalidate cached data, see `clearNuxtData` and `refreshNuxtData`.

**`refresh` 与 `refreshNuxtData` 的差别是作用域**：前者是你手上那次取数的实例方法，后者是按 key 从全局触发（229 那张粒度表就是为这个准备的）。

## 一条自带的防抖，和一个可传的选项

`refresh` 的默认行为里藏了一句保护（214 抄过）：

> By default, Nuxt waits until a `refresh` is finished before it can be executed again.

所以"用户连点两下提交"不会起两个 refresh——**但我仍然要让按钮禁用**，因为等待期间界面得给反馈。官方在按钮示例里给的就是最简形状：

```vue
<button @click="() => refresh()">
  Refresh data
</button>
```

第二层是入参：`refresh(opts?: AsyncDataExecuteOptions)`，三键 `dedupe`、`timeout`、`signal`（212/214 都引过）。**表单场景里我用到的是 `signal`**：用户关掉弹窗时把这次重取取消掉：

```ts
let abortController: AbortController | undefined

function handleUserAction () {
  abortController = new AbortController()
  refresh({ signal: abortController.signal })
}

function handleCancel () {
  abortController?.abort() // aborts the ongoing refresh request
}
```

这段是官方原文示例（`Make Your handler Abortable` 一节），注释也是官方的。

## 一个坑：refresh 可能仍吃到缓存

这是我认为这一篇最有用但最难自己发现的一条。`experimental.granularCachedData` 的定位句写的是：

> Whether to call and use the result from `getCachedData` when refreshing data for `useAsyncData` and `useFetch` (whether by `watch`, `refreshNuxtData()`, or a manual `refresh()` call.

也就是说**"手动 refresh"这条路上，`getCachedData` 仍然可能被调用并被采用**。如果你写过自己的 `getCachedData`（或者照抄了默认实现），刷新拿到的可能是缓存里的旧值。官方在升级指南里给的修法就一句判断：

```ts
getCachedData: (key, nuxtApp, ctx) => {
  // ctx.cause - can be 'initial' | 'refresh:hook' | 'refresh:manual' | 'watch'

  // Example: Don't use cache on manual refresh
  if (ctx.cause === 'refresh:manual') return undefined

  return cachedData[key]
}
```

**"return undefined"在这里的含义是"这次没有缓存可用"**，于是走 handler。这条对表单提交后的重取是关键——不然会出现"提交成功但界面还是旧的，再点一次才更新"。

## 官方给的完整提交-刷新形状

把 228 那段乐观更新示例按"表单提交"重排，四个动作全在里面（代码是官方的，我只调整了顺序并保留原注释）：

```vue
<script setup lang="ts">
const newTodo = ref('')
let previousTodos = []
const { data: todos } = useNuxtData('todos')

async function addTodo () {
  await $fetch('/api/addTodo', {
    method: 'post',
    body: { todo: newTodo.value },
    onRequest () {
      previousTodos = todos.value              // 存旧值
      todos.value = [...todos.value, newTodo.value] // 立刻改界面
    },
    onResponseError () {
      todos.value = previousTodos              // 失败回滚
    },
    async onResponse () {
      await refreshNuxtData('todos')           // 成功后用服务端真值覆盖
    },
  })
}
</script>
```

**`onResponse` 里那句 "Invalidate todos in the background if the request succeeded." 是官方注释**，它表达的节奏我很认同：先给用户即时反馈，再用服务端结果收尾，**而不是等一次整页重取**。

需要提醒的两处：`todos.value` 初值可能是 `undefined`（228 讲过 `useNuxtData` 无缓存时是 `undefined`，我在那篇补了 `?? []`）；`previousTodos = todos.value` 存的是引用，不可变写法才安全（228 也标过）。

## clear 的适用时刻：路由变了

`Clear` 一节给的场景与代码：

> If you want to clear the data provided, for whatever reason, without needing to know the specific key to pass to `clearNuxtData`, you can use the `clear` function provided by the composables.

```vue
<script setup lang="ts">
const { data, clear } = await useFetch('/api/users')

const route = useRoute()
watch(() => route.path, (path) => {
  if (path === '/') {
    clear()
  }
})
</script>
```

注意它把 `clear()` 放在 `watch` 回调里——**回调里调实例方法是安全的**（`clear` 不是需要上下文的 composable，只是普通函数，见 214 那条"三个函数"）。而 233 讲的上下文限制针对的是"调用 composable 本身"，两者别混。

`clear()` 之后 `status` 是 `idle`（214 那四件事之一），**不会自动重取**——这跟 `enabled` 从 `false` 切回 `true` 不自动重取是同一个设计（213）。

## 自查

1. 提交 POST 该用哪个函数发，重取列表又该用哪个？（答：发用 `$fetch`，重取用实例的 `refresh`/`execute` 或按 key 的 `refreshNuxtData`。）
2. 手动 refresh 有可能拿到旧数据吗？条件是什么？（答：有可能——`granularCachedData` 默认开启时 `getCachedData` 在手动刷新路径上仍被采用；解法是按 `ctx.cause === 'refresh:manual'` 返回 `undefined`。）
3. `clear()` 与 `refresh()` 的区别？（答：`clear` 只丢弃数据并把状态打回 `idle`、取消在飞请求，不会重新取；`refresh` 会真的再跑一次 handler。）

## 下一篇

241 - 取数与 SEO：只有服务端跑过的那次才进首屏

## 本文依据

- `getting-started/data-fetching`：Refresh and execute 段（定义句、按钮示例、`execute` 语义句、指向 `clearNuxtData`/`refreshNuxtData` 的 tip）；Clear 段（定义句与 route watch 示例）；`useFetch` + `$fetch` 提交示例与解释句（210 已逐句引）；Serializing/`pick` 各节不涉及本篇故不引。
- `api/composables/use-async-data`：返回值表 `refresh`/`execute`/`clear` 三行与"等上次跑完"那句、`AsyncDataExecuteOptions` 三键、手动 abort 示例、`enabled`/`getCachedData` 说明（213/214 已逐句引）。
- `api/composables/use-nuxt-data`：Optimistic Updates 完整示例（三钩子与三行注释）。
- `api/utils/refresh-nuxt-data`、`api/utils/clear-nuxt-data`：两个全局函数的参数与返回（215/229 已逐句引）。
- `guide/going-further/experimental-features`、`getting-started/upgrade`：`granularCachedData` 定位句与 `ctx.cause` 迁移写法（236 已逐句引）。
- `api/utils/dollarfetch`：`$fetch` 适合纯浏览器动作的理由（211 已逐句引）。
