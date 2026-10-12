---
url: /posts/e705a320aa7292998016bdef50d2ec39/
title: payload 有多重：载荷提取、pick 与序列化两道关
date: 2026-12-29T10:00:00+08:00
updated: 2026-12-29T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：一次取数的结果要经过两道"序列化"，很多人只想到其中一道：**服务端接口 → 浏览器**用的是 `JSON.stringify`，**服务端渲染 → 水合**用的是 `devalue`。官方对第二道的原话是："When using `useAsyncData` and `useLazyAsyncData` to transfer data fetched on server to the client (as well as anything else that utilizes the Nuxt payload), the payload is serialized with `devalue`. This allows us to transfer not just basic JSON but also to serialize and revive/deserialize more advanced kinds of data, such as regular expressions, Dates, Map and Set, `ref`, `reactive`, `shallowRef`, `shallowReactive` and `NuxtError` - and more."，紧接一条 note 划清界限：**"Note that this *does not apply* to data passed from your server routes when fetched with `$fetch` or `useFetch`"**——第一道的原话是 "the response is serialized using `JSON.stringify`. However, since serialization is limited to only JavaScript primitive types, Nuxt does its best to convert the return type of `$fetch` and `useFetch` to match the actual value."，而它的后果示例很扎眼：接口 `return new Date()`，页面里 **"Type of `data` is inferred as string even though we returned a Date object"**。想改第一道的行为，官方给的是在返回对象上定义 `toJSON`（"If you define a `toJSON` method, Nuxt will respect the return type of the function and will not try to convert the types."）；想换整个序列化器？官方直说 **"Nuxt does not currently support an alternative serializer to `JSON.stringify`."**。再加上体积这一头：`pick` 的定位是 "helps you to minimize the payload size stored in your HTML document"，载荷投递方式由 `experimental.payloadExtraction` 管（三种取值、默认 `true`、`compatibilityVersion: 5` 下是 `'client'`、`ssr: false` 强制 `false`），还有一个 `experimental.stripNeverHydratedData` 会给 `hydrate-never` 的组件默认套上 `serialize: false`。

categories:
  - 前端开发

tags:
  - nuxt4
  - payload
  - devalue
  - pick
  - 序列化

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先给两个日常症状，它们分别属于这两道关：接口返回了日期，前端拿到的却是字符串；本地一切正常，部署到静态托管后 `_payload.json` 请求 404。**这两个问题的根因不在同一层。**

## 两道关先分清

| 通道 | 序列化方式 | 能带什么 |
| --- | --- | --- |
| 服务端接口 → 浏览器（`$fetch`/`useFetch` 直接取到的响应） | `JSON.stringify` | "limited to only JavaScript primitive types" |
| 服务端渲染 → 水合（payload，含 `useAsyncData` 带过去的结果） | `devalue` | 正则、`Date`、`Map`/`Set`、`ref`/`reactive`/`shallowRef`/`shallowReactive`、`NuxtError` 等 |

官方那条 note 就是为了让读者不把第一道当成第二道：

> Note that this *does not apply* to data passed from your server routes when fetched with `$fetch` or `useFetch` - see the next section for more information.

**第一道症状的官方示例**（`server/api/foo.ts` 返回 `new Date()`）：

```vue [app/app.vue]
<script setup lang="ts">
// Type of `data` is inferred as string even though we returned a Date object
const { data } = await useFetch('/api/foo')
</script>
```

那句注释是我见过最省事的一句提醒：**类型是 `string`，不是你返回的 `Date`**。所以"日期不能 `.getTime()`"多半不是 Nuxt 的 bug，而是第一道只走 JSON。

## 第一道的两个出口

官方给的两个办法都不涉及换库。

**办法一：定义 `toJSON`。** 原话："To customize the serialization behavior, you can define a `toJSON` function on your returned object. If you define a `toJSON` method, Nuxt will respect the return type of the function and will not try to convert the types."

```ts [server/api/bar.ts]
export default defineEventHandler(() => {
  const data = {
    createdAt: new Date(),

    toJSON () {
      return {
        createdAt: {
          year: this.createdAt.getFullYear(),
          month: this.createdAt.getMonth(),
          day: this.createdAt.getDate(),
        },
      }
    },
  }
  return data
})
```

页面里的类型推断跟着变成那个结构（官方用注释把推断结果写了出来）。**关键在于"不再被转换"**——你交出什么形状、类型就是什么形状。

**办法二：换序列化器？官方直接说了不行。**

> Nuxt does not currently support an alternative serializer to `JSON.stringify`. However, you can return your payload as a normal string and utilize the `toJSON` method to maintain type safety.

它给的示例是用 superjson，但做法是"自己序列化、把结果当普通字符串返回"，即**换库不换通道**。我认为这条的真实含义是：想让复杂类型跨第一道活着，你得自己在字符串里包一层，Nuxt 不参与。

第二道（devalue）想扩展类型，官方指向的是 `useNuxtApp` 的 payload 一节（自定义 reducer/reviver），200 也记过那句"custom types (such as class instances) need payload plugins with custom reducers and revivers"。

## pick：唯一被明说"为体积而生"的选项

原文定位句：

> The `pick` option helps you to minimize the payload size stored in your HTML document by only selecting the fields that you want returned from the composables.

```vue
<script setup lang="ts">
/* only pick the fields used in your template */
const { data: mountain } = await useFetch('/api/mountains/everest', {
  pick: ['title', 'description'],
})
</script>
```

