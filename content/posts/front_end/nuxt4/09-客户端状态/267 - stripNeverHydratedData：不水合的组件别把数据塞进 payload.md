---
url: /posts/026fe017d7c50a5bf4c6691e2dc0d7de/
title: stripNeverHydratedData：不水合的组件别把数据塞进 payload
date: 2027-01-09T09:00:00+08:00
updated: 2027-01-09T09:00:00+08:00
author: cmdragon

summary:
  摘要：264 说清了"往 payload 少塞东西"的动机，这一篇给一个自动化手段。有些组件服务端渲染出来、但客户端永不水合（懒水合策略选的是 hydrate-never）——它们取的数据本来就不需要在浏览器接手时用，却默认被序列化进了 HTML payload。实验开关 stripNeverHydratedData 就是针对这种组件，给它们内部的 useAsyncData/useFetch 默认套上 serialize:false，把数据留在服务端、不进 payload。这一篇讲它的作用、和"同 key 共享"的冲突、以及和 payloadExtraction 的分工。

categories:
  - 前端开发

tags:
  - nuxt4
  - stripNeverHydratedData
  - 水合
  - payload
  - 实验特性
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

SSR 页面首屏 HTML 里内联的 payload（264 讲过），主要是为了让浏览器**水合**时能读到和服务端一致的数据。那如果某个组件在浏览器里根本不会水合呢？给它塞进 payload 的数据就是纯浪费——白白加宽首屏 HTML。`stripNeverHydratedData` 处理的就是这个浪费。

## 先补一句：什么是"永不水合的组件"

Nuxt 有"懒水合"（lazy hydration）策略：一个组件可以声明成服务端把 HTML 渲染好，但**推迟甚至完全不**在浏览器里挂上交互。其中有一档叫 `hydrate-never`——渲染出来就固定了，客户端不接手、不水合。官方对懒水合的定位是"内容密集、只展示不交互"的组件少发 JS、少做水合工作。

对这种组件来说，它内部用 `useAsyncData`/`useFetch` 在服务端取到的数据，渲染完 HTML 就用途结束了：浏览器不会水合它，也就不需要那份数据。可默认行为仍会把数据序列化进 payload。这就是 `stripNeverHydratedData` 要省的东西。

## 这个开关具体做什么

实验特性那一页给它的话，把"作用对象"和"结果"都点明了：

> Apply `serialize: false` by default to `useAsyncData` and `useFetch` calls made within components lazily hydrated with `hydrate-never`, keeping their data out of the `__NUXT_DATA__` payload. Since these components never hydrate on the client, their data is not needed for hydration.

拆三层：

1. 它不是让你手写 `serialize: false`，而是**对特定组件里的取数调用默认施加 `serialize: false`**。`serialize: false`（235 篇讲过）的含义是"服务端照取，但不把结果写进 payload"。
2. 施加范围只限 **`hydrate-never` 组件内部**的 `useAsyncData`/`useFetch`。别的水合策略（比如只是延后水合的）不在它管辖内。
3. `__NUXT_DATA__` 就是内联在 HTML 里那段 payload 的名字——被剔掉数据的那段就是它。

开法照旧在 experimental 节：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    stripNeverHydratedData: true,
  },
})
```

和所有实验特性一样，官方统一提醒 "these features are experimental and could be removed or modified in the future"。

## 会打架的地方：同一个 key 被两类组件共享

这一节是它最容易出事的地方，原文有两句约束：

> An explicit `serialize` option always takes precedence. If the same key is shared with components that do hydrate, keep the `serialize` option consistent across calls (Nuxt warns about mismatches in development).

对应两种情况：

- **你显式写了 `serialize`**：那你的值赢，这个开关不覆盖它。
- **同一个 key 被"水合的组件"和"不水合的组件"共用**：这就麻烦了。前面 215 篇讲过，同 key 的 `useAsyncData`/`useFetch` 共享 data/error/status 等 refs、并且要求若干选项在各调用处保持一致。现在这个开关会给 `hydrate-never` 那侧偷偷加 `serialize: false`，而水合那侧默认还是进 payload——两边就不一致了。官方给的应对是**手动把这些调用的 `serialize` 选项写成一致**，且开发环境下不一致会警告。

所以实际用法是：既然这个开关会自动改 serialize，那么凡是"可能被水合组件共享的 key"，就别让它落在 `hydrate-never` 组件里；真要共存，就显式写 `serialize` 对齐，别依赖默认值。

## 别和 payloadExtraction 混为一谈

两个都在管 payload，管的却不是一个维度：

| 开关 | 管的是 |
| --- | --- |
| `stripNeverHydratedData` | **哪些数据**进 payload（按组件的水合策略筛掉不需要的） |
| `payloadExtraction` | payload **怎么送达**（内联进 HTML，还是抽成 `_payload.json` 文件） |

`payloadExtraction` 那一页的定义是"Controls how payload data is delivered for prerendered and cached (ISR/SWR) pages"，三档 `'client'`/`true`/`false`，默认 `true`（`compatibilityVersion: 5` 时为 `'client'`），`ssr: false` 时强制 `false`。它决定的是"放进 payload 的那部分数据，是内联还是外置成文件"，跟"这条数据该不该进 payload"是两回事。两者可以叠着用：先用 `stripNeverHydratedData` 把不要的数据剔出 payload，再用 `payloadExtraction` 决定剩下这些怎么送。

## 自查

- 你页面里有没有 `hydrate-never` 的展示型组件、且它们内部取数？这是 `stripNeverHydratedData` 的收益场景。
- 有没有一个 key 同时被"会水合"和"不会水合"的组件用？有就别靠默认，显式把 `serialize` 写一致。
- 开了之后首屏 HTML 里 `__NUXT_DATA__` 那段该变短——真要验收就在浏览器看首屏源码，别只看构建是否通过。
- 记住它是实验开关，升级版本时复核有没有改默认或改名。

## 下一篇

到这里 P09 客户端状态的具体机制基本讲完。下一篇 268 把 useState / useCookie / app.config / runtimeConfig / payload 这几种"状态该放哪儿"的选择，收进一张决策表。

## 本文依据

- Experimental Features（`guide/going-further/experimental-features`）：
  - `stripNeverHydratedData` 一节："Apply `serialize: false` by default to `useAsyncData` and `useFetch` calls made within components lazily hydrated with `hydrate-never`, keeping their data out of the `__NUXT_DATA__` payload. Since these components never hydrate on the client, their data is not needed for hydration."
  - "An explicit `serialize` option always takes precedence. If the same key is shared with components that do hydrate, keep the `serialize` option consistent across calls (Nuxt warns about mismatches in development)."
  - `payloadExtraction` 一节："Controls how payload data is delivered for prerendered and cached (ISR/SWR) pages."、三档取值、默认与 `ssr:false` 强制 `false`。
  - 页首 note：实验特性可能被移除或修改。
- useNuxtData（`api/composables/use-nuxt-data`）：同 key 共享缓存值、需显式 key 的前提。
- useAsyncData 参数（`api/composables/use-async-data`）：`serialize: false` 的既定语义（服务端取但不进 payload）。
