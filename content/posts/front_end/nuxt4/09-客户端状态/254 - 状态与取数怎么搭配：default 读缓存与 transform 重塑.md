---
url: /posts/d05475faa4d1171c146ef459f38340d5/
title: 254 - 状态与取数怎么搭配：default 读缓存与 transform 重塑
date: 2027-01-04T11:00:00+08:00
updated: 2027-01-04T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`useState` 与取数函数不是竞争关系，官方的三个用法把它们接在一起。**一，用缓存当开场值**：`useNuxtData` 那一节的示例就是这套——列表页 `useFetch('/api/posts', { key: 'posts' })`，详情页 `const { data: posts } = useNuxtData('posts')` 之后把 `default () { return posts.value.find(post => post.id === route.params.id) }` 填进 `useLazyFetch`，官方注释分别是 "Access to the cached value of useFetch in posts.vue (parent route)" 与 "Find the individual post from the cache and set it as the default value."；而 `default` 的定义是 "Factory for default value of `data` before async resolves."，它同时是 `clear()` 的复位目标。**二，用 transform 重塑**："Function to transform the result after resolving."，官方在 `pick` 那句之后补的是 "If you need more control or map over several objects, you can use the `transform` function to alter the result of the query."。**三，副作用别塞进 handler**：这条有两处原文，`useAsyncData` 那一节的 warning 是 "The `handler` function should be **side-effect free** to ensure predictable behavior during SSR and CSR hydration. If you need to trigger side effects, use the `callOnce` utility to do so."，data-fetching 那一节更具体——"`useAsyncData` is for fetching and caching data, **not triggering side effects like calling Pinia actions**, as this can cause unintended behavior such as repeated executions with nullish values."，并给了 ❌ 示例。约束也别忘：同 key 的多次调用里 `default`、`transform`、`getCachedData` 都在**必须一致**那七项内（215）。

categories:
  - 前端开发

tags:
  - nuxt4
  - useState
  - default
  - transform
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


我把两件事分开学过："状态"是 `useState`，"取数"是 `useFetch`。真到写页面时它们总在同一个表达式里碰头——**首屏那一瞬间显示什么**。

## 一，先想清楚那一瞬间由谁定

| 谁定 | 键 | 官方定义 |
| --- | --- | --- |
| 取数的开场值 | `default` | "Factory for default value of `data` before async resolves." |
| 状态的开场值 | `useState` 的 `init` | "A function that provides initial value for the state when not initiated." |

两个都是工厂，都只在"还没有值"时求值（251/253 讲过 `init` 只在未初始化时跑）。**区别在寿命**：`default` 管的是这一格数据的空窗，`useState` 的 `init` 管的是一次加载里那份共享状态的空窗。

## 二，官方那一招：拿别人的缓存当自己的默认值

`useNuxtData` 页的示例（两段一起看才有意义，注释都是原文）：

```vue [app/pages/posts.vue]
<script setup lang="ts">
// We can access same data later using 'posts' key
const { data } = await useFetch('/api/posts', { key: 'posts' })
</script>
```

```vue [app/pages/posts/[id].vue]
<script setup lang="ts">
// Access to the cached value of useFetch in posts.vue (parent route)
const { data: posts } = useNuxtData('posts')

const route = useRoute()

const { data } = useLazyFetch(`/api/posts/${route.params.id}`, {
  key: `post-${route.params.id}`,
  default () {
    // Find the individual post from the cache and set it as the default value.
    return posts.value.find(post => post.id === route.params.id)
  },
})
</script>
```

这一招值得拆开看，它同时用了三个性质：

1. 列表页给了**显式 key**（`useNuxtData` 的前提："ensure that the data-fetching composable has been called with an explicitly provided key"）。
2. `useNuxtData` 返回的是**同一格的 ref**，所以父页数据变化它跟着变（"This `Ref` automatically updates if the cached data changes"）。
3. `default` 是**每次需要默认值时调用**的工厂，所以它能读到当时最新的 `posts.value`。

**但要注意 `default` 的返回值也要能进 payload**（251 的序列化边界）。这个例子里返回的是列表里那个普通对象，没问题；如果对象里有 `Date` 或类实例，就得按 252 的结论先测。

`useNuxtData` 那一节给的适用场景也是这三件：

> This is particularly useful for optimizing performance by reusing already-fetched data or implementing features like Optimistic Updates or cascading data updates.

