---
url: /posts/7a0a0f73ff403726f9b6d1f18b240a2a/
title: Kit 构建器四函数：extendViteConfig 已弃用与两个加插件的函数
date: 2027-05-20T09:00:00+08:00
updated: 2027-05-20T09:00:00+08:00
author: cmdragon

summary:
  摘要：构建器这一页六个函数里，**Vite 那一侧正在换写法**：`extendViteConfig` 被一条 `::warning` 标为**已弃用**，官方建议改用带 `config` 钩子的 Vite 插件，**环境相关的配置则用 `applyToEnvironment` 钩子**。同一页还有第二条警告点明顺序：**`config` 钩子在 `applyToEnvironment` 之前运行，改的是全局配置；要按环境改就用 `configEnvironment`**。选项表里 **`server` 与 `client` 两项在 Vite 侧被标 "Deprecated in Nuxt 5+"，而 webpack 那两节的同名选项没有任何弃用标记**。`addVitePlugin` 与 `addWebpackPlugin` 都接受"实例、数组，或一个可返回实例/数组的函数"，**函数还可以是 async 或返回 Promise，用来懒加载插件**。

categories:
  - 前端开发

tags:
  - nuxt4
  - Vite插件
  - webpack配置
  - 构建器
  - NuxtKit
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模块要动打包配置，以前就是 `extendViteConfig` 一行。这一页现在的写法是：**那一行本身已经被标弃用，替代方案是写一个 Vite 插件**。这一篇把四函数的分工与两条弃用线说清。

## 页面给的两种 builder

开场句：**Nuxt 有基于 Vite 与 webpack 的构建器；可以用 `extendViteConfig` 与 `extendWebpackConfig` 扩展传给各自的配置；也可以通过 `addVitePlugin`、`addWebpackPlugin`、`addBuildPlugin` 添加额外的插件**。

注意这句点了五个名字，**页上的 `##` 条目有六个**——`setBuildOutput` 不在开场句里（它是给自定义 builder 用的，另篇讲）。

四个条目的分派：

| 函数 | 做什么 | 状态 |
| --- | --- | --- |
| `extendViteConfig` | 扩展 Vite 配置 | **已弃用** |
| `extendWebpackConfig` | 扩展 webpack 配置 | 未标弃用 |
| `addVitePlugin` | 往配置里追加 Vite 插件 | 现役，**且是 `extendViteConfig` 的替代** |
| `addWebpackPlugin` | 往配置里追加 webpack 插件 | 未标弃用 |

## `extendViteConfig`：弃用与替代写法

```ts
function extendViteConfig (callback: ((config: ViteConfig) => void), options?: ExtendViteConfigOptions): void
```

定性：**扩展 Vite 配置；当同时作用于客户端与服务端构建时，回调函数会被调用多次**。

紧接着的警告是本页的方向盘：

> This hook is now deprecated, and we recommend using a Vite plugin instead with a `config` hook, or — for environment-specific configuration — the `applyToEnvironment` hook.

旧写法是这样的（页面仍然保留着它）：

```ts
extendViteConfig((config) => {
  config.optimizeDeps ||= {}
  config.optimizeDeps.include ||= []
  config.optimizeDeps.include.push('cross-fetch')
})
```

替代写法是**用 `addVitePlugin()` 交一个插件**，页面把"全局"和"按环境"两种各给一份：

```ts
// For global configuration (affects all environments)
addVitePlugin(() => ({
  name: 'my-global-plugin',
  config (config) {
    // This runs before environment setup
    config.optimizeDeps ||= {}
    config.optimizeDeps.include ||= []
    config.optimizeDeps.include.push('cross-fetch')
  },
}))

// For environment-specific configuration
addVitePlugin(() => ({
  name: 'my-client-plugin',
  applyToEnvironment (environment) {
    return environment.name === 'client'
  },
  configEnvironment (name, config) {
    // This only affects the client environment
    config.optimizeDeps ||= {}
    config.optimizeDeps.include ||= []
    config.optimizeDeps.include.push('client-only-package')
  },
}))
```

