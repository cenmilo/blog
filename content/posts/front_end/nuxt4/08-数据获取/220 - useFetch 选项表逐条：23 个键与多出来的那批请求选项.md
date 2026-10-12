---
url: /posts/ed9dd2eaccd9edd0f814601727e725d1/
title: useFetch 选项表逐条：23 个键与多出来的那批请求选项
date: 2026-12-24T10:00:00+08:00
updated: 2026-12-24T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇把 `useFetch` 的选项表整张抄完：**23 行**，官方在表格前写明它的来源——"options (object): Configuration for the fetch request. **Extends unjs/ofetch options and `AsyncDataOptions`.** All options can be a static value, a `ref`, or a computed value."也就是说这张表是"取数那一半（213 的 14 个键）+ 请求那一半"合起来的。请求那一半共八个键是 `useAsyncData` 没有的：`method`（默认 `'GET'`，"HTTP request method."）、`query`（"Query/search params to append to the URL. **Alias: `params`.**"）、`params`（"Alias for `query`."）、`body`（"Request body. Objects are automatically stringified."）、`headers`（三种形态都合法：`Record<string, string> | [key, value][] | Headers`）、`baseURL`、`cache`（`false | string`，"Boolean disables cache, or use Fetch API values: `default`, `no-store`, etc."）、`$fetch`（v3.2，"Custom $fetch implementation."）。两个 alias 关系我特别在意，因为 `query` 与 `params` 同时给会怎样文档没说；表后那条 note 也比 `useAsyncData` 版多了一个条件："These will be watched and new requests made automatically with any new values if they are updated (**unless `watch` is set to `false`**)."。带版本徽标的八个键我按官方原样标在行内。

categories:
  - 前端开发

tags:
  - nuxt4
  - useFetch
  - 选项
  - ofetch
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先说明一句读法：这张表我按官方的顺序抄，**分成两组看**会清楚很多——上面八个是"怎么发这个请求"，下面十五个是"这份数据怎么管"。

## 请求那一组（8 行，useAsyncData 没有）

| 选项 | 类型 | 默认 | 官方说明 |
| --- | --- | --- | --- |
| `method` | `MaybeRefOrGetter<string>` | `'GET'` | "HTTP request method." |
| `query` | `MaybeRefOrGetter<SearchParams>` | - | "Query/search params to append to the URL. Alias: `params`." |
| `params` | `MaybeRefOrGetter<SearchParams>` | - | "Alias for `query`." |
| `body` | `MaybeRefOrGetter<RequestInit['body'] \| Record<string, any>>` | - | "Request body. Objects are automatically stringified." |
| `headers` | `MaybeRefOrGetter<Record<string, string> \| [key, value][] \| Headers>` | - | "Request headers." |
| `baseURL` | `MaybeRefOrGetter<string>` | - | "Base URL for the request." |
| `cache` | `false \| string` | - | "Cache control. Boolean disables cache, or use Fetch API values: `default`, `no-store`, etc." |
| `$fetch` :v3.2 | `typeof globalThis.$fetch` | - | "Custom $fetch implementation. See Custom useFetch in Nuxt." |

几个值得停一下的点：

- **几乎每个请求选项的类型都是 `MaybeRefOrGetter<...>`**，也就是值本身可以是 ref 或 getter。这和表后那条 note 合起来才完整：**给成响应式的选项会被监听，值一变就重发请求**。
- `query` 与 `params` 互为别名。同时给两个会怎样，文档没写；`body` 那句"Objects are automatically stringified"（以及 `query` 那段说明里的同一句话）是官方对序列化行为唯一的交代。
- `cache` 是唯一一个类型写成 `false | string` 的：布尔用来关掉，字符串取的是 Fetch API 那套值（官方点名 `default`、`no-store`，其余用 "etc." 带过——**完整清单在浏览器标准的 `Request.cache` 里，不在这一页**）。
- `$fetch` 那行是"换实现"的口子，官方直接指向自定义 useFetch 那篇配方（222 会用到它）。

## 数据那一组（15 行，与 useAsyncData 同源）

