---
url: /posts/7c84a14503d1d80796582ed39fe97c92/
title: 680 - 构建器内幕：三个官方 builder、bundle 契约与六个 nuxt 子路径
date: 2027-05-26T11:00:00+08:00
updated: 2027-05-26T11:00:00+08:00
author: cmdragon

summary:
  摘要：这一节把"打包"从 Nuxt 里拆出来讲：**builder 是负责打包你应用的那一部分，Nuxt 自带三个官方 builder——Vite（默认）、webpack、Rspack**，用 `builder` 选项选，也可以自己提供一个。**分工是一句"要构建什么"归 Nuxt 核心与模块，"怎么构建"归 builder**：核心与模块生成虚拟应用（入口、路由表、插件、组件注册表以及 `#build` 下的虚拟文件系统），builder 把它变成真实的 JS 与 CSS。两条容易记错的边界：**可部署的服务端本身由 Nitro 通过 `@nuxt/nitro-server` 产出，不是 builder**；**builder 与其余部分之间靠一份 typed 契约传递产物**。契约就是那份 **六个键的 `NuxtBuildOutputs`**，每个键映射到一个 `nuxt/*` 子路径，**服务端运行时通过 `nuxt/entry`、`nuxt/manifest` 这些稳定子路径来 import 产物**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 构建器
  - 构建产物契约
  - 自定义builder
  - Nitro
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

为什么换了打包器，服务端那侧不用改？这一节回答的就是这个——它把 Nuxt 的构建拆成"生成虚拟应用"和"把虚拟应用变成产物"两段，并给出两段之间那份契约的全部键。

## 先确认有几个 builder

第一句给的是范围：**builder 是 Nuxt 中负责打包你应用的部分。Nuxt 自带三个官方 builder——Vite（默认）、webpack 与 Rspack——你可以用 `builder` 选项选一个，也可以提供自己的**。

这条信息与本系列前几处出现过的表述并不完全一致：**Kit 的 `buildNuxt` 那一节说"目前调用 @nuxt/vite-builder 或 @nuxt/webpack-builder"**，**而钩子表里只有 vite 与 webpack 两族钩子、没有任何 rspack 行**。这一节是全套文档里唯一把三个 builder 并列写明的地方。

页面还先划了一条门槛（`::note`）：**编写 builder 是一个进阶话题。大多数应用永远不需要自定义 builder；如果你只想影响产物，一个注册了打包器插件的模块通常是正确的工具**。

## 职责边界：谁产出什么

一句总纲：**Nuxt 把"要构建什么"与"怎么构建"分开**。

**"要构建什么"这侧**：Nuxt 核心与你的模块生成虚拟应用——入口、路由表、插件、组件注册表，以及 `#build` 之下虚拟文件系统里的其余部分。

**"怎么构建"这侧**：builder 拿走那个虚拟应用，把它变成真实的 JavaScript 与 CSS 包；开发期它还负责跑一个 dev server 来提供与热重载。

具体负责的三件事：**打包客户端构建（浏览器包）以及启用 SSR 时的服务端构建（SSR 应用入口）**；**产出服务端运行时需要用来渲染与水合的产物（客户端 manifest、按组件的样式映射等）**；**在开发期暴露 dev server 并在构建变化时触发重载**。

然后是最容易记混的一条：**可部署的服务端本身由 Nitro 通过 `@nuxt/nitro-server` 产出，而不是由 builder 产出**。**builder 只是把自己的产物通过一份 typed 契约交给 Nitro，Nitro 再把它们打进最终的 `.output`**。

## 接口：必填只有 `bundle`

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

**唯一必需的方法是 `bundle`**。第二个方法 `setupWatcher` 的注释把三件事说了：**用户通过 `experimental.watcher: 'builder'` 主动选择时 Nuxt 才调它**；**调它就不启动 Nuxt 自己的文件监视器，让 builder 复用自家的**；**builder 应该注册一个 `nuxt.hook('close', ...)` 来做清理**（这与模块侧"打开的东西要在 `close` 里关掉"是同一条要求）。