顺序问题由第二条警告回答，**这句是新手最容易踩的**：

> **Important:** The `config` hook runs before `applyToEnvironment` and modifies the global configuration. Use `configEnvironment` for environment-specific configuration changes.

也就是说：**在 `config` 里写的东西对所有环境生效；只想改客户端（或只想改服务端）就得放进 `configEnvironment`**，并用 `applyToEnvironment` 决定这个插件适用于哪个环境。

选项表五项：

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `dev` | `boolean` | `false` | 为 `true` 时在开发模式构建时调用回调 |
| `build` | `boolean` | `false` | 为 `true` 时在生产模式构建时调用回调 |
| `server` | `boolean` | `false` | 为 `true` 时在构建服务端 bundle 时调用回调。**Nuxt 5+ 起弃用**，改用带 `applyToEnvironment()` 的 `addVitePlugin()` |
| `client` | `boolean` | `false` | 为 `true` 时构建客户端 bundle 时调用回调。**Nuxt 5+ 起弃用**，同上 |
| `prepend` | `boolean` | `false` | 为 `true` 时回调用 `unshift()` 而不是 `push()` 放进数组 |

**`server` 与 `client` 这两项被单独标了弃用**，`dev`/`build`/`prepend` 没有——所以"按端过滤"这个能力整体在退场，而"按 dev/build 过滤"仍在。

## `extendWebpackConfig`：同一套选项，没有弃用

```ts
function extendWebpackConfig (callback: ((config: WebpackConfig) => void), options?: ExtendWebpackConfigOptions): void
```

定性与 Vite 那句**逐字同构**（"扩展 webpack 配置；同时作用于客户端与服务端构建时回调会被调用多次"），示例是加一条 loader：

```ts
extendWebpackConfig((config) => {
  config.module!.rules!.push({
    test: /\.txt$/,
    use: 'raw-loader',
  })
})
```

它的选项表也是五项，**但 `server` 与 `client` 后面没有那句 "Deprecated in Nuxt 5+"**：

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `dev` | `boolean` | `false` | 开发模式构建时调用回调 |
| `build` | `boolean` | `false` | 生产模式构建时调用回调 |
| `server` | `boolean` | `false` | 构建服务端 bundle 时调用回调 |
| `client` | `boolean` | `false` | 构建客户端 bundle 时调用回调 |
| `prepend` | `boolean` | `false` | 用 `unshift()` 而不是 `push()` |

**这一处不对称很重要**：同样的 `{ server: false }` 在 webpack 侧是正常用法（下一个示例就这么用），在 Vite 侧则是被弃用的写法。**别把 Vite 的迁移规则套到 webpack 上**。

## `addVitePlugin`：三种入参与一个懒加载技巧

```ts
function addVitePlugin (pluginOrGetter: VitePlugin | VitePlugin[] | (() => VitePlugin | VitePlugin[]), options?: ExtendViteConfigOptions): void
```

**`pluginOrGetter` 可以是插件实例、实例数组，或一个返回实例/数组的函数；这个函数还可以是 async 或返回 Promise，用于懒加载插件**：

```ts
// Lazy load the plugin - only imported when the build actually runs
addVitePlugin(() => import('my-vite-plugin').then(r => r.default()))
```

**这句注释就是它的目的：插件等到真正构建时才被 import**，模块冷启动不必为它付代价。

带配置的完整示例（一个把 SVG 当组件用的模块形状）：

```ts
import { addVitePlugin, defineNuxtModule } from '@nuxt/kit'
import { svg4VuePlugin } from 'vite-plugin-svg4vue'

export default defineNuxtModule({
  meta: {
    name: 'nuxt-svg-icons',
    configKey: 'nuxtSvgIcons',
  },
  defaults: {
    svg4vue: {
      assetsDirName: 'assets/icons',
    },
  },
  setup (options) {
    addVitePlugin(svg4VuePlugin(options.svg4vue))

    // or, to add a vite plugin to only one environment
    addVitePlugin(() => ({
      name: 'my-client-plugin',
      applyToEnvironment (environment) {
        return environment.name === 'client'
      },
      // ... rest of your client-only plugin
    }))
  },
})
```

