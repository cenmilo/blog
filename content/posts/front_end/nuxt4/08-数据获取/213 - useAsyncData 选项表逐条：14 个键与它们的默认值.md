---
url: /posts/b2d0248b9168ba8747cd8cf2c329bb5f/
title: 213 - useAsyncData 选项表逐条：14 个键与它们的默认值
date: 2026-12-22T09:00:00+08:00
updated: 2026-12-22T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇把 `useAsyncData` 的第三个参数（`options`）整张表逐行抄完：**14 个键**，官方在表格前面写了一句适用范围很宽的话——"options (object): Configuration for the asynchronous function call. All options can be a static value, a `ref`, or a computed value."，表后又补了后果："All options can be given a `computed` or `ref` value. These will be watched and new requests made automatically with any new values if they are updated."。五个有默认值的键是 `server` `true`、`lazy` `false`、`immediate` `true`、`deep` `false`、`dedupe` `'cancel'`、`enabled` `true`、`serialize` `true`（其余七个没有默认值，表里写的是 `-`）；带版本徽标的四个是 `timeout` v4.2、`getCachedData` v3.8、`deep` v3.8、`dedupe` v3.9、`enabled` v4.5、`serialize` v4.6、`middleware` v4.6。两条我必须单独标出来：一，`getCachedData` 的默认实现只在 `nuxtApp.isHydrating` 时读 `payload.data[key]`、否则读 `static.data[key]`，而官方紧跟一句"**This only caches data when `experimental.payloadExtraction` in `nuxt.config` is enabled.**"；二，`lazy: false` 不是"不异步"，而是官方说的"Under the hood, `lazy: false` uses `<Suspense>` to block the loading of the route before the data has been fetched"，并且官方建议反过来——"Consider using `lazy: true` and implementing a loading state instead for a snappier user experience."

categories:
  - 前端开发

tags:
  - nuxt4
  - useAsyncData
  - 选项
  - 缓存
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


212 讲清了参数怎么排，这一篇把第三个参数里能放的东西一次列全。表里每个词我都照官方原文写，**没有默认的键我就留 `-`**，不替它编一个。

## 十四行选项表（逐行）

| 选项 | 类型 | 默认 | 官方说明 |
| --- | --- | --- | --- |
| `server` | `boolean` | `true` | "Whether to call the function on the server." |
| `lazy` | `boolean` | `false` | "If true, resolves after route loads (does not block navigation)." |
| `immediate` | `boolean` | `true` | "If false, prevents function from being called immediately." |
| `default` | `() => DataT` | - | "Factory for default value of `data` before async resolves." |
| `timeout` :v4.2 | `number` | - | "A number in milliseconds to wait before timing out the call (defaults to `undefined`, which means no timeout)" |
| `transform` | `(input: DataT) => DataT \| Promise<DataT>` | - | "Function to transform the result after resolving." |
| `getCachedData` :v3.8 | `(key, nuxtApp, ctx) => DataT \| undefined` | - | "Function to return cached data. See below for default." |
| `pick` | `string[]` | - | "Only pick specified keys from the result." |
| `watch` | `MultiWatchSources` | - | "Array of reactive sources to watch and auto-refresh." |
| `deep` :v3.8 | `boolean` | `false` | "Return data in a deep ref object. Defaults to `false` for improved performance (shallow ref object)." |
| `dedupe` :v3.9 | `'cancel' \| 'defer'` | `'cancel'` | "Policy when triggering an execution more than once at a time." |
| `enabled` :v4.5 | `boolean` | `true` | "Barrier that gates whether the `handler` may run."（长说明见下） |
| `serialize` :v4.6 | `boolean` | `true` | "Whether to store resolved data in the Nuxt payload (`__NUXT_DATA__`)."（长说明见下） |
| `middleware` :v4.6 | `AsyncDataMiddleware[]` | - | "Functions wrapping the execution of the `handler`."（长说明见下） |

`:vX` 是官方在那个键名后面挂的版本徽标，意思是"该取值从那个版本起才有"。我把它抄在表里，是因为**低版本上写这些键不会报错但可能没作用**——徽标是我判断能不能用的唯一依据。

## 表格外面那两句话

这两句比表本身更有用，因为它们决定了"改选项"这件事的语义：

- 表前：**"All options can be a static value, a `ref`, or a computed value."**
- 表后（note）："All options can be given a `computed` or `ref` value. **These will be watched and new requests made automatically with any new values if they are updated.**"

也就是说：**选项给成响应式，值一变就自动重发请求**。216 会讲这条与 `watch` 的关系，以及官方在哪里给了关掉它的开关（`useFetch` 那一节有 `watch: false`，`useAsyncData` 这一节的 `watch` 行**类型里没有 `false`**——这个不对称我留到 216 单独说）。

## 三条长说明，值得整段读

`enabled`（v4.5）原文，四条行为压在一句话里：

> Barrier that gates whether the `handler` may run. While `false`, every execution is blocked (initial fetch, `execute`/`refresh`, and watch triggers), and switching `true` → `false` cancels any in-flight request without clearing `data`. Re-enabling does not refetch on its own.