`builder` 选项接受两种值——**默认导出 `NuxtBuilder` 的模块 specifier，或直接给一个内联对象**：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  // a package that exports `{ bundle }`
  builder: '@nuxt/vite-builder',
})
```

```ts [nuxt.config.ts]
import type { NuxtBuilder } from '@nuxt/schema'

const myBuilder: NuxtBuilder = {
  async bundle (nuxt) {
    // ...
  },
}

export default defineNuxtConfig({
  builder: myBuilder,
})
```

调用时机与错误处理各一句：**Nuxt 在 `nuxt build` 与 `nuxt dev` 期间调用 `bundle(nuxt)` 一次，发生在虚拟应用已生成、`build:before` 已触发之后**；**Nuxt 会包装你的 `bundle`，使任何抛出的错误自动触发 `build:error` 钩子**。

还有一条关于"被认出来"的限制：

> Only the three official builder specifiers (`@nuxt/vite-builder`, `@nuxt/webpack-builder`, `@nuxt/rspack-builder`) are recognised by name for builder-specific behaviour elsewhere in Nuxt. A custom builder still works through the generic contract described below.

**只有这三个 specifier 会被按名字识别**（用于别处的 builder 专属行为）。自定义 builder 依然能工作，**但它走的是通用契约**——也就是说**依赖"Nuxt 知道我在用哪个打包器"的那些行为，自定义 builder 拿不到**。

## 生命周期五步

运行 `nuxt build` 或 `nuxt dev` 时：

1. 创建 `nuxt` 上下文并运行模块，填充 `nuxt.options` 与构建钩子。
2. 生成虚拟应用（模板、路由表、插件）进 `#build` 虚拟文件系统。
3. 触发 `build:before`。
4. 解析 builder 并调用 `builder.bundle(nuxt)`。**这就是你的 builder 运行的位置。**
5. 触发 `build:done`，**在生产模式下关闭 `nuxt` 实例**。

第 5 步那句"生产模式才关闭"是差异点：**`nuxt dev` 下实例不会被这一步关掉**。

`bundle` 的实现按页面说法通常要**分支于 `nuxt.options.dev`**：**生产分支**要把客户端与（若 `nuxt.options.ssr`）服务端构建跑完，产物写进 `nuxt.options.buildDir` 并登记为构建产物；**开发分支**要建 dev server、启动 watch 模式的构建、**给 `nuxt.server` 赋值**，然后持续运行。

其余沟通几乎全部通过钩子完成：**builder 从 `nuxt.options` 读构建配置，并在合适时让模块扩展它自己的打包器配置**。

## 模块影响打包的四条通道

**这是"为什么模块不需要写成 builder"的答案**，页面列了四条并说明 builder 应当尊重它们：

| Kit 助手 | 页面给的作用 |
| --- | --- |
| `addVitePlugin` / `addWebpackPlugin` | 注册**打包器专属**插件 |
| `addBuildPlugin` | 注册一个 **unplugin 工厂**，于是**一个插件在所有 builder 上都可用** |
| `extendViteConfig` / `extendWebpackConfig` | 修改**已解析的打包器配置** |

页面并补了一句：**官方 builder 也会发出它们自己的钩子**（举例 `vite:extendConfig`、`vite:serverCreated`、`webpack:config`），以便模块与 Nitro 参与构建。**自定义 builder 可以发出自己的钩子，但下面那份构建产物契约才是让它与 Nuxt 服务端运行时互操作的东西**。

**注意这份清单里把 `extendViteConfig` 列为正常通道，而 Kit 那一节已把它标为弃用**——两处状态不一致，本系列按各自原文记录，**实际写代码以 Kit 那一节的弃用标记为准**。

## 六个键的契约

**服务端运行时（`@nuxt/nitro-server`）不知道是哪个 builder 产出的应用。它通过稳定的 `nuxt/*` 子路径 import 每一项产物，而当前 builder 用 `nuxt.buildOutputs` 填充这些子路径。这就是构建产物契约。**

契约由 `@nuxt/schema` 里的 `NuxtBuildOutputs` 接口声明：