这句 "stored in your HTML document" 是我之前没意识到的：**没 `pick` 时，整个响应会被塞进首屏 HTML 里的 payload**——不是只在内存里。所以列表接口返回 30 个字段、模板只用 2 个的时候，HTML 体积差的是 30 倍字段量。

需要更复杂的形状（比如映射多个对象），官方给的下一句就是 `transform`："If you need more control or map over several objects, you can use the `transform` function to alter the result of the query."

218 讲过两者的顺序文档没说；但这里能补一条确定信息：**`pick` 是"选择字段"，`transform` 是"改形"**，所以想同时用，先自己确认最终 payload 里到底剩了什么。

## payloadExtraction：决定这份数据放在哪

`experimental.payloadExtraction` 一节（原文）：

> Controls how payload data is delivered for prerendered and cached (ISR/SWR) pages.
>
> - `'client'` - Payload is inlined in HTML for the initial server render, and extracted to `_payload.json` files for client-side navigation.
> - `true` - Payload is extracted to a separate `_payload.json` file for both the initial server render and client-side navigation.
> - `false` - Payload extraction is disabled entirely. Payload is always inlined in HTML and no `_payload.json` files are generated.
>
> The default is `true`, or `'client'` when `compatibilityVersion: 5` is set. It is forced to `false` when `ssr: false` is set.

三档取舍我用一句大白话记：**`'client'` = 首屏内联、导航走文件；`true` = 两次都走文件（HTML 更小，多一次请求）；`false` = 全都内联（没有文件）**。第二句里的代价官方在预渲染页写得更明确（200 抄过）：`true` 时 "The HTML is smaller and the payload file can be cached by a CDN, at the cost of one extra request on first load."

这也解释了开头那个"部署后 404"的症状：**默认 `true` 时客户端导航要额外请求 `_payload.json`**，如果托管层没有把这些文件一起交付（或路由规则没放行），导航就会失败。

另外它是**取数缓存的前置条件**（213 引过那句）："This only caches data when `experimental.payloadExtraction` in `nuxt.config` is enabled." ——所以 `ssr: false`（被强制 `false`）的项目里，默认 `getCachedData` 实现拿不到东西。

## 还有一把专管体积的实验开关

`experimental.stripNeverHydratedData` 的说明我只引一句就够，但它把两件事连上了：

> Apply `serialize: false` by default to `useAsyncData` and `useFetch` calls made within components lazily hydrated with `hydrate-never`, keeping their data out of the `__NUXT_DATA__` payload. Since these components never hydrate on the client, their data is not needed for hydration.

也就是说：**永远不会水合的组件，它取的数据没必要进 payload**，这个开关就是把 217 讲的 `serialize: false` 自动应用到那一类组件上。（`hydrate-never` 本身是 166 讲的延迟水合策略之一。）

## 一张排查表

| 症状 | 先看哪一层 |
| --- | --- |
| 接口返回 Date/Map，前端拿到 string 或普通对象 | 第一道（`JSON.stringify`）——考虑 `toJSON` 或前端重建 |
| 服务端渲染带过去的值类型丢了 | 第二道（devalue）——是否是需要 reducer/reviver 的自定义类型 |
| 首屏 HTML 特别大 | `pick`/`transform` 有没有用；`payloadExtraction` 是不是 `false` |
| 部署后客户端导航失败、请求 `_payload.json` 404 | `payloadExtraction: true` 与托管交付内容是否匹配（`'client'` 可让首屏不必取文件） |
| 缓存读不到 | `payloadExtraction` 是否被关（含 `ssr: false` 强制关） |

## 自查

1. 两次序列化分别用什么？官方划清界限的那句是什么？（答：接口→浏览器是 `JSON.stringify`；服务端渲染→水合是 `devalue`；note 明写"does not apply to data passed from your server routes when fetched with `$fetch` or `useFetch`"。）
2. 想只把用到的字段带进 HTML，用什么？（答：`pick`，官方说明是"minimize the payload size stored in your HTML document"。）
3. `payloadExtraction` 三档的取舍各是什么，默认值受什么影响？（答：见上；默认 `true`，`compatibilityVersion: 5` 下是 `'client'`，`ssr: false` 强制 `false`。）

## 下一篇

236 - 缓存的两把新开关：granularCachedData 与 purgeCachedData

## 本文依据

- `getting-started/data-fetching`：Serializing Data From Server to Client 全段（devalue 与类型清单、自定义序列化器指向 useNuxtApp、那条 does not apply 的 note）；Serializing Data From API Routes 段（`JSON.stringify`、"limited to only JavaScript primitive types"、"does its best to convert the return type"）；Example（`server/api/foo.ts` 返回 Date 与 `inferred as string` 注释）；Custom serializer function 段（`toJSON` 两句与 `bar.ts` 示例、推断结果注释）；Using an alternative serializer 段（"does not currently support" 与 superjson 做法）；Minimize payload size 段（`pick` 定位句与 `mountain` 示例）与 `transform` 那句；Caching and refetching 一节。
- `guide/going-further/experimental-features`：`payloadExtraction` 一节全文（三取值、默认与 `compatibilityVersion: 5`、`ssr: false` 强制 `false`、指向 prerendering 的 read-more）；`stripNeverHydratedData` 一节原文。
- `api/composables/use-async-data`：`pick`/`transform`/`serialize` 三行原文与 "This only caches data when `experimental.payloadExtraction` in `nuxt.config` is enabled."（213 已逐句引）。
- `getting-started/prerendering`：`_payload.json` 生成时机、CDN 缓存与"多一次请求"那句、devalue reducer/reviver 那条（200 已逐句引）。
- `guide/best-practices/performance`（lazy hydration）与延迟水合策略（166/167 已逐句引）。
