---
url: /posts/c17157c69265d79bf4ca2e37418c4fec/
title: 635 - 基础配方九节：每类资产用哪个 Kit 函数、路径为什么必须 resolver
date: 2027-05-11T11:00:00+08:00
updated: 2027-05-11T11:00:00+08:00
author: cmdragon

summary:
  摘要：模块注入东西的九种情况，官方在基础配方页一节一条：**改配置**直接写 `nuxt.options`、**暴露选项给运行时**用 `runtimeConfig.public`、**插件** `addPlugin`、**组件** `addComponent` 或 `addComponentsDir`、**组合式函数** `addImports` 或 `addImportsDir`、**带 key 的函数**推 `optimization.keyedComposables`、**路由中间件** `addRouteMiddleware`、**服务端路由** `addServerHandler`、**其他资产** `nuxt.options.css` 或 Nitro 的 `publicAssets`。除改配置外**每个示例都以 `createResolver(import.meta.url)` 开头**。这一节最容易出事的是带 key 函数那一节的两条警告：**它不跟 barrel 导出**（必须从 `source` 指定的那个文件导入）、**调用必须静态可分析**（重命名解构、当回调传、动态属性访问都不注入）。

categories:
  - 前端开发

tags:
  - nuxt4
  - 模块注入
  - Kit工具函数
  - keyedComposables
  - 服务端路由
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模块写完跑不起来，最常见的原因不是逻辑错，是**注入的那一行没生效**：路径算错了、名字撞了、或者用了自动导入。这一节九节，每一节都是"要注入什么 → 用哪个函数"的对应关系。

## 九节对应表

| 节 | 要做什么 | 用到的东西 |
| --- | --- | --- |
| Modify Nuxt Configuration | 改宿主的配置 | 直接写 `nuxt.options`，复杂改动建议用 defu |
| Expose Options to Runtime | 让运行时代码读到模块选项 | `nuxt.options.runtimeConfig.public` ＋ defu |
| Add Plugins | 注入插件 | `addPlugin` |
| Add Components | 注入组件 | `addComponent` / `addComponentsDir` |
| Add Composables | 注入组合式函数 | `addImports` / `addImportsDir` |
| Add Keyed Functions（`###`） | 注册服务端与客户端要保持一致的函数 | `nuxt.options.optimization.keyedComposables` |
| Add Route Middleware | 注入路由中间件 | `addRouteMiddleware` |
| Add Server Routes | 注入 API 路由 | `addServerHandler` |
| Add Other Assets | 注入样式表或一堆静态资源 | `nuxt.options.css` / Nitro 的 `publicAssets` |
| Use Other Modules | 依赖别的模块 | `moduleDependencies` |

严格说页面上是 9 个 `##` 加 1 个 `###`，最后一节又跳回依赖那一节。

## 起手两行是所有示例的共同点

```js
import { defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  setup (options, nuxt) {
    // We create the `experimental` object if it doesn't exist yet
    nuxt.options.experimental ||= {}
    nuxt.options.experimental.componentIslands = true
  },
})
```

改配置这一节的示例特意写了 `||=` 那一行：**宿主可能压根没有 `experimental` 这个对象**，直接赋值会炸。往任何可能不存在的中间层写值时都要这样兜一下。

页面紧接着给的一句是：需要处理更复杂的配置改动时，**应该考虑用 defu**。

## 把选项暴露给运行时

前提说得很直白：**模块不是应用运行时的一部分，所以模块选项也不是**。要在运行时代码里读到，就走 Nuxt 的 `runtimeConfig`。

```js
import { defineNuxtModule } from '@nuxt/kit'
import { defu } from 'defu'

export default defineNuxtModule({
  setup (options, nuxt) {
    nuxt.options.runtimeConfig.public.myModule = defu(nuxt.options.runtimeConfig.public.myModule, {
      foo: options.foo,
    })
  },
})
```

**这里用 defu 的目的是"扩展用户提供的公共运行时配置，而不是覆盖它"**——用户如果自己也写了 `runtimeConfig.public.myModule`，两边要合起来。

读的时候：

```js
import { useRuntimeConfig } from '@nuxt/kit'

const options = useRuntimeConfig().public.myModule
```

这一节有个 `::warning` 是安全线：**不要把敏感的模块配置放到 public runtime config 上，比如私有 API key，它们最终会出现在公开 bundle 里**。

两处要留意：这节代码块上面留着一条未完成的注释 `<!-- TODO: Update after #18466 (or equivalent) -->`；示例把 `useRuntimeConfig` 从 `@nuxt/kit` 导入，**而这一节没有解释这个导入面与运行时应从 `#imports` 之类显式导入之间的关系**（那条限制在模块结构页的警告里）。

## 插件、组件、组合式函数

```js
import { addPlugin, createResolver, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  setup (options, nuxt) {
    // Create resolver to resolve relative paths
    const resolver = createResolver(import.meta.url)

    addPlugin(resolver.resolve('./runtime/plugin'))
  },
})
```

组件两个函数、三种入参形状：

```ts
addComponent({
  name: 'MySuperComponent', // name of the component to be used in vue templates
  export: 'MySuperComponent', // (optional) if the component is a named (rather than default) export
  filePath: resolver.resolve('runtime/app/components/MySuperComponent.vue'),
})

// From a library
addComponent({
  name: 'MyAwesomeComponent',
  export: 'MyAwesomeComponent',
  filePath: '@vue/awesome-components',
})

addComponentsDir({
  path: resolver.resolve('runtime/app/components'),
})
```