```ts
export interface NuxtBuildOutputs {
  /** Module body re-exporting the SSR app entry. */
  serverEntry: () => string | Promise<string>
  /** Path to the emitted per-component SSR styles map, or `undefined` when no inline styles are produced. */
  ssrStyles: string | undefined
  /** Serialized client manifest for `vue-bundle-renderer`. */
  clientManifest: () => string | Promise<string>
  /** Serialized precomputed client dependency data for `vue-bundle-renderer`. */
  clientPrecomputed: () => string | Promise<string>
  /** Module body exporting the hashed entry chunk filename for import maps. */
  entryChunkName: () => string | Promise<string>
  /** Module body exporting the entry module IDs used for inline style extraction. */
  entryIds: () => string | Promise<string>
}
```

每个键到一个子路径的对应表：

| 构建产物 | 子路径 | 被当作什么 import |
| --- | --- | --- |
| `serverEntry` | `nuxt/entry` | 传给 `createRenderer` 的 SSR 应用工厂 |
| `clientManifest` | `nuxt/manifest` | `vue-bundle-renderer` 的客户端 manifest |
| `clientPrecomputed` | `nuxt/precomputed` | 预计算好的依赖数据 |
| `ssrStyles` | `nuxt/styles` | 按组件的内联样式映射 |
| `entryChunkName` | `nuxt/entry-chunk` | 带 hash 的入口 chunk 文件名（import map 用） |
| `entryIds` | `nuxt/entry-ids` | 用于样式抽取的入口模块 id |

还有一条让这套机制能"提前可用"的设计：**每个 `nuxt/*` 子路径都在 `nuxt` 包里带一个默认 stub，所以服务端运行时在 builder 跑之前也能通过类型检查并构建**。builder 通过设置对应构建产物来覆盖这个 stub，**它提供的值在构建期替换掉 stub**。

## 两类产物形状，差别有理由

页面把六个键分成两类，并给出**为什么第二行必须是路径**：

**值提供者**（`serverEntry`、`clientManifest`、`clientPrecomputed`、`entryChunkName`、`entryIds`）：**是返回"模块体字符串"的函数**。字符串会被逐字内联进服务端 bundle，所以**它不能依赖文件在磁盘上的位置**：

```ts
setBuildOutput('serverEntry', () => `export { default } from ${JSON.stringify(serverEntryURL)}`)
```

**已产出文件的路径**（只有 `ssrStyles`）：**是一个绝对路径，不是代码**，指向 builder 真正产出的一个模块。运行时的 `nuxt/styles` import 会解析到这个文件，**于是可部署物的打包器会把这份样式映射里的相对兄弟导入按该文件自身位置来解析**。页面接着写了理由：**把它建模成代码字符串会丢掉那个目录上下文并破坏相对导入，所以这一项是路径**：

```ts
setBuildOutput('ssrStyles', resolve(serverOutDir, 'styles.mjs'))
```

并且：**当构建不产出内联样式时，把 `ssrStyles` 留成 `undefined`（它的默认值），运行时会退回到一个空样式映射**。

写入的方式有两种：**`setBuildOutput` 写的是 `nuxt.buildOutputs[key]`**；**在已经持有 `nuxt` 实例的打包器插件里可以直接赋值那个键**，而 `setBuildOutput` 是给"通过 `useNuxt()` 取实例"的代码用的便利封装。

**provider 可以是异步的，并且是惰性读取的**——在**服务端构建解析对应的 `nuxt/*` import 时**才被读。页面点出这条能力的用途：**让 builder 提前注册 provider（例如客户端构建还没结束时），等数据真的存在时再返回最终值**。

## 最小骨架

```ts
import { resolve } from 'node:path'
import { pathToFileURL } from 'node:url'
import { setBuildOutput } from '@nuxt/kit'
import type { NuxtBuilder } from '@nuxt/schema'

export const bundle: NuxtBuilder['bundle'] = async (nuxt) => {
  const serverDir = resolve(nuxt.options.buildDir, 'dist/server')

  // ...run your client and server bundles here, writing artifacts to disk...
  const { serializedClientManifest } = await runBundles(nuxt, serverDir)

  if (nuxt.options.ssr) {
    // Point `nuxt/entry` at the built SSR app entry.
    const serverEntryURL = pathToFileURL(resolve(serverDir, 'server.mjs')).href
    setBuildOutput('serverEntry', () => `export { default } from ${JSON.stringify(serverEntryURL)}`)

    // Provide the client manifest produced by your client build.
    setBuildOutput('clientManifest', () => `export default ${serializedClientManifest}`)

    // If you emit a per-component styles map alongside its CSS chunks:
    setBuildOutput('ssrStyles', resolve(serverDir, 'styles.mjs'))
  }
}
```