## 三，transform：形状不合时用它

官方在 `pick` 之后给的就是这一句：

> If you need more control or map over several objects, you can use the `transform` function to alter the result of the query.

选项表里的类型说明了一件事：`(input: DataT) => DataT | Promise<DataT>`——**返回类型和被转换的类型同一个**，所以它适合"改形"，不适合"换类型"。想换类型要靠签名上的第三个泛型：`AsyncDataOptions<ResT, DataT = ResT>`（`DataT` 默认等于 `ResT`，212 解释过这正是给 `transform`/`pick` 留的位置）。

`transform` 与 `pick` 谁先跑，218 已经记过是文档空白；而它们与状态的交接点在这里：**`transform` 的结果就是会被塞进 payload 的那个值**，所以"在 transform 里造一个类实例"和"在 `default` 里返回类实例"是同一种风险（252）。

## 四，副作用去哪：不是 handler，是 callOnce

两处原文都指同一件事。`useAsyncData` 那一节的 warning：

> The `handler` function should be **side-effect free** to ensure predictable behavior during SSR and CSR hydration. If you need to trigger side effects, use the `callOnce` utility to do so.

data-fetching 那一节把"store 的 action"单独点出来（并配了 ❌ 示例）：

> `useAsyncData` is for fetching and caching data, not triggering side effects like calling Pinia actions, as this can cause unintended behavior such as repeated executions with nullish values. If you need to trigger side effects, use the `callOnce` utility to do so.

那句 "repeated executions with nullish values" 我读过才理解症状：**handler 被重跑时，参数可能还是空的**，于是 action 被用无意义入参执行一遍。所以官方的分工是：

| 我要做的事 | 放哪 |
| --- | --- |
| 把数据取回来并缓存 | `useAsyncData` 的 handler（保持无副作用） |
| 初始化一个 store / 全局配置 | `callOnce` 里做，值写进 `useState`（250 那个 `websiteConfig` 例子） |
| 提交/删除这类用户动作 | 事件处理函数里 `$fetch`（211/240） |

## 五，一个容易忘的约束

215 那张清单里，**`default`、`transform`、`getCachedData` 都在"必须一致"的七项中**。所以一旦两处调用共用同一个 key：

```ts
// ❌ 官方明说会触发开发警告的形状（把 deep 换成 transform 同理）
useAsyncData('users', handler, { transform: a => a })
useAsyncData('users', handler, { transform: b => ({ ...b, extra: 1 }) })
```

我的做法是**把 transform 收进封装函数内部**，调用方不允许传——这既避免不一致，也避免"两格数据形状不同却共用 key"。

## 自查

1. `default` 与 `useState` 的 `init` 有什么共同点？（答：都是工厂，都只在还没有值时求值；前者管一格的空窗，后者管一次加载里的共享状态。）
2. 官方用 `useNuxtData` + `default()` 的那一招依赖哪三个性质？（答：显式 key、`useNuxtData` 返回同一格 ref 且自动更新、`default` 每次按需求值。）
3. 为什么 store 的 action 不该放进 handler？（答：官方说会造成"用空值重复执行"这类非预期行为；副作用要放 `callOnce`。）

## 下一篇

255 - useCookie：一个能在水合两侧写的 ref

## 本文依据

- `api/composables/use-nuxt-data`：`posts` / `posts-[id]` 两段示例与三条注释；"This is particularly useful for optimizing performance by reusing already-fetched data or implementing features like Optimistic Updates or cascading data updates."；返回值的 "automatically updates" 句与"必须显式 key"前提（215/228 已逐句引）。
- `api/composables/use-async-data`：`default`/`transform`/`pick`/`getCachedData` 四行原文、`AsyncDataOptions<ResT, DataT = ResT>` 与 `DataT` 默认关系、side-effect free 的 warning、Shared State 的"必须一致七项"（212/213/215/218 已逐句引）。
- `api/composables/use-state`：`init` 说明句与 important 序列化边界（251/252 已逐句引）。
- `getting-started/data-fetching`：`useAsyncData` 不用于触发副作用（含 Pinia action 的 ❌ 示例）整段 note；Minimize payload size 一节里 `transform` 那句（210/235 已逐句引）。
- `api/utils/call-once`：Purpose、示例（`useState` + `callOnce`）与"不返回任何东西"（230 已逐句引）。