拆开是四件事：`false` 时**首次请求、手动 refresh、watch 触发**三种都会被挡住；从 `true` 切到 `false` 会取消进行中的请求但**不清空 `data`**；再切回 `true` **不会自动重新取**。最后这条最容易误判——很多人以为 `enabled` 重新变真是"闸门开了就放一次请求过去"，官方明说不会。

`serialize`（v4.6）原文：

> Whether to store resolved data in the Nuxt payload (`__NUXT_DATA__`). When `false`, server-fetched data is kept out of the payload and the client will refetch after hydration if a component renders it. Pair with lazy hydration to avoid hydration mismatches and unnecessary client fetches.

这一条正好接上 202 之前讲的 payload 机制（210）：`serialize: false` 是**把某块数据从 payload 里摘出去**，代价是水合后若真有组件在渲染它，客户端会再取一次。官方还给了配对建议——和延迟水合（`guide/best-practices/performance` 的 lazy hydration 那一节）搭配用，避免水合不匹配与多余请求。

`middleware`（v4.6）原文：

> Functions wrapping the execution of the `handler`. Each receives `next`, which runs the rest of the chain and resolves to the data, and the handler context (`{ signal }`). The first entry is the outermost wrapper. Call `next()` to continue or throw to abort.

它的类型是 `AsyncDataMiddleware<ResT> = (next: () => Promise<ResT>, ctx: { signal: AbortSignal }) => Promise<ResT>`。形状就是"洋葱"：数组第一项包在最外面，每一项要么调 `next()` 放行、要么抛错中止。218 里我会把它的顺序与 `transform`、`pick` 的关系放在一起谈。

## 默认缓存实现，和它的前提

`getCachedData` 那行写着"See below for default"，下面给的是：

```ts [Default getCachedData Implementation]
const getDefaultCachedData = (key, nuxtApp, ctx) => nuxtApp.isHydrating
  ? nuxtApp.payload.data[key]
  : nuxtApp.static.data[key]
```

读法：**水合期间读 `payload.data`，其他时候读 `static.data`**。这也解释了为什么客户端导航时的"缓存"和首屏的"缓存"是两个存放点（`payload` 是一次加载带过来的数据、`static` 是运行期攒下的数据）。

然后是最要紧的那句限定：

> This only caches data when `experimental.payloadExtraction` in `nuxt.config` is enabled.

`payloadExtraction` 的三种取值与默认（`true`，`compatibilityVersion: 5` 下是 `'client'`，`ssr: false` 时强制 `false`）我在 200 抄过。**所以 `ssr: false` 的项目里默认这套缓存实现拿不到东西**——这是把两句原文叠起来才看得出的结论，单看任一句都会漏。

## lazy 那一档的真相

最后一句 note 我觉得该放在显眼位置，因为它是官方少见的"建议你反过来用"：

> Under the hood, `lazy: false` uses `<Suspense>` to block the loading of the route before the data has been fetched. Consider using `lazy: true` and implementing a loading state instead for a snappier user experience.

同页 read-more 补了等价写法："You can use `useLazyAsyncData` to have the same behavior as `lazy: true` with `useAsyncData`."（224 会读那一节确认这句。）

也就是说默认的 `lazy: false` 换来的是"导航被阻塞、落地即满数据"，官方认为**带加载状态的体验更轻**（210 那句"Neither approach is universally better"是这个意思的另一半：不站队，但倾向说了）。

## 自查

1. 表里哪七个键没有默认值？（答：`default`、`timeout`、`transform`、`getCachedData`、`pick`、`watch`、`middleware`。）
2. `enabled` 从 `false` 切回 `true` 会发生什么？（答：官方写明 "Re-enabling does not refetch on its own."——不会自动重取。）
3. 默认缓存实现依赖哪个配置？它对 `ssr: false` 的项目意味着什么？（答：`experimental.payloadExtraction`；`ssr: false` 时它被强制 `false`，默认实现拿不到缓存。）

## 下一篇

214 - 返回值七个成员：data、status、pending、error、refresh、execute、clear

## 本文依据

- `api/composables/use-async-data`：Parameters 一节 `options` 说明句；14 行选项表全部原文（含各键类型、默认值、七个版本徽标）；表后 note（computed/ref 会被监听并自动重发）；`enabled`、`serialize`、`middleware` 三条长说明原文；`AsyncDataOptions` 与 `AsyncDataMiddleware`、`AsyncDataRequestContext` 类型定义；"getCachedData default" 代码块与 "This only caches data when `experimental.payloadExtraction` in `nuxt.config` is enabled."；lazy/Suspense 那条 note 与 `useLazyAsyncData` 的 read-more。
- `getting-started/prerendering`：`experimental.payloadExtraction` 三取值与默认、`ssr: false` 强制 `false`（200 已逐句引）。
- `guide/best-practices/performance`：`serialize: false` 那句指向的 lazy hydration 小节（165/167 已细讲）。