配套的 `::note` 是减压的：**你很少需要提供每一个键。默认值已经合理（空 manifest、无内联样式、未定义的入口 chunk），只提供你的构建真正产出的那些。禁用 SSR 时 `serverEntry` 的默认值是一个空操作应用，其余多数产物也不会被使用**。

注意三处写法：**SSR 相关登记全部包在 `if (nuxt.options.ssr)` 里**、**入口 URL 要经过 `pathToFileURL(...).href`**（不是文件系统路径）、**manifest 用 `export default ` 前缀拼成模块体**。

## dev 期与部署描述

开发期有两点：**`nuxt.server` 持有正在运行的 dev server，Nuxt 的 CLI 消费它**——官方 builder 在其上暴露 `handler`（一个 Node 请求监听器）、`fetch`（一个 Web fetch 处理器）、以及 `reload` / `close` 方法；**自己建 dev server 还是委托 Nitro 的（`createDevServer` from `nitro/builder`）由 builder 决定**。重载**通过调用 `nuxt.server.reload()` 表示**，官方 builder 会发出一个 `compiled` 钩子（例如 `vite:compiled`、`webpack:compiled`）让服务端集成监听以触发重载。**dev 下构建产物通常接的是内存里的活数据而不是磁盘文件**。

给部署工具的那份描述**在 `nuxt.serverBuild` 上，用 `@nuxt/kit/internal` 里的 `useServerBuild()` 读**：

```ts
import { useServerBuild } from '@nuxt/kit/internal'

const build = useServerBuild()

build.output.root() // the project root: where a target's own config file lives
build.output.dir() // the build output directory
build.output.publicDir() // the deployable static assets within it
```

三条要点：**每个路径都是函数**，因为 builder 可能只在初始化过程中才解析它，所以**要在服务端 builder 已运行之后的钩子里调用**；**`buildsSeparately` 声明这份服务端构建是不是独立一趟**（`@nuxt/nitro-server` 是、`@nuxt/vite-server` 不是），**需要知道的一方从描述里读，而不是去猜有没有 Nitro 实例**；**`output.root()` 故意不是打包器的 root**——因为 **Nuxt 把 Vite 的 `root` 指向 `srcDir`**，从打包器 root 去找配置文件的部署目标会**跑到你的应用目录里找**。

最后一条 `::warning` 划定成熟度：**`nuxt.serverBuild` 与 `setServerBuild()`／`useServerBuild()` 是实验性的，从 `@nuxt/kit/internal` 导出；在第二个服务端 builder 建设期间，这个形状会在不升大版本的情况下改变**。

页面末尾两条链接分别指向"Nuxt 内部机制"（讲构建与运行时切分）与 Kit 的 builder 工具页。

## 一句话取用

**这一节对多数人的用处是判断"该写模块还是该写 builder"**：**能靠钩子与 Kit 四通道解决的就不要写 builder**；真要写，**要落的只有 `bundle` 一个方法，其余全在这份六键契约里**。

这一节未记载的：`nuxt.buildOutputs` 里 stub 的具体实现位置、`experimental.watcher: 'builder'` 的默认值与副作用、自定义 builder 如何注册 dev 的 handler 之外的接口、以及 `entryChunkName` 与 import map 的使用条件（**只有表格里一句括号说明**）。

下一篇是一篇短配方：把现成的 Vite 插件接进 Nuxt。

## 下一篇

下一篇 681：用 Vite 插件接入 Nuxt：一条配置数组、addVitePlugin 与那条替代建议

## 本文依据

- https://nuxt.com/docs/4.x/guide/going-further/builders
