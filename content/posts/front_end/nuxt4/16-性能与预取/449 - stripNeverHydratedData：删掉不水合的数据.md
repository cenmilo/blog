---
url: /posts/ecf2ef9fb7fccdd72f0a28b5ab7788b7/
title: 449 - stripNeverHydratedData：删掉不水合的数据
date: 2027-03-10T11:00:00+08:00
updated: 2027-03-10T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：标了 `hydrate-never` 的组件在客户端永远不水合，但它内部 `useAsyncData`/`useFetch` 的结果照样被序列化进 `__NUXT_DATA__`——一份没人用的数据。`experimental.stripNeverHydratedData` 就是删这一份：它对这类调用默认施加 `serialize: false`，把数据留在服务端渲染结果里、不写进载荷。官方同时留了三条限制：显式写的 `serialize` 永远优先；同一个 key 若与被水合的组件共用，`serialize` 必须保持一致（开发模式会就不一致报警）；而组件属性一旦变化会**立即触发水合**，`hydrate-never` 也不例外。

categories:
  - 前端开发

tags:
  - nuxt4
  - 水合
  - payload
  - experimental
  - 性能
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

一段纯展示的页脚免责声明，你给它标了 `hydrate-never`——客户端不再水合它，很好。但查看源码会发现，它内部那次取数的结果仍然完整躺在 `__NUXT_DATA__` 里。

**没人会读它**：这个组件在浏览器里永远不激活，那份数据不会被拿去水合任何东西。`experimental.stripNeverHydratedData` 删的就是这一段。

## 官方一句话里有四个要素

原文：

> Apply `serialize: false` by default to `useAsyncData` and `useFetch` calls made within components lazily hydrated with `hydrate-never`, keeping their data out of the `__NUXT_DATA__` payload. Since these components never hydrate on the client, their data is not needed for hydration.

拆开：

| 要素 | 内容 |
| --- | --- |
| 动作 | 默认施加 `serialize: false` |
| 作用对象 | 在**被 `hydrate-never` 懒水合的组件里**发起的 `useAsyncData` 与 `useFetch` 调用 |
| 结果 | 这些数据**不进 `__NUXT_DATA__` 载荷** |
| 理由 | 这些组件在客户端永不水合，数据对水合没用 |

配置写法照官方示例，就一行的事：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    stripNeverHydratedData: true,
  },
})
```

这一节没有"This flag is enabled by default"那句（对照 448 讲的 `renderJsonPayloads` 和 `purgeCachedData` 都有），所以按**默认关、要显式开**处理。

## 前提：这个组件得真的是 never

`hydrate-never` 在延迟水合那节里的定义只有三个字："Never hydrates the component."，用法是 `<LazyMyComponent hydrate-never />`。

但它前面有一条 `::note`，正拿它举例：

> Any prop change on a lazily hydrated component will trigger hydration immediately. (e.g., changing a prop on a component with `hydrate-never` will cause it to hydrate)

也就是说，**"永不水合"的条件是没人改它的属性**。父组件一旦改动 prop，这个组件当场就水合了——而此刻它需要的数据可能已经因为这个开关被删出载荷。

官方给的处理办法不在这一句里，而在开关自身那段的后半句：

> An explicit `serialize` option always takes precedence. If the same key is shared with components that do hydrate, keep the `serialize` option consistent across calls (Nuxt warns about mismatches in development).

两句、两条规则：

**显式优先**。你在某一次调用里自己写了 `serialize`，那它的值压过这个全局开关。

**同 key 必须同 `serialize`**。如果这个 key 同时被别的、会被水合的组件用，两边的 `serialize` 要一致；开发模式下不一致会警告。

## 还有一层：懒水合自身的适用限制

延迟水合那一节有一条 `::warning`，限定的是这套机制本身，跟这个开关能不能生效绑在一起：

> Currently Nuxt's built-in lazy hydration only works in single-file components (SFCs), and requires you to define the prop in the template (rather than spreading an object of props via `v-bind`). It also does not work with direct imports from `#components`.

三条：**只在单文件组件里工作**、**属性要写在模板上**（不能靠 `v-bind` 展开一个对象）、**从 `#components` 直接导入的组件不走这套**。

这三条里任何一条不满足，那个组件就不是"被 `hydrate-never` 懒水合的组件"，这个开关对它内部的取数调用自然也不会施加 `serialize: false`。

## 什么时候值得开

把条件列全，判断就具体了：

- 你**已经在用** `hydrate-never`，而且这些组件内部**确实有取数调用**。没有这类组件，这个开关无事可做。
- 这些组件的 prop 在运行期**不会变**（否则会水合，见上）。
- 同一个 key 没有被别的会水合的组件共用（否则 `serialize` 不一致会警告）。

三条都满足，它删掉的才只是死数据。这个开关的作用面很窄——它是给"已经决定永不水合"的组件收尾的，不是给项目普遍减载荷的。真正普适的载荷控制是 448 那些。

## 自查

- 动作是给 `hydrate-never` 组件内的 `useAsyncData`/`useFetch` 默认加 `serialize: false`，数据不进 `__NUXT_DATA__`。
- 该节无默认开启句，按显式开处理；显式写出的 `serialize` 永远优先。
- 同一个 key 跨"会水合"与"永不水合"的组件时，`serialize` 必须一致，开发模式会警告不一致。
- `hydrate-never` 不是绝对：属性一变立刻水合。
- 懒水合本身要求 SFC、模板上写属性、不能从 `#components` 直接导入，这三条决定了开关有没有对象。

## 下一篇

下一篇 450 把视角从载荷挪到首字节：`experimental.ssrStreaming` 让服务端立刻发出 HTML 壳（含 `<head>`、样式、预加载提示与入口脚本），再用 Vue 的 `renderToWebStream` 把渲染出的正文一段段推出去，官方说这"显著提升 TTFB"。代价也写得硬：状态与响应头必须在壳冲出去之前定好，之后写的都到不了客户端。
