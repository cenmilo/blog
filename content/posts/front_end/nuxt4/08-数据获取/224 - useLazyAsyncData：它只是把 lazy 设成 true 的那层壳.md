---
url: /posts/e300bbe403fd30662d151c20daebfa83/
title: useLazyAsyncData：它只是把 lazy 设成 true 的那层壳
date: 2026-12-25T11:00:00+08:00
updated: 2026-12-25T11:00:00+08:00
author: cmdragon

summary:
  摘要：这一族 `useLazy*` 名字看着像新东西，官方定义其实只有一句："useLazyAsyncData provides a wrapper around useAsyncData that triggers navigation before the handler is resolved by **setting the `lazy` option to `true`**."，Parameters 一节写得更机械："`useLazyAsyncData` accepts the same parameters as `useAsyncData`, **with the `lazy` option automatically set to `true`**."，Type 一节两句补充是 "has the same signature as useAsyncData" 与 "returns the same values as useAsyncData"。**两条重载也在**（`(handler, options?)` 与 `(key, handler, options?)`），返回类型同样是 `AsyncData<DataT, DataE> & Promise<...>`。行为差异只有一条 note：「By default, useAsyncData blocks navigation until its async handler is resolved. `useLazyAsyncData` allows navigation to occur immediately while data fetching continues in the background.」——**换的就是这一件事**。代价是模板要自己管状态，官方示例给的判据是 `status === 'pending'` 与 `status === 'error'` 两支，配一句提醒："Check `status === 'pending'` and `status === 'error'` in your component's template before using the result."；另一段示例还留了一句很实在的注释：「Because count might start out null, you won't have access to its contents immediately, but you can watch it.」最后一个必须记住的点：它同样是编译器保留名（warning 原话）。

categories:
  - 前端开发

tags:
  - nuxt4
  - useLazyAsyncData
  - lazy
  - 状态
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


我先入为主地以为 `useLazyAsyncData` 里有"延迟加载组件"或"分批取数"之类的机制，读完这一页发现它薄得只有两层：**一个壳，加一个必须自己处理的后果。**

## 官方定义就这一句

> `useLazyAsyncData` provides a wrapper around [`useAsyncData`] that triggers navigation before the handler is resolved by setting the `lazy` option to `true`.

Parameters 一节把它说成机械等价：

> `useLazyAsyncData` accepts the same parameters as [`useAsyncData`], with the `lazy` option automatically set to `true`.

Type 一节还有两句同级声明："`useLazyAsyncData` has the same signature as `useAsyncData`." 与 Return Values 的 "`useLazyAsyncData` returns the same values as `useAsyncData`."

签名两条我也照抄了（**两条重载都在**，形状和 212 那两条一致）：

```ts [Signature]
export function useLazyAsyncData<ResT, DataE = unknown, DataT = ResT> (
  handler: AsyncDataHandler<ResT>,
  options?: AsyncDataOptions<ResT, DataT>,
): AsyncData<DataT, DataE> & Promise<AsyncData<DataT, DataE>>

export function useLazyAsyncData<ResT, DataE = unknown, DataT = ResT> (
  key: MaybeRefOrGetter<string>,
  handler: AsyncDataHandler<ResT>,
  options?: AsyncDataOptions<ResT, DataT>,
): AsyncData<DataT, DataE> & Promise<AsyncData<DataT, DataE>>
```

所以我读出来的结论只有一条：**用它可以，但它不给你任何新能力**。选项表（213 那 14 行）、返回值七个成员（214）在这里全部照旧。

## 唯一的行为差异

note 原文：

> By default, `useAsyncData` blocks navigation until its async handler is resolved. `useLazyAsyncData` allows navigation to occur immediately while data fetching continues in the background.

这和 210 讲的 `await` 话题是相邻但不同的两件事：`await` 管的是"我后面代码要不要等"，`lazy` 管的是"**导航要不要等**"。213 里那句实现说明（"Under the hood, `lazy: false` uses `<Suspense>` to block the loading of the route before the data has been fetched."）正好解释了这里为什么会"立刻导航"——不挂 `<Suspense>`。

## 代价：模板必须自己管状态

官方 Usage 示例把这点演得很直白：

```vue [app/pages/index.vue]
<script setup lang="ts">
const { status, data: posts } = await useLazyAsyncData('posts', () => $fetch('/api/posts'))
</script>

<template>
  <div>
    <div v-if="status === 'pending'">
      Loading...
    </div>
    <div v-else-if="status === 'error'">
      Error loading posts
    </div>
    <div v-else>
      {{ posts }}
    </div>
  </div>
</template>
```

