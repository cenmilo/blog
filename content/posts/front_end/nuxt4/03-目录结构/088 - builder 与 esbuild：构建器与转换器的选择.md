---
url: /posts/f1153dae6e6b8572a736239ca6c5f2a9/
title: 088 - builder 与 esbuild：构建器与转换器的选择
date: 2026-11-10T10:00:00+08:00
updated: 2026-11-10T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：顶层 builder 管的是"打包应用 Vue 那一侧"用哪个构建器，类型是 'vite' | 'webpack' | 'rspack' | string | { bundle: (nuxt: Nuxt) => Promise<void> }、默认 "@nuxt/vite-builder"；Nuxt 自带三个官方构建器（默认的 Vite，以及 webpack 和 Rspack），但官方同时写明：如果用 webpack 或 rspack，必须确保 @nuxt/webpack-builder 或 @nuxt/rspack-builder 被显式安装进项目。构建器的接口是 NuxtBuilder，唯一必需的方法是 bundle，另有一个可选的 setupWatcher（只在用户通过 experimental.watcher: 'builder' 选入时，Nuxt 才调用它而不启动自己的开发文件监视器）。Nuxt 在 nuxt build 与 nuxt dev 期间各调用一次 bundle(nuxt)，时机是虚拟应用生成完、build:before 钩子触发之后，并且会把你的 bundle 包一层，让抛出的错误自动触发 build:error。要划清的界线是：可部署的服务端本身是由 Nitro 通过 @nuxt/nitro-server 产出的，不是构建器产出的。而 esbuild 这一节既不是开关也不是构建器，它只有一个 options，官方说明是"配置 Nuxt 内部使用、并传给其他构建器（如 Vite 或 webpack）的共享 esbuild 选项"，四格默认值分别是 jsxFactory "h"、jsxFragment "Fragment"、target "esnext"、tsconfigRaw 为 object。另有一条错误码值得记住：Nuxt 发现在 nuxt.config 旁边有独立的 vite.config 或 webpack.config 文件时会报 NUXT_B5004，因为 Nuxt 内部管理打包器并忽略这些文件，正确做法是把配置移进 nuxt.config 的 vite / webpack / nitro / postcss 对应键里再删掉外部文件。

categories:
  - 前端开发

tags:
  - nuxt4
  - builder
  - esbuild
  - vite
  - 构建

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`nuxt.config` 里有三个都叫构建的东西，混在一起的后果是配置写了不生效。这一篇把它们分开：打包器选哪个、服务端由谁产出、esbuild 那一节到底管什么。

先把三条线画清楚。

| 名称 | 官方定义 | 位置 |
| --- | --- | --- |
| `builder` | 打包你的应用 Vue 那一侧 所用的构建器 | 顶层配置项 |
| `server.builder` | 打包你的应用服务端那一侧 的构建器 | `server` 节里唯一的子项 |
| Nitro | 可部署的服务端本身是由 Nitro 通过 `@nuxt/nitro-server` 产出的，不是 由构建器产出的。构建器通过一个带类型的契约把自己的输出交给 Nitro，由 Nitro 打包成最终的 `.output` | 官方构建器指南 |

也就是说，`builder` 换掉不会改变你的服务端运行时从哪来。这两件事分别归两个键管，写错位置是白写。

`builder`：三个官方构建器和一条硬性前提。

官方定义：The builder to use for bundling the Vue part of your application。

官方补充：Nuxt 为客户端应用支持多种构建器。默认使用 Vite，但可以切换到 webpack、Rspack，甚至提供一个自定义的构建器实现。

类型：`'vite' | 'webpack' | 'rspack' | string | { bundle: (nuxt: Nuxt) => Promise<void> }`。默认：`"@nuxt/vite-builder"`。

官方给的写法是注释掉前两个、启用第三个。

```ts
export default defineNuxtConfig({
  // default - uses @nuxt/vite-builder
  // builder: 'vite',

  // uses @nuxt/webpack-builder
  // builder: 'webpack',

  // uses @nuxt/rspack-builder
  builder: 'rspack',
})
```

这里有个必须单独记住的前提：官方原文是，如果你使用 `webpack` 或 `rspack`，你需要确保 `@nuxt/webpack-builder` 或 `@nuxt/rspack-builder` 被显式安装到你的项目中。

换个字符串就完事，是不成立的。装完依赖再改配置，顺序反过来报的是找不到模块。

自定义构建器：两种写法。

写法一，内联对象。官方说明你可以通过传入一个带 `bundle` 函数的对象来提供自定义构建器。

```ts
export default defineNuxtConfig({
  builder: {
    async bundle (nuxt) {
      const entry = await resolvePath(resolve(nuxt.options.appDir, 'entry'))

      // Build client and server bundles
      await buildClient(nuxt, entry)
      if (nuxt.options.ssr) {
        await buildServer(nuxt, entry)
      }

      // ... it's a bit more complicated than that, of course!
    },
  },
})
```

那句官方注释要读进去：真正的构建器远不止这几行。

写法二，独立包。官方说明：要创建一个作为独立包的自定义构建器，它应当导出一个 `bundle` 函数，然后你就可以在 `nuxt.config.ts` 里指定这个包名。

```ts
export default defineNuxtConfig({
  builder: 'my-custom-builder',
})
```

接口本身长这样，唯一必需的方法是 `bundle`。

```ts
import type { Nuxt } from '@nuxt/schema'

export interface NuxtBuilder {
  bundle: (nuxt: Nuxt) => Promise<void>
  /**
   * Optional. When the user opts in via `experimental.watcher: 'builder'`,
   * Nuxt calls this instead of starting its own dev file watcher, letting
   * the builder reuse its own watcher. The builder should register a
   * `nuxt.hook('close', ...)` to clean up.
   */
  setupWatcher?: (nuxt: Nuxt) => Promise<void> | void
}
```