`export` 这个可选键管的是**具名导出还是默认导出**；`filePath` 除了指文件，还能**直接指一个库**。

组合式函数是 `addImports`，三个键：

```ts
addImports({
  name: 'useComposable', // name of the composable to be used
  as: 'useMyComposable', // optional alias that will be available for the consuming apps
  from: resolver.resolve('runtime/app/composables/useComposable'), // path of composable
})
```

`as` 是**给使用方应用暴露的别名**，这是组件那条没有的键。多条可以传数组，整目录用 `addImportsDir(resolver.resolve('runtime/composables'))`。

组件与组合式函数这两节末尾各重复了同一条 `::note` 和同一条 `::tip`：

> Note that all components, pages, composables and other files that would be normally placed in your `app/` folder need to be in `runtime/app/`. This will mean they can be type checked properly.

**该放 `app/` 的文件要放到 `runtime/app/`**，理由是类型检查能正常工作。这不是可选的目录美化——上面两个示例的路径都写着 `runtime/app/...`。`::tip` 重复的是**给导出加前缀**以免与用户代码或其他模块冲突。

## 带 key 的函数：最容易静默失效的一节

用途：**服务端与客户端之间要保持状态一致的函数**，官方举的例子就是内置的 `useState` 与 `useAsyncData`。机制一句话：**函数被调用时实参少于指定数量，编译器自动注入一个额外的 key 参数**，这个 key 在 SSR 与客户端 hydration 之间保持稳定。`::tip` 补了 key 的来源：**由文件路径与调用位置算出的哈希**。

```ts
nuxt.options.optimization.keyedComposables.push({
  name: 'useMyState',
  source: resolver.resolve('./runtime/composables/state'),
  argumentLength: 2,
})
```

三个属性的定义表：

| 属性 | 类型 | 文档描述 |
| --- | --- | --- |
| `name` | `string` | 函数名。**默认导出写 `'default'`**，可调用名从文件名按 camelCase 推导 |
| `source` | `string` | 函数定义所在文件的**已解析路径**，支持 Nuxt 别名（`~`、`@` 等） |
| `argumentLength` | `number` | 函数接受的最大参数数量；**调用时参数少于它就注入一个唯一 key** |

`argumentLength: 2` 时的三种结果：

```ts
useMyState() // useMyState('$HJiaryoL2y')
useMyState('myKey') // useMyState('myKey', '$HJiaryoL2y')
useMyState('a', 'b') // not transformed (already has 2 arguments)
```

两条 `::warning` 才是这节的重点。

**第一条：不跟 barrel 导出。** 注入插件校验的是每个调用的**确切已解析导入源**，函数必须从 `source` 指定的那个文件导出：

```ts
// ✅ Works - direct import matches the configured source
import { useMyState } from 'my-module/runtime/composables/state'

// ❌ Won't work - re-exported through a barrel file
import { useMyState } from 'my-module/runtime/composables' // index.ts barrel
```

**第二条：调用必须静态可分析。** 能注入的两种写法是直接调用 `useMyState()` 和命名空间调用 `composables.useMyState()`；**不能注入的六种**是动态属性访问、把函数赋给变量再调、当回调传出去、嵌套作用域里带重命名的解构（示例给的是 `const { useMyState: localState } = composables`），以及前面那条 barrel。

这节的失效方式最恼人：**代码不报错，只是 key 没注入**，表现是服务端与客户端各拿一份状态。

## 中间件、服务端路由与其他资产

```ts
addRouteMiddleware({
  global: true,
  name: 'name-of-your-middleware',
  path: resolver.resolve('./runtime/middleware/name-of-your-middleware'),
})
```

服务端路由三种形状都在示例里：固定路由、`:name` 参数配 `[name].get` 文件、以及 catch-all 的 `/api/_my-module/files/**:path` 配 `[...path].get`：

```ts
addServerHandler({
  route: '/api/_my-module/hello',
  handler: resolver.resolve('./runtime/server/api/hello/index.get'),
})
```

这一节的 `::tip` 单独强调服务端路由要加前缀，理由很具体：**`/api/auth`、`/api/login`、`/api/user` 这类常见路径很可能已经被应用用了**。示例里用的前缀形态是 `_my-module`。

其他资产两种：塞样式表就是一行 `nuxt.options.css.push(resolver.resolve('./runtime/style.css'))`；公开一整个目录走 Nitro：

```js
nuxt.hook('nitro:config', (nitroConfig) => {
  nitroConfig.publicAssets ||= []
  nitroConfig.publicAssets.push({
    dir: resolver.resolve('./runtime/public'),
    maxAge: 60 * 60 * 24 * 365, // 1 year
  })
})
```

`publicAssets ||= []` 与第一节的 `experimental ||= {}` 是同一个模式。

最后一节 Use Other Modules 只有一个空壳示例（`moduleDependencies: { '@nuxtjs/tailwindcss': {} }`）加一个跳转。

这一节没写的：`addPlugin` 的注册顺序选项、`addComponent` 的 `transpile` 等其余键（参数表在 Kit 的对应小节）、以及 key 注入失败时有没有任何提示。

下一篇讲高级配方：钩子、虚拟文件与类型注入。

## 下一篇

下一篇 636：高级配方六节：钩子表、自定义钩子、虚拟文件与类型注入

## 本文依据

- https://nuxt.com/docs/4.x/guide/modules/recipes-basics
