---
url: /posts/071bc392aa16743dd8e69e9554b42997/
title: useLazyFetch 与 await 的错觉：初始化了但没等到数据
date: 2026-12-26T09:00:00+08:00
updated: 2026-12-26T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：`useLazyFetch` 这一页比 `useLazyAsyncData` 多了一句我认为是整族最重要的警告：**"Awaiting `useLazyFetch` initializes the call but does not wait for the data. During client-side navigation, check `status === 'pending'` and `status === 'error'` in your component's template before using the result."**——`await` 在这里只完成"发起"，**不完成"等值"**，这跟 210 里那条"await 一个 lazy 调用在客户端导航会立刻返回"是同一件事的第二处记载。等价性官方写得更明确：note 是 "`useLazyFetch` is equivalent to `useFetch` with `lazy: true` option set."，Parameters 写 "Same as useFetch options, **with `lazy` automatically set to `true`**"，签名只有一条（`url` 可以是字符串、Request、ref 或返回它们的函数），返回类型仍是 `AsyncData<DataT, ErrorT> & Promise<...}`。返回值表里有一个别处没写的细节：**这一页的 `clear` 描述是 "Resets `data` to `undefined`, `error` to `undefined`, sets `status` to `idle`, and cancels any pending requests."——少了 `useFetch` 与 `useAsyncData` 那句括号里的"若有 `options.default()` 则复位到它"**，三页口径不一致，我在文末按待验证处理。

categories:
  - 前端开发

tags:
  - nuxt4
  - useLazyFetch
  - lazy
  - await
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


224 那页只说"导航不被阻塞"，这一页多写了一句我以为会写在第一页的话：**`await` 它，等于没等。** 这就是很多人 `await useLazyFetch(...)` 之后直接访问 `data.value.xxx` 会炸的原因。

## 等价关系：官方用了 strongest 的那个词

三处措辞放在一起看：

| 位置 | 原话 |
| --- | --- |
| note | "`useLazyFetch` is equivalent to `useFetch` with `lazy: true` option set." |
| Parameters | "`options` (object): Same as `useFetch` options, with `lazy` automatically set to `true`." |
| 开头定位句 | "provides a wrapper around `useFetch` that triggers navigation before the handler is resolved by setting the `lazy` option to `true`." |

`useLazyAsyncData` 那页用的是 "has the same signature as"，这里直接用 **equivalent**。所以我的读法是：**它不产生任何新语义，就是一个预设了 `lazy: true` 的调用形式。**

签名只有一条（不像 `useLazyAsyncData` 有两条重载，因为 `useFetch` 本来就没有 key 位置）：

```ts [Signature]
export function useLazyFetch<ResT, ErrorT = NuxtError<unknown>, DataT = ResT> (
  url: string | Request | Ref<string | Request> | (() => string | Request),
  options?: UseFetchOptions<ResT, DataT>,
): AsyncData<DataT, ErrorT> & Promise<AsyncData<DataT, ErrorT>>
```

`url` 支持 ref 与函数（221 那条"响应式 URL"在这里同样成立），`ErrorT` 默认 `NuxtError<unknown>`（219 第 5 条差异）。

## 那条警告：await 只完成发起

原文两条 warning，第一条是我今天的主菜：

> Awaiting `useLazyFetch` initializes the call but does not wait for the data. During client-side navigation, check `status === 'pending'` and `status === 'error'` in your component's template before using the result.

第二条是保留名（同族三条一样）："`useLazyFetch` is a reserved function name transformed by the compiler, so you should not name your own function `useLazyFetch`."

为什么值得单独写一条 warning：因为返回类型里那个 `& Promise<...>` 会给你错觉——**看起来可以 await，await 了也不报错**。210 引过的 data-fetching 那条讲得更完整（"it still blocks server rendering as usual, but on **client-side navigation** the `await` resolves immediately without waiting for the request"），两处合起来才是完整规则：

| 场合 | await 一个 lazy 调用的效果 |
| --- | --- |
| 服务端渲染 | 照常等完，HTML 里有数据（官方："it still blocks server rendering as usual"） |
| 客户端导航 | **立刻返回**，`data` 仍是默认值/空 |

所以"本地首屏正常、点进去白屏"就是这个不对称的表现。文档给的做法只有一条：**在模板里判断 status**，不要指望 await。

## 官方示例：把加载态单独列为一个小节

Example 一节标题就叫 "Handling Loading State"，整段是这一族的模板标准形状：

