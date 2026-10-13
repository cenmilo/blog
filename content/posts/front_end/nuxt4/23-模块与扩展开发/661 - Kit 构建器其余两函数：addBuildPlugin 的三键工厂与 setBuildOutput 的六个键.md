---
url: /posts/fe05bc55f994835f6fdec40f722955a6/
title: 661 - Kit 构建器其余两函数：addBuildPlugin 的三键工厂与 setBuildOutput 的六个键
date: 2027-05-20T10:00:00+08:00
updated: 2027-05-20T10:00:00+08:00
author: cmdragon

summary:
  摘要：这两个函数都不面向普通模块。`addBuildPlugin` 是 **`addVitePlugin` 与 `addWebpackPlugin` 的构建器无关版本——如果两种配置都存在，它会把插件同时加进去**，它的类型里那个工厂接口有 **`vite`、`webpack`、`rspack` 三个键**，而**参数说明的文字只提到 `vite` 与 `webpack` 两个**。`setBuildOutput` 则定义的是**构建器与 Nitro 服务端运行时之间的那份契约**：构建产物里每个键都映射到一个 `nuxt/*` 子路径导入，服务端运行时在构建期解析它；**键一共六个——`serverEntry`、`clientManifest`、`clientPrecomputed`、`ssrStyles`、`entryChunkName`、`entryIds`**，页面明说它**通常是在构建器内部、或从参与构建的模块里被调用**。这一节**连一个 Usage 小节都没有**，`addBuildPlugin` 也是**只有类型与参数、没有示例**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 构建器接口
  - 构建产物
  - 自定义builder
  - NuxtKit
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上一篇那四个函数是给用 Vite 或 webpack 的模块用的。剩下这两个不同：**一个跨三种构建器，一个根本不属于模块的日常**。

## 两个的定位

| 函数 | 页面给的定位 |
| --- | --- |
| `addBuildPlugin` | `addVitePlugin` 与 `addWebpackPlugin` 的**构建器无关版本**；**如果两种配置都存在，就把插件同时加进 Vite 与 webpack** |
| `setBuildOutput` | **为给定键设置一个构建产物提供者**；**通常在构建器内部调用，或从参与构建的模块里调用** |

第二行的那句"通常在构建器内部调用"已经把范围划清了：**普通模块不会用到它**，页面上它存在的意义是告诉你这份契约长什么样。

## `addBuildPlugin`：工厂里有三个键

```ts
function addBuildPlugin (pluginFactory: AddBuildPluginFactory, options?: ExtendConfigOptions): void
```

接口本身：

```ts
interface AddBuildPluginFactory {
  vite?: () => VitePlugin | VitePlugin[]
  webpack?: () => WebpackPluginInstance | WebpackPluginInstance[]
  rspack?: () => RspackPluginInstance | RspackPluginInstance[]
}
```

三处要点：

**每个键都是一个函数而不是插件实例**——与上一篇那两个 `add*Plugin` 不同（那两个可以直接传实例）。**三个键都是可选的**，所以只写 `vite` 也成立。

**类型里有 `rspack`，而参数说明的文字漏了它**：原文只写"**返回一个带有 `vite` 和/或 `webpack` 属性的工厂函数**，这些属性必须是返回 Vite 插件实例/数组、和/或 webpack 插件实例/数组的函数"——**没有提到 `rspack`**，但它就在同一个接口里，类型来自 `@rspack/core` 的 `RspackPluginInstance`。**这是文档自身的缺口，按类型写而不是按那句话写**。

这也解释了一件前面的事：兼容性那一节的 `bridge` 约束可以点名 `rspack` 这个 builder，**说明 rspack 在这套体系里是被承认的构建器之一**，但**本系列所读的 4.x 页面里，除了这一处类型与 `setBuildOutput` 的那句列举，几乎没有讲到 rspack 怎么用**。

选项表与那两个 `add*Plugin` 完全同形（五项：`dev`、`build`、`server`、`client`、`prepend`），**且没有任何一项标弃用**。

**这一节没有 Usage、也没有 Example**：整节只有类型与参数两段。**要写跨构建器的插件，只能照类型自己拼**，形态大致是：

```ts
addBuildPlugin({
  vite: () => myVitePlugin(),
  webpack: () => new MyWebpackPlugin(),
})
```

（**上面这段是按接口类型写的，不是文档原文**。）

## `setBuildOutput`：构建器与 Nitro 之间的契约

