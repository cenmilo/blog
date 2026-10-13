---
url: /posts/093c951eb43e7a0837ee608847a09873/
title: 618 - Bridge 配置：四个显式开启项、七个默认项与两条旧选项的去向
date: 2027-05-06T09:00:00+08:00
updated: 2027-05-06T09:00:00+08:00
author: cmdragon

summary:
  摘要：`bridge` 这个键下面挂着**十一项开关**，官方把它们分成两组：四个**要显式打开**的（`vite`、`meta`、`macros.pageMeta`、`typescript.esbuild`）与七个**默认就开着、可以关掉**的（`nitro`、`app`、`capi`（还能细到 `capi.legacy`）、`transpile`、`scriptSetup`、`imports`、`constraints`）。这一节的态度写得很明白：**"In normal circumstances, it is always best to stick with defaults!"**，并且让你去 `bridge/src/module.ts` 看最新默认值——**那一节不给默认值表**。另一半内容更有用：**两条旧选项的确切去向**，`router.base` 搬进 `app.baseURL`、`build.publicPath` 搬进 `app.cdnURL`。

categories:
  - 前端开发

tags:
  - nuxt4
  - bridge配置
  - 特性开关
  - baseURL迁移
  - 升级迁移
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

Bridge 装完之后 `bridge: false` 改成正文，接下来一堆开关名字开始出现：`nitro`、`capi`、`scriptSetup`、`meta`。这一节的作用就是把它们分成"你要主动开的"和"已经开着、除非必要别关的"两组。

## 官方态度先摆出来

> You can optionally disable some features from bridge or opt-in to less stable ones. **In normal circumstances, it is always best to stick with defaults!**
>
> You can check bridge/src/module.ts for latest defaults.

两句合起来是一条重要提示：**这一节不给你权威默认值**，要准确值去看 `nuxt/bridge` 仓库里 `packages/bridge/src/module.ts`（页面给的就是这个源码链接）。所以**照注释块改配置之前，先确认你要动的那一项默认是什么**。

## 两组开关

官方示例把两组分开写，注释也逐条解释了每项管什么：

```ts [nuxt.config.ts]
import { defineNuxtConfig } from '@nuxt/bridge'

export default defineNuxtConfig({
  bridge: {

    // -- Opt-in features --

    // Use Vite as the bundler instead of webpack 4
    // vite: true,

    // Enable Nuxt 3 compatible useHead
    // meta: true,

    // Enable definePageMeta macro
    // macros: {
    //   pageMeta: true
    // },

    // Enable transpiling TypeScript with esbuild
    // typescript: {
    //   esbuild: true
    // },

    // -- Default features --

    // Use legacy server instead of Nitro
    // nitro: false,

    // Disable Nuxt 3 compatible `nuxtApp` interface
    // app: false,

    // Disable Composition API support
    // capi: false,

    // ... or just disable legacy Composition API support
    // capi: {
    //   legacy: false
    // },

    // Do not transpile modules
    // transpile: false,

    // Disable <script setup> support
    // scriptSetup: false,

    // Disable composables auto importing
    // imports: false,

    // Do not warn about module incompatibilities
    // constraints: false
  },

  vite: {
    // Config for Vite
  },
})
```

**四、七个的分配是这样的**：

| 组 | 键 | 打开后的作用（官方注释原文要点） |
| --- | --- | --- |
| Opt-in | `vite: true` | 用 Vite 当打包器，**而不是 webpack 4** |
| Opt-in | `meta: true` | 启用 Nuxt 3 兼容的 `useHead` |
| Opt-in | `macros: { pageMeta: true }` | 启用 `definePageMeta` 宏 |
| Opt-in | `typescript: { esbuild: true }` | 用 esbuild 转译 TypeScript |
| 默认开 | `nitro: false` | 关掉则**改用旧服务端而不是 Nitro** |
| 默认开 | `app: false` | 关掉 Nuxt 3 兼容的 `nuxtApp` 接口 |
| 默认开 | `capi: false` | 关掉 Composition API 支持 |
| 默认开 | `capi: { legacy: false }` | **只**关旧版 Composition API 支持 |
| 默认开 | `transpile: false` | 不再转译模块 |
| 默认开 | `scriptSetup: false` | 关掉 `<script setup>` 支持 |
| 默认开 | `imports: false` | 关掉组合式函数的自动导入 |
| 默认开 | `constraints: false` | 不再就模块不兼容发出警告 |