`setupWatcher` 的条件写得很具体：只有当用户通过 `experimental.watcher: 'builder'` 选入时，Nuxt 才调用它、而不是启动自己的开发文件监视器，好让构建器复用自己的监视器，并且构建器应当注册一个 `nuxt.hook('close', ...)` 来做清理。

还有两条容易被忽略的官方注记。

一，Nuxt 在 `nuxt build` 与 `nuxt dev` 期间各调用一次 `bundle(nuxt)`，时机是虚拟应用已经生成、`build:before` 钩子已经触发之后。Nuxt 会把你的 `bundle` 包一层，使得其中抛出的错误自动触发 `build:error` 钩子。

二，只有三个官方构建器标识，`@nuxt/vite-builder`、`@nuxt/webpack-builder`、`@nuxt/rspack-builder`，会在 Nuxt 其他位置被按名字识别，用来做构建器特定的行为。自定义构建器仍然通过上面描述的通用契约工作。

构建器到底做哪些事。官方把职责列成了三条。

一，打包客户端 构建（浏览器包），以及在启用 SSR 时打包服务端 构建（SSR 应用入口）。

二，产出服务端运行时渲染与水合所需的产物：客户端清单、按组件的样式映射等等。

三，在开发中，暴露一个开发服务器，并在构建变化时触发重载。

官方还给了实现的分支范式：`bundle` 通常按 `nuxt.options.dev` 分支。生产环境把客户端和（若 `nuxt.options.ssr`）服务端构建跑完，把产物写入 `nuxt.options.buildDir` 并注册为构建输出。开发环境则设置开发服务器、启动一个 watch 构建、赋值 `nuxt.server`，并持续运行。

至于要不要自己写一个，官方直接劝退：编写构建器是一个高级话题。大多数应用永远不需要自定义构建器，如果你只是想影响包的内容，一个注册了打包器插件的模块通常是正确的工具。

独立配置文件是不被支持的。这是很多人从别的项目搬过来的习惯：在项目根放一个 `vite.config.ts`。官方错误码 NUXT_B5004 就是给这件事的。

> Nuxt found a standalone `vite.config` or `webpack.config` file next to your `nuxt.config`. Nuxt manages the bundler internally and ignores these files, so they are usually leftovers from a migration or a copied non-Nuxt project.

即 Nuxt 在 `nuxt.config` 旁边发现独立的 `vite.config` 或 `webpack.config` 时会报这个错，Nuxt 内部管理打包器并且会忽略这些文件，它们通常是一次迁移或复制来的非 Nuxt 项目的遗留物。官方的解决方式是移进 `nuxt.config` 的对应键，然后删掉外部文件。

| 外部文件 | 应当改用的键 |
| --- | --- |
| `vite.config` | `vite` |
| `webpack.config` | `webpack` |
| `nitro.config` | `nitro` |
| `postcss.config` | `postcss` |

```ts
export default defineNuxtConfig({
  vite: {
    // your Vite config here
  },
})
```

注意这条与 `builder` 的关系：无论选哪个构建器，它的配置都写在 `nuxt.config` 里，不存在换构建器就能用独立配置文件这条路。

`esbuild`：不是开关，是共享选项。`esbuild` 这一节只有一个子项 `options`，官方的说明是配置 Nuxt 内部使用的、并传递给其他构建器（例如 Vite 或 webpack）的共享 esbuild 选项。

| 键 | 类型 | 默认 |
| --- | --- | --- |
| `esbuild.options.jsxFactory` | `string` | `"h"` |
| `esbuild.options.jsxFragment` | `string` | `"Fragment"` |
| `esbuild.options.target` | `string` | `"esnext"` |
| `esbuild.options.tsconfigRaw` | `object` | 无 |

这四项解释了为什么换了构建器之后它们仍然有效：esbuild 在这里是被共享的转换层，不是构建器。你选 `vite`、`webpack` 还是 `rspack`，都不改变 Nuxt 内部用这几个 esbuild 选项这一事实，反过来，改 `builder` 也不会让这几格失效。

其中 `target` 默认 `"esnext"` 是最值得注意的一格：它决定了 esbuild 转换时不为你降级语法，面向旧浏览器的兜底不在这一格，而在构建器那一侧。

一张决策表。

| 你想做的事 | 该动哪个 |
| --- | --- |
| 换成 webpack 或 Rspack | `builder`，并且先显式安装 `@nuxt/webpack-builder` / `@nuxt/rspack-builder` |
| 调整 JSX 工厂、碎片名、转换 target | `esbuild.options.*` |
| 改打包器的行为（插件、别名、构建选项） | `nuxt.config` 里的 `vite` / `webpack` 键，不要 放独立配置文件 |
| 换服务端那一侧的构建方式 | `server.builder`（那仍是实验性的） |
| 只是想让某个模块影响包内容 | 写模块注册打包器插件，而不是写构建器 |

## 自查

- [ ] `builder` 与 `server.builder` 分别打包应用的哪一部分？默认值各是什么？
- [ ] 把 `builder` 设为 `'webpack'` 之前必须先做什么？官方原话是哪一句？
- [ ] `NuxtBuilder` 接口里唯一必需的方法是什么？`setupWatcher` 在什么条件下才会被调用？
- [ ] 可部署的服务端是由构建器产出的吗？官方怎么描述构建器与 Nitro 的交接？
- [ ] 根目录放一个 `vite.config.ts` 会发生什么？官方给出的四个移进 `nuxt.config` 的对应键是哪些？

## 下一篇

089 是目录结构这一部分的收束：把前面所有关于目录、基准与配置项的结论压成一页速查表。