这一节的定性句是整篇里最值得读的一段：

> Build outputs are the contract between builders (Vite, webpack, Rspack, or a custom builder) and the Nitro server runtime: each key maps to a `nuxt/*` subpath import the server runtime resolves at build time.

> This is typically called from inside a builder, or from a module that participates in the build.

拆开是四件事：**参与方是 builder 与 Nitro 服务端运行时**；**列举的 builder 有 Vite、webpack、Rspack 或自定义 builder**；**每个键映射到一个 `nuxt/*` 子路径导入**；**服务端运行时在构建期解析这些导入**，并且**provider 是被"懒读取"的**（下一段参数说明里写的是 "read lazily when the server build resolves the corresponding `nuxt/*` import"）。

```ts
function setBuildOutput<K extends keyof NuxtBuildOutputs> (key: K, provider: NuxtBuildOutputs[K]): void
```

**`key` 的六个取值**，页面逐项列出：

| 键 | 用途（按示例推断） |
| --- | --- |
| `serverEntry` | 服务端入口 |
| `clientManifest` | 客户端清单 |
| `clientPrecomputed` | 客户端预编译产物 |
| `ssrStyles` | SSR 样式 |
| `entryChunkName` | 入口 chunk 名 |
| `entryIds` | 入口 id |

**这行右侧的"用途"是从示例与键名推出来的，文档没有逐项解释**——`clientPrecomputed`、`entryChunkName`、`entryIds` 三个键**在那一节没有任何示例**，只有名字。

`provider` 的说明是一句很有用的话：**该键对应的值，一个（可能是 async 的）函数，返回作为字符串的模块体**。

三个示例覆盖了两个键：

```ts
import { setBuildOutput } from '@nuxt/kit'

// Re-export the built SSR entry by absolute specifier.
setBuildOutput('serverEntry', () => `export { default } from ${JSON.stringify(serverEntryURL)}`)

// Provide the serialized client manifest.
setBuildOutput('clientManifest', () => `export default ${serializedManifest}`)

// Re-export the emitted per-component styles map for `nuxt/styles`.
setBuildOutput('ssrStyles', () => `export { default } from ${JSON.stringify(pathToFileURL(resolve(serverDir, 'styles.mjs')).href)}`)
```

三条共同点：**provider 返回的是"一段源代码文本"**，而且**两条用了"重新导出"这个模式**（生成 `export { default } from <绝对路径>`），**一条直接把已经序列化好的清单字符串内联进去**。第三行还顺带交代了键与导入的对应关系——`ssrStyles` 供的是 `nuxt/styles` 这个子路径。

**`JSON.stringify` 出现在两处**：把路径变成字符串字面量，避免拼出来的源码里出现未加引号的值。

页面在这节末尾挂了一条 read-more，指向自定义构建器那一章的"构建产物契约"小节——**六个键各自的完整语义在那里，那一节只给了形状**。

## 什么情况下真的会用到它们

| 你在做的事 | 需要哪个 |
| --- | --- |
| 模块要同时给 Vite 与 webpack 注入同一个插件 | `addBuildPlugin` |
| 只针对一个构建器 | 上一篇的 `addVitePlugin`／`addWebpackPlugin` |
| 写一个自定义 builder，或参与构建产出 | `setBuildOutput` |
| 只想改配置里的某个键 | 都不是——用插件的 `config`／`configEnvironment` 钩子 |

`addBuildPlugin` 的实际收益很窄：**它免去 `if (builder === ...)` 的判断**，代价是**必须为每种构建器各准备一个工厂函数**。文档没写"两个 builder 都不存在时会怎样"（那句只说"如果两种配置都存在就都加"），**这一点属于文档未说明**。

## 一个衔接

`setBuildOutput` 的存在解释了为什么"构建器"这件事在文档里被反复单独对待：**Nuxt 把 builder 与 Nitro 之间的关系收成一份用 `nuxt/*` 表达的契约**，所以插件、路由规则、服务端 handler 这些模块动作才能在不同 builder 下保持同一套 API。服务端兼容性那一章讲 `nuxt/server` 时说的"其他 server builder 也能跑"，根子就在这里。

下一篇讲图层目录那一个函数：`getLayerDirectories` 与它的十个目录字段。

## 下一篇

下一篇 662：getLayerDirectories：十个目录字段、优先级顺序与那两条 note

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/builder