它的选项表**与 `extendViteConfig` 那张完全一样**（同样五项、同样 `server`/`client` 标弃用）。

这里还有一条与上面呼应的警告：

> In Nuxt 5+, plugins registered with `server: false` or `client: false` options will not have their `config` or `configResolved` hooks called. Instead, use the `applyToEnvironment()` method instead for environment-specific plugins.

**后果写得很具体：用旧的两个选项注册的插件，它的 `config` 与 `configResolved` 钩子根本不会被调用**——这类失效不会报错，只会让插件"看起来没生效"。

## `addWebpackPlugin`：与它同形，例子更有信息量

```ts
function addWebpackPlugin (pluginOrGetter: WebpackPluginInstance | WebpackPluginInstance[] | (() => WebpackPluginInstance | WebpackPluginInstance[]), options?: ExtendWebpackConfigOptions): void
```

描述与 Vite 那段**逐字对应**（实例/数组/可返回的函数、可 async、可懒加载），选项五项且**无弃用标记**。示例是 ESLint 的 webpack 插件，它同时示范了三个技巧：

```ts
import EslintWebpackPlugin from 'eslint-webpack-plugin'
import { addWebpackPlugin, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  meta: {
    name: 'nuxt-eslint',
    configKey: 'eslint',
  },
  defaults: nuxt => ({
    include: [`${nuxt.options.srcDir}/**/*.{js,jsx,ts,tsx,vue}`],
    lintOnStart: true,
  }),
  setup (options, nuxt) {
    const webpackOptions = {
      ...options,
      context: nuxt.options.srcDir,
      files: options.include,
      lintDirtyModulesOnly: !options.lintOnStart,
    }
    addWebpackPlugin(new EslintWebpackPlugin(webpackOptions), { server: false })
  },
})
```

三处都能借用：**`defaults` 写成 `nuxt => ({...})` 这种函数形态**（默认值要引用当前配置时只能这样）、**`include` 用 `nuxt.options.srcDir` 拼 glob**、**`{ server: false }` 把插件只挂在客户端**。

`lintDirtyModulesOnly: !options.lintOnStart` 这一行是把模块选项翻译成插件选项的常见做法——**用户的布尔值取反后喂给第三方插件**。

两个 tip 分别给了 Vite 插件文档＋awesome-vite 插件列表、webpack 插件文档＋awesome-webpack 集合，**找现成插件时从这两处入手**。

## 该写哪一种

| 你要做的事 | 4.x 现在的写法 |
| --- | --- |
| 给 Vite 加一个插件 | `addVitePlugin(pluginOrGetter)`，**需要懒加载就传函数** |
| 改 Vite 配置里的某个键 | **写一个带 `config` 钩子的 Vite 插件**（`extendViteConfig` 已弃用） |
| 只改某个环境的 Vite 配置 | 插件里用 `applyToEnvironment` ＋ `configEnvironment` |
| 改 webpack 配置或加 webpack 插件 | `extendWebpackConfig`／`addWebpackPlugin`，`server`/`client` 仍可用 |
| 一份代码同时给两个 builder | 用 `addBuildPlugin`（下一篇） |

这一页没写的：`ExtendViteConfigOptions` 与 `ExtendWebpackConfigOptions` 两个类型各自的完整字段（表里只给了五项）、`config` 与 `configEnvironment` 的调用次序细节（只说了 `config` 更早）、以及 Nuxt 5 环境 API 本身的行为（在实验特性那章）。

下一篇讲这一页剩下两个函数：跨 builder 的 `addBuildPlugin` 与给自定义 builder 用的 `setBuildOutput`。

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/builder