| 选项 | 类型 | 默认 | 官方说明 |
| --- | --- | --- | --- |
| `key` | `MaybeRefOrGetter<string>` | auto-gen | "Unique key for de-duplication. If not provided, generated from the URL, options and call site location in the source code." |
| `server` | `boolean` | `true` | "Whether to fetch on the server." |
| `lazy` | `boolean` | `false` | "If true, resolves after route loads (does not block navigation)." |
| `immediate` | `boolean` | `true` | "If false, prevents request from firing immediately." |
| `default` | `() => DataT` | - | "Factory for default value of `data` before async resolves." |
| `timeout` :v4.2 | `number` | - | "A number in milliseconds to wait before timing out the request (defaults to `undefined`, which means no timeout)" |
| `transform` | `(input: DataT) => DataT \| Promise<DataT>` | - | "Function to transform the result after resolving." |
| `getCachedData` :v3.8 | `(key, nuxtApp, ctx) => DataT \| undefined` | - | "Function to return cached data. See below for default." |
| `pick` | `string[]` | - | "Only pick specified keys from the result." |
| `watch` | `MultiWatchSources \| false` | - | "Array of reactive sources to watch and auto-refresh. `false` disables watching." |
| `deep` :v3.8 | `boolean` | `false` | "Return data in a deep ref object. Defaults to `false` for improved performance (shallow ref object)." |
| `dedupe` :v3.9 | `'cancel' \| 'defer'` | `'cancel'` | "Avoid fetching same key more than once at a time." |
| `enabled` :v4.5 | `boolean` | `true` | "Barrier that gates whether the request may run."（四句长说明见 218） |
| `serialize` :v4.6 | `boolean` | `true` | "Whether to store resolved data in the Nuxt payload (`__NUXT_DATA__`)." |
| `middleware` :v4.6 | `AsyncDataMiddleware[]` | - | "Functions wrapping the execution of the request." |

**这一组里和 `useAsyncData` 措辞不同的有四处**，都不是小改动，我逐条比：

| 键 | `useFetch` 版本 | `useAsyncData` 版本 |
| --- | --- | --- |
| `watch` | 类型带 `\| false`，说明多一句 "`false` disables watching." | 类型只有 `MultiWatchSources` |
| `dedupe` | "Avoid fetching **same key** more than once at a time." | "Policy when triggering an execution more than once at a time." |
| `immediate` | "prevents **request** from firing immediately" | "prevents **function** from being called immediately" |
| `middleware` | "Functions wrapping the execution of the **request**" | "... execution of the **handler**" |

这些差异我的读法是：**用词跟着各自的模型走**（`useFetch` 眼里是一次 request，`useAsyncData` 眼里是一个 handler 函数），**行为描述没有因此改变**。真正有语义差的第一行——只有 `useFetch` 那页写明可以用 `watch: false` 关掉自动重取。

## 表后那条 note 多了半句

`useFetch`：

> All fetch options can be given a `computed` or `ref` value. These will be watched and new requests made automatically with any new values if they are updated (unless `watch` is set to `false`).

`useAsyncData` 的同位置那条没有括号那半句（216 抄过）。这半句正好是 216 那个不对称的正面证据。

## 默认缓存实现，两边一字不差

```ts [Default getCachedData Implementation]
const getDefaultCachedData = (key, nuxtApp, ctx) => nuxtApp.isHydrating
  ? nuxtApp.payload.data[key]
  : nuxtApp.static.data[key]
```

> This only caches data when `experimental.payloadExtraction` in `nuxt.config` is enabled.

两页的这块内容完全相同，包括那句限定。**所以我现在把"缓存能不能命中"当成配置层问题而不是取数函数选择问题**（213 里那条与 `ssr: false` 的连带关系同样适用于这里）。

## 自查

1. `useFetch` 比 `useAsyncData` 多出的八个键分别是什么？（答：`method`、`query`、`params`、`body`、`headers`、`baseURL`、`cache`、`$fetch`。）
2. 关掉自动重取这个开关在哪个组合式的文档里写明了？（答：`useFetch` 的 `watch: false`；`useAsyncData` 那页的 `watch` 类型没有 `false`。）
3. `query` 与 `params` 的关系是什么，文档留了什么空白？（答：互为别名；同时给两个会怎样没写。）

## 下一篇

221 - 查询参数三种写法：query 与 params、响应式 URL 与四个拦截器

## 本文依据

- `api/composables/use-fetch`：Parameters 两句（`URL` 类型与 "Extends unjs/ofetch options and `AsyncDataOptions`. All options can be a static value, a `ref`, or a computed value."）；23 行选项表全部原文（含 8 个版本徽标与 `key` 行的 auto-gen 说明）；表后 note（含 "unless `watch` is set to `false`"）；"getCachedData default" 代码块与 `payloadExtraction` 限定句；`AsyncDataRequestContext`、`AsyncData`、`AsyncDataExecuteOptions`、`AsyncDataRequestStatus` 定义；`createUseFetch` 的 tip（"pre-defined defaults (like `baseURL` or auth headers）"）。
- `api/composables/use-async-data`：14 行选项表原文，用于逐条措辞对照（`watch`、`dedupe`、`immediate`、`middleware` 四行差异）。
- `getting-started/prerendering`：`experimental.payloadExtraction` 取值与默认（200 已逐句引）。
