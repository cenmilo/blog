---
url: /posts/9f38cd8799aee73e9f0ef2d12d34168a/
title: 219 - useFetch 与 useAsyncData 的六处差异：自动 key、可响应式 URL 与错误类型
date: 2026-12-24T09:00:00+08:00
updated: 2026-12-24T09:00:00+08:00
author: cmdragon

summary:
  摘要：官方对两者关系的原话只有一句："This composable provides a convenient wrapper around `useAsyncData` and `$fetch`."，后面紧跟三件 `useFetch` 自动做的事："It automatically generates a key for the request, provides type hints for request url based on server routes, and infers API response type."；data-fetching 那一节又给了一个"近似等价"的展开式——"**`useFetch(url)` is nearly equivalent to `useAsyncData(url, () => event.$fetch(url))`**"，并说明它是"developer experience sugar for the most common use case"。**注意 "nearly"（近似）这个词是官方自己选的**，差异就在这六处：① 定位（包装 vs 被包装）与那三件自动事项；② 调用位置措辞不同——`useAsyncData` 写 "meant to be called directly in the Nuxt context"，`useFetch` 写 "meant to be called directly in a setup function, plugin, or route middleware"；③ 参数形状（`url` 可以是字符串、Request、ref 或返回它们的函数）；④ 自动 key 的生成规则不同（文件名+行号 对 URL+选项+调用位置）；⑤ 错误泛型的默认值不同（`DataE = unknown` 对 `ErrorT = NuxtError<unknown>`）；⑥ `useFetch` 多出八个请求侧选项（`method`、`query`、`params`、`body`、`headers`、`baseURL`、`cache`、`$fetch`），且它的 `watch` 类型多了 `| false`。**什么时候该退回 `useAsyncData`，官方只给了一类：CMS 或第三方自带查询层的时候。**

categories:
  - 前端开发

tags:
  - nuxt4
  - useFetch
  - useAsyncData
  - 类型推断
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


两节文档分开放，很容易只读一个就以为自己懂了。这一篇把差异逐条对齐，每条都引两边的原文。

## 官方给的关系式

`useFetch` 页开头：

> This composable provides a convenient wrapper around `useAsyncData` and `$fetch`. It automatically generates a key for the request, provides type hints for request url based on server routes, and infers API response type.

data-fetching 那一节的展开式：

> `useFetch(url)` is nearly equivalent to `useAsyncData(url, () => event.$fetch(url))`.
> It's developer experience sugar for the most common use case.

"近似"这个词很关键——**它等于官方承认了差异存在**，只是没在正文逐条列。所以我来列。

顺带记一处文档自身的小混乱：这段的补充句写的是 "You can find out more about `event.fetch` at `useRequestFetch`"，而等式里用的是 `event.$fetch(url)`。两个名字都指向同一个代理请求工具，我不替它统一，只提示读到时别当成两种东西。

## 六处差异，逐条对照

| # | 维度 | `useAsyncData` | `useFetch` |
| --- | --- | --- | --- |
| 1 | 定位 | 被包装者："wrapping async logic and returning the result once it is resolved." | 包装者：`useAsyncData` + `$fetch`，并自动做三件事 |
| 2 | 调用位置措辞 | "meant to be called directly in the **Nuxt context**" | "meant to be called directly in a **setup function, plugin, or route middleware**" |
| 3 | 参数形状 | `(handler, options?)` 或 `(key, handler, options?)` | `(url, options?)`，url 类型是 `string \| Request \| Ref<string \| Request> \| (() => string \| Request)` |
| 4 | 自动 key 规则 | "unique to the **file name and line number**" | "generated from the **URL, options and call site location**" |
| 5 | 错误泛型默认 | `<ResT, DataE = unknown, DataT = ResT>` | `<ResT, ErrorT = NuxtError<unknown>, DataT = ResT>` |
| 6 | 选项集合 | 14 个键（213 那张表） | 23 个键：多出 `method`、`query`、`params`、`body`、`headers`、`baseURL`、`cache`、`$fetch`，且 `watch` 的类型是 `MultiWatchSources \| false` |

