---
url: /posts/cc2067a7db685830535cac4994f05d28/
title: 429 - 样式相关的构建配置：vite 与 postcss 的交界
date: 2027-03-04T09:00:00+08:00
updated: 2027-03-04T09:00:00+08:00
author: cmdragon

summary:
  摘要：样式在构建侧分属两个互不嵌套的节：`vite` 与 `postcss`。`vite` 的定性是"Configuration that will be passed directly to Vite"，紧跟一句限制"not all vite options are supported in Nuxt"；预处理器配置就落在它下面（`vite.css.preprocessorOptions`、`vite.css.preprocessorMaxWorkers`），而**环境分流用 `$client` / `$server`**（官方："Top-level `vite` options are shared across both client and server environments. Use `$client` and `$server` to provide environment-specific configuration that will be merged into their respective builds."）。`postcss` 则是**顶层另一节**，只有 `order`（排 PostCSS 插件的策略）与 `plugins` 两个键，不嵌在 vite 里。还要记住两条前提：独立的 `vite.config.ts`/`postcss.config.js` 不被读取（必须写进 nuxt.config 对应键）；以及这一切只在 Vite 构建器下成立——`features.inlineStyles` 官方就明写"only when using Vite"，模块作者要跨构建器得走 `addBuildPlugin`（unplugin）而不是 `addVitePlugin`。

categories:
  - 前端开发

tags:
  - nuxt4
  - vite节
  - postcss节
  - client-server分流
  - 构建器
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

调样式相关构建配置时最容易犯的错，是把 PostCSS 塞进 `vite` 里——它们在 `nuxt.config` 是**平级的两节**。这一篇把这条界线与两侧的可用范围画出来。

## 两节的定位不同

| 节 | 官方定性 | 样式相关的是什么 |
| --- | --- | --- |
| `vite` | "Configuration that will be passed directly to Vite." | `vite.css.preprocessorOptions`（418）、`vite.css.preprocessorMaxWorkers` |
| `postcss` | 只有两个键：`order`、`plugins` | 插件链与插件顺序（419） |

第一条前提也在这：`vite.config.ts`、`postcss.config.js` 这些**独立文件不再被 Nuxt 读取**，必须写进 `nuxt.config` 的对应键（347，错误码 B5004）。所以"我把 `@nuxt/postcss8` 配进 postcss.config.js 了却不生效"这类问题，第一站是看文件放对了没。

## vite 节的一条硬限制

官方在示例后面直接写了这句：

> Please note that not all vite options are supported in Nuxt.

也就是 `vite` 不是 Vite 配置的透传全集。它下面文档具名展开的有：

| 键 | 类型 / 默认 | 说明 |
| --- | --- | --- |
| `$client` | object | 合并进 Vite 的**客户端（浏览器）**构建配置 |
| `$server` | object | 合并进 Vite 的**服务端**构建配置 |
| `build.assetsDir` | string，`"_nuxt/"` | 构建产物资源目录 |
| `build.emptyOutDir` | boolean，`false` | — |
| `cacheDir` | string，`"<rootDir>/node_modules/.cache/vite"` | — |
| `clearScreen` | boolean，`true` | — |
| `define` | object | 默认含 `__VUE_PROD_HYDRATION_MISMATCH_DETAILS__: false`、`process.dev: false`、`import.meta.dev: false`、`process.test: false`、`import.meta.test: false` |
| `esbuild` | object | 默认 `target: "esnext"`、`jsxFactory: "h"`、`jsxFragment: "Fragment"`、`tsconfigRaw: {}` |
| `mode` | string，`"production"` | — |
| `optimizeDeps.esbuildOptions` | object | 默认同上面那组 esbuild 值 |

`define` 那默认值值得停一下：`import.meta.dev`/`import.meta.test` 出现在这里，正对应 426 说的"运行时属性是静态注入的"——注入就是在这一层做的。而 `__VUE_PROD_HYDRATION_MISMATCH_DETAILS__` 是 373 那类水合警告的详细开关（默认关）。

