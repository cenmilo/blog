---
url: /posts/e6859a028b7638657ef0d2ec895288b8/
title: 426 - import.meta 在样式与资源路径里的用法
date: 2027-03-03T09:00:00+08:00
updated: 2027-03-03T09:00:00+08:00
author: cmdragon

summary:
  摘要：`import.meta` 在 Nuxt 里分两类属性，边界官方写得很清。**运行时（应用）属性**是"statically injected and can be used for tree-shaking your runtime code"——八个：`client`/`browser`（客户端为真）、`server`/`nitro`（服务端为真）、`dev`（跑 dev server 时）、`envName`（v4.5+，字符串，含 `--envName` 传的自定义值）、`test`（测试上下文）、`prerender`（**构建的预渲染阶段**在服务端渲 HTML 时为真）。**构建器属性**只有两个，且"available both in modules and in your `nuxt.config`"：`import.meta.env`（等于 `process.env`）与 `import.meta.url`（当前文件的可解析路径）。样式与资源上最实用的就是 `import.meta.url` 配 `createResolver` 解析模块内路径，以及 `import.meta.prerender` 在预渲染阶段跳过按请求取数——注意 Vite 那个 `new URL(..., import.meta.url)` 写法**在 SSR 下不工作**。

categories:
  - 前端开发

tags:
  - nuxt4
  - import.meta
  - prerender
  - createResolver
  - 资源路径
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

写样式和资源路径时反复遇到三个东西：`import.meta.client`、`import.meta.prerender`、`import.meta.url`。它们不是同一类属性——一类是运行时注入的布尔判断、一类是构建期解析路径。这一篇把官方那张表逐行落地。

## 定义

> With ES modules you can obtain some metadata from the code that imports or compiles your ES-module. This is done through `import.meta`, which is an object that provides your code with this information. Throughout the Nuxt documentation you may see snippets that use this already to figure out whether the code is currently running on the client or server side.

## 第一类：运行时（应用）属性

官方给这类定了一条重要性质：**"These values are statically injected and can be used for tree-shaking your runtime code."** ——静态注入，所以能被用来摇掉不会执行的那部分代码。八个属性：

| 属性 | 类型 | 官方描述 |
| --- | --- | --- |
| `import.meta.client` | boolean | True when evaluated on the client side. |
| `import.meta.browser` | boolean | True when evaluated on the client side. |
| `import.meta.server` | boolean | True when evaluated on the server side. |
| `import.meta.nitro` | boolean | True when evaluated on the server side. |
| `import.meta.dev` | boolean | True when running the Nuxt dev server. |
| `import.meta.envName` :badge[v4.5] | string | The current Nuxt environment name, including custom values passed via `--envName`. |
| `import.meta.test` | boolean | True when running in a test context. |
| `import.meta.prerender` | boolean | True when rendering HTML on the server in the prerender stage of your build. |

注意成对重复：`client`/`browser` 描述一字不差（都是"客户端求值为真"）、`server`/`nitro` 同理——是**别名关系**，不是两种不同判断。`envName` 是唯一的字符串，且官方明确它会带上你用 `--envName` 传的自定义环境名（363 那套），其余七个都是 boolean。

`prerender` 的含义要读准：**不是**"这是个静态站点"，而是"当前正处在构建的预渲染阶段、在服务端渲 HTML"。

## 第二类：构建器属性

> These values are available both in modules and in your `nuxt.config`.

| 属性 | 类型 | 官方描述 |
| --- | --- | --- |
| `import.meta.env` | object | Equals `process.env`. |
| `import.meta.url` | string | Resolvable path for the current file. |

两个要点：**只在模块与 `nuxt.config` 里**（不是应用代码的通用能力）；`import.meta.url` 的价值在于"可解析的当前文件路径"。

## 用法一：模块里解析资源路径

官方给的标准做法是 `createResolver(import.meta.url)`：