第 2 行这条我得说明它的分量：**两处措辞不同不等于适用范围不同**，文档没有说哪个更宽。第 5 行才是每次都会碰到的：`useFetch` 的 `error` 默认是 `NuxtError` 形态（056 讲过那八个字段），而 `useAsyncData` 的 `DataE` 默认 `unknown`——**同一个 `error.value?.message` 在两边给你的类型检查不一样**，用 `useAsyncData` 时要么显式给泛型，要么自己收窄。

## 三件自动做的事里，最值钱的是哪件

| 自动事项（官方原话） | 我的理解 |
| --- | --- |
| "automatically generates a key for the request" | 省掉命名，但 215 那张共享表也因此默认"不共享" |
| "provides type hints for request url based on server routes" | 打 `/api/` 时有候选，**这一条依赖你有 `server/api/` 那些文件**（065 讲过那套目录） |
| "infers API response type" | `data` 的类型直接来自响应，不用手写泛型 |

第二、第三件是我优先用 `useFetch` 的真实理由——**它们影响的是编辑器里能不能帮我，而不是运行时行为**。

## 什么时候退回 useAsyncData

官方给的场合只有一类（data-fetching 页）：

> There are some cases when using the `useFetch` composable is not appropriate, for example when a CMS or a third-party provides their own query layer. In this case, you can use `useAsyncData` to wrap your calls and still keep the benefits provided by the composable.

它的示例长这样，两次调用一次带 key、一次省略 key：

```vue [app/pages/users.vue]
<script setup lang="ts">
const { data, error } = await useAsyncData('users', () => myGetFunction('users'))

// This is also possible:
const { data, error } = await useAsyncData(() => myGetFunction('users'))
</script>
```

"keep the benefits provided by the composable" 指的是 payload 那一套（210），不是 `$fetch` 的类型推断——**换成 `useAsyncData` 就放弃了 URL 候选与响应类型推断**，这两件正是 `useFetch` 自动做的。这是我把两者差异写在一起的实际用处：知道自己在换什么。

另一类我按文档态度列出来但不夸大的：同一个 data-fetching 页面还演示了"多路请求聚成一个"的用法（把 `useAsyncData` 当成"包装一组异步逻辑"的地方），这属于 handler 自由度带来的能力，不是 `useFetch` 缺功能。

## 两个 warning 是 useFetch 专属的

第一条撞名警告：

> `useFetch` is a reserved function name transformed by the compiler, so you should not name your own function `useFetch`. To create a custom variant with pre-defined options, use `createUseFetch` instead.

（`useAsyncData` 那一节有同款，只是没提 `createUseFetch`。）

第二条是我真被咬过的：

> If you encounter the `data` variable destructured from a `useFetch` returns a string and not a JSON parsed object then make sure your component doesn't include an import statement like `import { useFetch } from '@vueuse/core'`.

症状说得很直白：**`data` 是字符串而不是解析好的对象**。原因是 `@vueuse/core` 也有个 `useFetch`，你显式 import 了它，就把 Nuxt 那个覆盖了。这类"装了别的库以后取数结果形状变了"的问题，我现在的排查顺序是先去 `<script setup>` 顶部看有没有显式 `import { useFetch }`，再怀疑后端。

另外 `useFetch` 页给了自定义默认值的方向：**"Need a custom `useFetch` with pre-defined defaults (like `baseURL` or auth headers)? Use `createUseFetch`"** ——这是 222 会碰到的那类需求（给内部请求统一挂 baseURL 或鉴权头）的官方出口。

## 自查

1. 官方那句等价关系里，哪个词表示"不是完全等价"？（答："nearly"——"is nearly equivalent to `useAsyncData(url, () => event.$fetch(url))`"。）
2. 两者的 `error` 类型默认有什么不同？（答：`useFetch` 是 `NuxtError<unknown>`，`useAsyncData` 是 `unknown`。）
3. `data` 忽然变成字符串，官方点名要查什么？（答：组件里是否有 `import { useFetch } from '@vueuse/core'` 这类显式导入。）

## 下一篇

220 - useFetch 选项表逐条：23 个键与多出来的那批请求选项