## 环境分流：`$client` 与 `$server`

官方对这一点的表述很实用：

> Top-level `vite` options are shared across both client and server environments. Use `$client` and `$server` to provide environment-specific configuration that will be merged into their respective builds.

配法：

```ts
export default defineNuxtConfig({
  vite: {
    $client: {
      build: {
        rollupOptions: {
          output: {
            manualChunks: {
              analytics: ['analytics-package'],
            },
          },
        },
      },
    },
    $server: {
      build: {
        sourcemap: 'inline',
      },
    },
  },
})
```

规则记两句：**顶层 vite 选项两端共享**；只想影响一端就放进对应的 `$client`/`$server`，它会 merge 进那一份构建。样式相关的落点是：全局 CSS 与 chunk 划分是**客户端**的事，`$client`；服务端渲出时的 sourcemap 之类是 `$server`。

## postcss 节保持独立

419 讲过内容，这里只强调结构：`postcss` 是**顶层节**，两个键 `order`（"A strategy for ordering PostCSS plugins."，function）与 `plugins`（键为包名、值为选项对象）。它不嵌在 `vite.css` 下面——Vite 的 `css` 命名空间管的是**预处理器**（scss/less/stylus 的 `preprocessorOptions`），PostCSS 插件链走 Nuxt 自己的 `postcss` 节，因为 **Nuxt 自带 postcss 并已预配四个插件**。

两条链的顺序是：预处理器先把 `.scss` 编成 CSS，再由 PostCSS 插件加工。所以 `postcss.order` 与 `vite.css.preprocessorOptions` 是两个阶段，不是同一处的两种写法。

## 一切以"用 Vite"为前提

换构建器就会断掉一批配置，官方在两处点到：

- 预处理器那页："Nuxt uses Vite by default. If you wish to use webpack instead, refer to each preprocessor loader documentation."（418）
- `features.inlineStyles`："This is currently available **only when using Vite**."（427）

builders 那页解释了为什么：builder 负责把虚拟应用打包成 JS/CSS，官方三个构建器（Vite 默认、webpack、Rspack）用 `builder` 选项选。它同时给了模块作者的**跨构建器**出路：

- `addVitePlugin` / `addWebpackPlugin` 注册构建器专属插件；
- `addBuildPlugin` 注册 **unplugin** 工厂，"so a single plugin works across every builder"；
- `extendViteConfig` / `extendWebpackConfig` 改已解析的打包器配置。

官方构建器还会发自己的钩子（`vite:extendConfig`、`vite:serverCreated`、`webpack:config`）。所以：写在自己项目里，选一个构建器配它就行；**写成要发布的模块**，优先考虑 `addBuildPlugin`，别把插件挂在单一构建器的键上。

另外记一句 builders 页的定位，避免把样式产物错怪给构建器：**可部署的服务端是 Nitro 产的**，builder 只是把产物按契约交给 Nitro 打进 `.output`（331/088）。

## 自查

- `vite` 与 `postcss` 是 `nuxt.config` 的**平级两节**；独立的 `vite.config.ts`/`postcss.config.js` 不被读取（347）。
- `vite` 是直传 Vite，但官方明写**并非所有 vite 选项都被支持**。
- 顶层 vite 选项两端共享；只作用一端用 `$client` / `$server`（merge 进各自构建）。
- `vite.css.*` 管预处理器；PostCSS 插件链与插件顺序在顶层 `postcss.plugins` / `postcss.order`，两者是不同阶段。
- 换 webpack/Rspack 会让这些失效（预处理器要看各 loader 文档、`inlineStyles` 仅 Vite）；模块要跨构建器用 `addBuildPlugin`（unplugin）。
- `.output` 由 Nitro 产出，builder 只按契约交产物。

## 下一篇

下一篇 430 收这一章的排错：样式路径 404、资源取不到、首屏闪一下，各自对应前面哪一条事实，按什么顺序查。