示例后面那句提醒是这一页的实操要点：

> `useLazyAsyncData` lets navigation continue while it fetches data. Check `status === 'pending'` and `status === 'error'` in your component's template before using the result.

**两支都写了**：pending 与 error。我原来只补了 loading，错误那一支漏掉时表现是"转圈消失、内容空白、也不报错"——因为 `v-else` 把错误态当成"成功但没数据"渲染了。

不过这里我要如实说一句：这个例子的 `v-else` 分支其实没区分 `success` 与 `idle`（214 那张表里 status 是四值：`idle`/`pending`/`success`/`error`），而且官方示例的 `<template>` 有两个并列根节点（外层 `<div>` 里再放三个 `<div>`——如果这块组件要参加页面过渡，118/208 的单根要求就会撞上）。**这两点我按其它页的规则处理，不当成这一页示例的推荐形状。**

## 另一段示例里那句关于 null 的注释

页面末尾的 Example 给了一个更完整的形状（`count` 与 `watch`）：

```vue [app/pages/index.vue]
<script setup lang="ts">
/* useLazyAsyncData lets navigation continue before the fetch completes.
  Handle loading and error states in the template.
*/
const { status, data: count } = await useLazyAsyncData('count', () => $fetch('/api/count'))

watch(count, (newCount) => {
  // Because count might start out null, you won't have access
  // to its contents immediately, but you can watch it.
})
</script>
```

那句注释我觉得是官方在这一页最诚实的一句话：**刚开始它是空的，你拿不到内容，但你能监听它**。这跟 213 里 `default` 那一行是配套的——不想处理"空的第一帧"，就给 `default` 一个可渲染的初值；想处理，就用 `status` 分支或 `watch`。

这里也顺带说明一件事：`count` 是 ref，所以 `watch(count, ...)` 监听的是这个引用本身，不是 `.value`（214 那条"四个 ref 三个函数"的分界在这就用上了）。

## 保留名这一条别忘

> `useLazyAsyncData` is a reserved function name transformed by the compiler, so you should not name your own function `useLazyAsyncData`.

和 212/219 那几条同一个模式：**这一族名字都是编译器会转换的保留名**。自己写包装函数时起别名（比如 `useLazyMountains`），别占用这些名字。

## 什么时候值得用它

把 210 与 213 的官方倾向串起来，我的判断顺序是：

1. 首屏一定要这份数据，否则内容是空的 ⇒ 老老实实用 `useAsyncData`/`useFetch`（阻塞导航，落地即满）。
2. 首屏不需要，但导航后想尽早显示 ⇒ `lazy`（这一族或选项都行，官方建议选项写法则"`lazy` is the explicit way to opt into non-blocking navigation"）。
3. 想省掉空状态处理 ⇒ 配 `default`。
4. 需要"切换导航都要重取一次"这种节奏 ⇒ 230 的 `callOnce` 的 `navigation` 模式，而不是 `lazy`（这两者常被混为一谈，管的事完全不同）。

## 自查

1. `useLazyAsyncData` 相对 `useAsyncData` 的唯一差异是什么？（答：自动把 `lazy` 设为 `true`，即导航不被数据阻塞。）
2. 官方要求模板里检查哪两个状态？（答：`status === 'pending'` 与 `status === 'error'`。）
3. 不想在模板里处理"第一帧为空"，文档给的两个办法是什么？（答：给 `default` 初值；或用 `watch` 等它填上——官方注释原话是 "you won't have access to its contents immediately, but you can watch it"。）

## 下一篇

225 - useLazyFetch 与 await 的错觉：初始化了但没等到数据

## 本文依据

- `api/composables/use-lazy-async-data`：定位句（"triggers navigation before the handler is resolved by setting the `lazy` option to `true`"）；note（默认阻塞导航 vs 立即导航）；Usage 示例与那句 "Check `status === 'pending'` and `status === 'error'`..."；reserved name warning；Type 两条重载与 "has the same signature as useAsyncData"；Parameters（"with the `lazy` option automatically set to `true`"）；Return Values（"returns the same values as useAsyncData"）；Example 段（`count` + `watch` 与关于 null 的注释）。
- `api/composables/use-async-data`：`lazy` 与 `<Suspense>` 那条 note、选项表 `lazy`/`default` 两行、返回值表与 Status Values 四条（212-214 已逐句引）。
- `getting-started/data-fetching`：`await` 与 `lazy` 的关系与"lazy 是显式做法"那句（210 已逐句引）。