```ts [modules/my-module/index.ts]
import { createResolver } from 'nuxt/kit'

// Resolve relative from the current file
const resolver = createResolver(import.meta.url)

export default defineNuxtModule({
  meta: { name: 'myModule' },
  setup () {
    addComponent({
      name: 'MyModuleComponent',
      // Resolves to '/modules/my-module/components/MyModuleComponent.vue'
      filePath: resolver.resolve('./components/MyModuleComponent.vue'),
    })
  },
})
```

为什么必须这样：模块的相对路径不能相对"运行时的 cwd"解析，只能相对模块自身文件；`import.meta.url` 给的正是后者。Kit 那一节注册插件也是同一招（`const { resolve } = createResolver(import.meta.url)` 再 `addPlugin({ src: resolve('runtime/plugin.js'), mode: 'client' })`）——注册组件、注册插件、指运行时文件，都走它。

## 用法二：预渲染阶段跳过按请求的东西

421 提到过 `import.meta.prerender` 用于样式/资源场景，它的官方出处是渲染那一节的错误页示例：

```vue
const { data: suggestions } = await useAsyncData(
  'error-suggestions',
  () => $fetch('/api/suggestions', { query: { path: route.path } }),
  // at build time the path is `/404.html`, so fetch on the client instead
  { server: !import.meta.prerender },
)
```

配套的官方解释是：

> `import.meta.prerender` is only `true` while the page is being generated, so the same `error.vue` still server-renders its data when the app is deployed with a server.

这条对样式的意义在于：**预渲染时路径/请求是构建期的值**（例子里路径是 `/404.html`），运行时才知道的信息不能在这时候取——`server: !import.meta.prerender` 就是把"这一步交给客户端"写成条件。同理，请求相关内容要包进 `<ClientOnly>`（421/376）。

## 一条要记住的坑：Vite 的 new URL 写法

421 从资源侧引过这条 `::warning`，它在 `import.meta` 语境下更要紧：

> Await a lazy import before using its URL in server-rendered markup. Vite's `new URL(..., import.meta.url)` pattern does not work with SSR.

也就是说社区里常见的 `new URL('./img/x.png', import.meta.url)` 这种拿资源 URL 的写法，**在 SSR 下不工作**。Nuxt 文档给的替代是显式 `?url` 导入或 `import.meta.glob`（421），并且在服务端渲出的标记里用该 URL 之前要先 await 那个懒导入。

## 自查

- 运行时属性八个（client/browser、server/nitro 各是同义对、dev、envName(v4.5+，字符串，含 `--envName` 值)、test、prerender），**静态注入、可用于 tree-shaking**。
- `prerender` 指"正处在构建的预渲染阶段"，不是"我是静态站"。
- 构建器属性只有两个：`env`（等于 `process.env`）、`url`（当前文件可解析路径），且官方限定为**模块与 nuxt.config 里可用**。
- 模块里解析路径的正解是 `createResolver(import.meta.url).resolve('./...')`（组件、插件、运行时文件同理）。
- 预渲染阶段不能取按请求才知道的值：用 `server: !import.meta.prerender` 交给客户端，请求相关标记包 `<ClientOnly>`。
- Vite 的 `new URL(..., import.meta.url)` 在 SSR 不工作；要 URL 用 `?url` 或 `import.meta.glob`，并在 SSR 标记里用之前先 await。

## 下一篇

下一篇 427 纠正一个 3.x 时代的名字：4.x 里没有 `inlineSSRStyles`，控制内联粒度的是 `features.inlineStyles`，默认只内联 Vue 组件样式、不再内联全局 CSS。

## 本文依据

- `import.meta` 定义、Runtime (App) Properties 八行表（含 "statically injected and can be used for tree-shaking"、envName 的 v4.5 徽标与 `--envName` 说明、prerender 描述）、Builder Properties 两行表与"available both in modules and in your nuxt.config"、`createResolver(import.meta.url)` + `addComponent` 示例（`api/advanced/import-meta`）；`addPlugin` 同一写法（`api/kit/plugins`，见 296）；`import.meta.prerender` 在错误页的用法与"only `true` while the page is being generated"解释（`guide/concepts/rendering`，见 382）；`::warning` 关于 `new URL(..., import.meta.url)` 不支持 SSR（`getting-started/assets`，见 421）。