（第二组是七个键、八种写法，`capi` 有整块关与只关 legacy 两种形态。）

三点要从这张表里读出来：

- **`vite` 与 `nitro` 是两条独立决定**：换打包器（Vite 取代 webpack 4）与换服务端（Nitro 取代旧服务端）分别开关，**不是同一个动作**。
- **`capi` 与 `scriptSetup` 是分开的**：前者管 Composition API 支持，后者管 `<script setup>`；而配置页里还有一处细节——**`imports: false` 关的是组合式函数的自动导入**，与 Composition API 支持本身也不是一回事。
- **`constraints: false` 关的是警告**，不是功能。关掉它等于主动放弃"模块不兼容"的提示，排查时会少一条线索。

另外注意示例结尾：`bridge` 同级还写了一个顶层 `vite: {}` ——**给 Vite 的具体配置项放在顶层 `vite` 键里**，`bridge.vite: true` 只负责"换用 Vite"。这与配置参考里 `## vite` 那一节的定位一致。

## 两条旧选项的确切去向

`## Migration of Each Option` 下面只有两个 `###`，但都是最容易配错的两条：

**`router.base` → `app.baseURL`**

```diff
export default defineNuxtConfig({
- router: {
-   base: '/my-app/'
- }
+ app: {
+   baseURL: '/my-app/'
+ }
})
```

**`build.publicPath` → `app.cdnURL`**

```diff
export default defineNuxtConfig({
- build: {
-   publicPath: 'https://my-cdn.net'
- }
+ app: {
+   cdnURL: 'https://my-cdn.net'
+ }
})
```

两条都是"旧键在 `router`／`build` 下面，新键在 `app` 下面"，而且**语义上并不完全同名**：CDN 前缀这件事旧名字叫 `publicPath`，新名字是 `cdnURL`。这与 4.x 侧的记载吻合——`app.cdnURL` 与 `app.buildAssetsDir` 是**默认值 JSON 里就有的两项**，官方还写明资源目录名是"相对 `baseURL`（若设了 `cdnURL` 则相对它）"，并且**这一项在构建期定下、不该在运行时改**。

## 自查

- 开关共**十一项**：四个 opt-in（`vite`、`meta`、`macros.pageMeta`、`typescript.esbuild`）＋七个默认开（`nitro`、`app`、`capi`（可细到 `capi.legacy`）、`transpile`、`scriptSetup`、`imports`、`constraints`）。
- 官方态度是"默认最好别动"，**权威默认值在 `bridge/src/module.ts`，不在这一节**。
- `vite` 与 `nitro` 是两条独立决定；`capi`、`scriptSetup`、`imports` 三者也各管一段，别当成一个开关。
- `constraints: false` 只关警告。
- Vite 的具体配置写在**顶层 `vite`** 键里，与 `bridge.vite` 分工不同。
- 两条旧选项去向：**`router.base`→`app.baseURL`**、**`build.publicPath`→`app.cdnURL`**（名字也换了）。

## 下一篇

下一篇 619 读 Bridge 的 TypeScript 那一节：**先删两个模块**（`@nuxt/typescript-build` 因为 Bridge 已提供同功能、`@nuxt/typescript-runtime` 与 `nuxt-ts` 因为 Nuxt 2 自带运行时支持），再设 `bridge.typescript: true`（同一块里常要带 `nitro: false`），然后把 `tsconfig.json` 改成 `extends: "./.nuxt/tsconfig.json"`——这一步有个具体后果：**该文件是生成的、不入库，所以跑测试前要先 `nuxi prepare`，否则会看到 `TS5083: Cannot read file '~/.nuxt/tsconfig.json'`**。

## 本文依据

- https://nuxt.com/docs/4.x/bridge/configuration（`## Feature Flags` 两句定性与 `bridge/src/module.ts` 源码链接、整段带注释的配置示例（四个 opt-in 项与七个默认项的全部注释原文、`capi` 两种写法、末尾顶层 `vite: {}`）；`## Migration of Each Option` 下 `### router.base` 与 `### build.publicPath` 两条 diff 全文）
- https://nuxt.com/docs/4.x/api/nuxt-config（`### app.baseURL`／`### app.cdnURL`／`### app.buildAssetsDir` 的默认值与 "relative to `baseURL` (or `cdnURL` if set)…should not be customized at runtime" 那句，用于印证两条旧选项的落点）