```vue [app/pages/index.vue]
<script setup lang="ts">
/* useLazyFetch lets navigation continue before the fetch completes.
 * Handle loading and error states in the template.
 */
const { status, data: posts } = await useLazyFetch('/api/posts')
watch(posts, (newPosts) => {
  // Because posts might start out null, you won't have access
  // to its contents immediately, but you can watch it.
})
</script>

<template>
  <div v-if="status === 'pending'">
    Loading ...
  </div>
  <div v-else-if="status === 'error'">
    Error loading posts
  </div>
  <div v-else>
    <div v-for="post in posts">
      <!-- do something -->
    </div>
  </div>
</template>
```

这段和 224 那段几乎一模一样，只多了 `v-for`。两处共同暴露的那个问题也还在：`v-else` 覆盖了 `success` 与 `idle` 两种状态，而 status 是四值。**我的处理是显式判断 `status === 'success'` 才渲染内容**，把 `idle` 留出来（触发条件见 217：`immediate: false` 或 `server: false` 都会让它是 `idle`）。

## 返回值表：这里有一处口径不一致

这一页把七个成员完整重列了一遍（不是一句 "same as useFetch" 打发）。但 `clear` 那行的措辞和另外两页不同，我逐字比过：

| 页 | `clear` 的说明 |
| --- | --- |
| `useAsyncData` | "Resets `data` to `undefined` **(or the value of `options.default()` if provided)**, `error` to `undefined`, set `status` to `idle`, and cancels any pending calls." |
| `useFetch` | 同上（含那句括号） |
| `useLazyFetch` | "Resets `data` to `undefined`, `error` to `undefined`, sets `status` to `idle`, and cancels any pending requests." ——**没有那句括号** |

三种可能：文档漏写、`useLazyFetch` 真的不复位到 `default()`、或者行为相同只是描述省略。**这一页没有给出判断依据**，我不替它选。要弄清只要一次实验：

```ts
const { data, status, clear } = await useLazyFetch('/api/x', { default: () => 'placeholder' })
// 先等 status 变 success，再 clear()，然后看 data 是 'placeholder' 还是 undefined
```

顺手另一条：这一页 `pending` 那行写的是 "`true` while a request is in flight. See `useFetch`." ——它把 `experimental.pendingWhenIdle` 那段留给了 `useFetch` 页，所以三页的 `pending` 描述详略也不同。

## 我什么时候用这一族

按文档给的理由，我把判据缩到三条：

1. 需要"点了就走、数据后面补"的导航手感 ⇒ 用 `lazy: true` 或 `useLazyFetch`（官方倾向选项写法，因为意图显式）。
2. 页面首屏就要内容 ⇒ 别用；`await` 在客户端导航救不了你（那条 warning 说的就是这个）。
3. 用了它 ⇒ 模板必须有 `pending`、`error` 两支，并且**不要在 `<script setup>` 里读 `data.value` 的字段**。

## 自查

1. `await useLazyFetch(...)` 在客户端导航时等到数据了吗？（答：没有，官方原话是 "initializes the call but does not wait for the data"。）
2. 三页里 `clear` 的说明有什么不同？（答：`useLazyFetch` 那页少了"若有 `options.default()` 则复位到它"那句，文档未解释原因。）
3. 这一族的名字为什么不能自己占用？（答：都是编译器转换的保留名。）

## 下一篇

226 - createUseFetch 与 createUseAsyncData：默认值模式与覆盖模式

## 本文依据

- `api/composables/use-lazy-fetch`：定位句；"By default, useFetch blocks navigation until its async handler is resolved..." 段；Usage 模板示例（pending/error/`v-for`）；note "has the same signature as useFetch"；warning 两条（await 不等待数据、reserved name）；Type 签名与 note "is equivalent to `useFetch` with `lazy: true` option set"；Parameters 两句；返回值表七行（尤其 `clear` 与 `pending` 两行的措辞）；Example 之 Handling Loading State 全段。
- `api/composables/use-lazy-async-data`：同族对照（"has the same signature as" 与两条重载、`clear` 未列表、`count` 示例与 null 注释）（224 已逐句引）。
- `api/composables/use-fetch`：`clear` 行含 `options.default()` 括号的原文；`pending` 行的 `pendingWhenIdle` 说明（214/220 已逐句引）。
- `api/composables/use-async-data`：`clear` 行原文与 Status Values 四条（213/214 已逐句引）。
- `getting-started/data-fetching`：await 与 lazy 的独立性 warning（"it still blocks server rendering as usual, but on client-side navigation the `await` resolves immediately..."，210 已逐句引）。
