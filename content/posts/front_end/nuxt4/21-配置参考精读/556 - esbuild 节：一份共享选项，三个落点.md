---
url: /posts/e183ecf37d0f84ba6a1d23986e691c61/
title: esbuild 节：一份共享选项，三个落点
date: 2027-04-15T10:00:00+08:00
updated: 2027-04-15T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：`esbuild` 这一节只有一个子项 `options`，说明句同时交代了它的两个身份：**"Configure shared esbuild options used within Nuxt and passed to other builders, such as Vite or webpack."**——既是 Nuxt 内部用的，也会转交给别的构建器。四个字段与默认值：`target` **`"esnext"`**、`jsxFactory` **`"h"`**、`jsxFragment` **`"Fragment"`**、`tsconfigRaw`（类型 `object`，**无默认值**）。同一份四项默认值在文档里出现三处、形状却不一样：**顶层要写成 `esbuild: { options: { ... } }`（多一层）**，而 `vite.esbuild` 与 `webpack.loaders.esbuild` 都是直接一个对象。**这一节没有任何解释说明"改它会影响什么"**，也没写 `target: "esnext"` 与浏览器兼容目标之间的关系。

categories:
  - 前端开发

tags:
  - nuxt4
  - esbuild
  - 构建
  - jsx
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`jsxFactory` 这个键在项目里出现了三次：`esbuild` 下、`vite.esbuild` 下、`webpack.loaders.esbuild` 下。

三份默认值一模一样，但**只有一处是"共享源"**。

## 这一节的结构：多一层 options

`## esbuild` 下面**没有直接给类型和默认值**，只有一个 `### options`：

> Configure shared esbuild options used within Nuxt and passed to other builders, such as Vite or webpack.

它的四个子项：

| 子项 | 类型 | 默认 | 说明句 |
| --- | --- | --- | --- |
| `jsxFactory` | `string` | `"h"` | 无 |
| `jsxFragment` | `string` | `"Fragment"` | 无 |
| `target` | `string` | `"esnext"` | 无 |
| `tsconfigRaw` | `object` | 未列 | 无 |

**四项里三项有默认值，全部没有说明句。** 这一节给的信息量就是"键名＋类型＋默认值"，作用、影响面一概没写。

由于结构是多一层的，写法是：

```ts
export default defineNuxtConfig({
  esbuild: {
    options: {
      target: 'es2020',
    },
  },
})
```

**上面这段的嵌套形状来自文档（`## esbuild` → `### options`），但官方在这一节没有给任何 Example**——`target: 'es2020'` 这个值是我为演示填的，**不是文档推荐值**；文档给的默认是 `"esnext"`。

## 三处同名，形状不同

同一份四项默认值在配置参考里出现三次：

| 位置 | 写法形状 | 默认 |
| --- | --- | --- |
| 顶层 `esbuild.options` | 多一层 `options` | `{target:"esnext", jsxFactory:"h", jsxFragment:"Fragment", tsconfigRaw:{}}` |
| `vite.esbuild` | 直接一个对象（`object` 类型） | 同一份四项 |
| `webpack.loaders.esbuild` | 直接一个对象（`object` 类型） | 同一份四项，See 指向 esbuild-loader |

三份默认逐字相同，**但 `options` 那一层只有顶层那份有。** 这类形状差异最容易写错：把 `esbuild: { target: 'es2020' }` 这样直接写（少了 `options`），字段就不在文档记载的位置上了。

至于它们的关系——`esbuild.options` 的说明句已经给了答案：**它 "passed to other builders, such as Vite or webpack"**，也就是**共享源在顶层，两个构建器侧是接收方**。

不过**这一句没有回答"改构建器侧那份会怎样"**：`vite.esbuild` 与 `webpack.loaders.esbuild` 各自也有默认值，说明它们可以独立于顶层被设。**两处同时存在时以哪一份为准，官方没有写。** 稳妥做法是只改顶层那一份（它的定位就是"shared"），改完看产物是否变化来确认转交是否发生。

## `target: "esnext"` 这条默认值要怎么看

这一项最容易被当成"降级目标"来配。但要说清能确证到哪一步：

- 官方给的默认是字符串 `"esnext"`，**这一节没有任何一句解释它的效果**。
- 它与浏览器兼容范围的关系、改小之后产物会怎么变，**文档都没写**。
- 同一节里 `vite` 的 `esbuild.target`、`webpack.loaders.esbuild.target` 也都是 `"esnext"`，也就是说**两条构建链的默认都是"不降级"**。

能安全说出口的只有一句：**默认值不会把语法降到旧目标**。至于"要不要按你的浏览器支持范围改它"，这需要你自己的兼容性目标来决定，文档没给建议，也没有给出可选值清单（`esnext` 之外能填什么字符串，本节没写）。

## jsxFactory 与 jsxFragment：与自定义 pragma 有关

`"h"` 与 `"Fragment"` 这两个默认值成对出现，形状就是"JSX 工厂函数名＋片段组件名"。

**这一节没有解释这两个键做什么**，能确证的只有默认值本身；要追到实际效果，得配合两处已记载的东西看：`optimizeDeps.esbuildOptions`（Vite 侧依赖预优化用的也是这同一份四项）、以及 esbuild-loader 的文档（`webpack.loaders.esbuild` 那条 See 指过去的地方）。

一句提醒：**若你的项目里没有用 JSX，这两项就是死配置**；真要改 JSX pragma，官方在这个层级没给任何示例，改完必须实际构建一次确认产物里的调用变了。

## 自查

- `## esbuild` 只有一个子项 `options`，说明句点明两重身份：Nuxt 内部使用＋转交给别的构建器（Vite、webpack）；四个字段是 `target`/`jsxFactory`/`jsxFragment`/`tsconfigRaw`，**全部没有说明句**，`tsconfigRaw` 也没有默认值。
- 写法多一层：`esbuild: { options: { ... } }`；`vite.esbuild` 与 `webpack.loaders.esbuild` 是直接对象，三份默认值逐字相同。
- 顶层是"共享源"，构建器侧是接收方；**两边同时设时谁优先，文档未写**——只改顶层最稳。
- 默认 `target: "esnext"` 只能读成"默认不降级语法"，可选值清单与影响面文档都没给。
- 官方在这一节没有 Example；任何具体值（包括本文演示的）都不是文档推荐值。

## 下一篇

下一篇 557 读 `builder`：默认 `"@nuxt/vite-builder"`、可写 `'vite'`/`'webpack'`/`'rspack'` 三个简写，换 webpack 或 rspack **必须自己显式装对应包**，还能传一个带 `bundle` 函数的对象或一个导出 `bundle` 的包名。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## esbuild` 与 `### options` 及其四个 `####` 子项：shared esbuild options／passed to other builders 的说明句、`jsxFactory` `"h"`、`jsxFragment` `"Fragment"`、`target` `"esnext"`、`tsconfigRaw` 仅 `object` 无默认；对照 `### vite.esbuild` 与 `#### webpack.loaders.esbuild` 的同一份四项默认与 esbuild-loader 的 See、`### vite.optimizeDeps.esbuildOptions` 同一份默认）
