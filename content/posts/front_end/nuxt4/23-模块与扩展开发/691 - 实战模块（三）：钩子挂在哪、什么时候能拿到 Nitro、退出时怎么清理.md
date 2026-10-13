---
url: /posts/b6082cc3c686fcd3f9f3d484864857f9/
title: 691 - 实战模块（三）：钩子挂在哪、什么时候能拿到 Nitro、退出时怎么清理
date: 2027-05-30T10:00:00+08:00
updated: 2027-05-30T10:00:00+08:00
author: cmdragon

summary:
  摘要：模块跟框架的交互几乎全靠钩子，官方给的两节在这一块有四条硬规矩：钩子可以用定义对象里的 `hooks` 映射挂，也可以在 `setup` 里用 `nuxt.hook` 程序化地挂；如果你开了 watcher，Nuxt 生命周期结束时要靠 `close` 钩子把它关掉；想让别的模块订阅你的自定义钩子，触发点要放在 `modules:done` 里；`useNitro()` 只能在 `ready` 之后调用，而且改了它的配置也不会生效。

categories:
  - 前端开发

tags:
  - nuxt4
  - 模块开发
  - 生命周期钩子
  - nitro
  - 预渲染
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 两种挂法，一种声明一种命令式

高级配方页的第一节说：钩子几乎能让你扩展 Nuxt 的每个方面，模块可以**以编程方式**挂钩子，也可以通过定义对象里的 `hooks` 映射来挂。官方示例把两种放在同一个模块里：

```js
import { addPlugin, createResolver, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  // Hook to the `app:error` hook through the `hooks` map
  hooks: {
    'app:error': (err) => {
      console.info(`This error happened: ${err}`)
    },
  },
  setup (options, nuxt) {
    // Programmatically hook to the `pages:extend` hook
    nuxt.hook('pages:extend', (pages) => {
      console.info(`Discovered ${pages.length} pages`)
    })
  },
})
```

（示例的 import 里带了 `addPlugin` 和 `createResolver` 但代码里没用到，这是文档原样。）

`hooks` 在定义对象里的位置是这样的：类型 `Partial<NuxtHooks>`、非必填，说明只有一句——"为模块安装的钩子；如果提供了，模块就会安装这些钩子"。**文档没有说明这个映射里钩子注册的时机**，与 `setup` 内 `nuxt.hook` 的先后关系要靠你自己排。

## 开了 watcher 就要在 `close` 里收尾

这一节后面单独挂了一条关于清理的 note，条件写得很具体：

> 如果你的模块会打开、持有或启动一个 watcher，那么当 Nuxt 生命周期结束时你应该把它关掉。`close` 钩子就是为此提供的。

```ts
import { defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  setup (options, nuxt) {
    nuxt.hook('close', async (nuxt) => {
      // Your custom code here
    })
  },
})
```

这条在开发时最容易漏：`nuxt dev` 反复重启，或者构建进程退出后残留句柄，通常就是模块没做这一步。讲构建器的那一节也出现过同一个钩子——builder 如果被要求接管文件监听，应当注册一个 `close` 钩子来清理。

## 自定义钩子的触发时机是有讲究的

模块也能定义并调用自己的钩子，文档把这称为"让你的模块可扩展"的有力模式。但它给了一条明确的时间点建议：

> 如果你预期其他模块能够订阅你模块的钩子，那么你应该在 `modules:done` 钩子里调用它们。这能确保所有其他模块都有机会完成自己的 setup，并在各自的 `setup` 函数里向你的钩子注册监听器。

示例三段都在同一个文件里：

```ts
// my-module/module.ts
import { defineNuxtModule } from '@nuxt/kit'

export interface ModuleHooks {
  'my-module:custom-hook': (payload: { foo: string }) => void
}

export default defineNuxtModule({
  setup (options, nuxt) {
    // Call your hook in `modules:done`
    nuxt.hook('modules:done', async () => {
      const payload = { foo: 'bar' }
      await nuxt.callHook('my-module:custom-hook', payload)
    })
  },
})
```

两个细节：钩子名带模块名前缀（`my-module:custom-hook`），以及 `callHook` 前面有 `await`——你的监听者可能是异步的。

## Nitro 实例：两个时机问题加一个不存在的情况

讲 Kit 服务端函数那一节把 Nitro 相关的函数集中在一起，`useNitro` 和 `tryUseNitro` 这两个取实例的函数各有一条时机限制，而且这两条经常写错。

`useNitro` 返回 Nitro 实例，配两条提示：

> 你只能在 `ready` 钩子之后调用 `useNitro()`。

> 对 Nitro 实例配置的改动不会被应用。

第二条是很多人在这里白费时间的地方：拿到实例以后改它的配置字段，不会有效果。官方示例本身就示范了正确姿势——把用法放在 `nuxt.hook('ready', () => { const nitro = useNitro() })` 里。

`tryUseNitro` 返回实例**或 `undefined`**，文档写明了两种拿不到的情况：一是 `ready` 钩子还没跑完之前本来就没有实例；二是配置里的 `server.builder` 不使用 Nitro 时压根没有实例，例子是那种只输出客户端 SPA 的 builder。文档给的结论是：任何"没有服务端也应该继续工作"的逻辑都用这个函数，因为**服务端路由、路由规则、预渲染在那种情况下全部不存在**。

```ts
import { defineNuxtModule, tryUseNitro } from '@nuxt/kit'

export default defineNuxtModule({
  setup (options, nuxt) {
    nuxt.hook('ready', () => {
      const nitro = tryUseNitro()
      if (!nitro) {
        // no server: skip anything that would only run there
        return
      }
    })
  },
})
```

## 服务端插件有一条显式导入的硬要求

`addNitroPlugin` 用来扩展 Nitro 的运行时行为，它的名字在 Nuxt v4.6 之前是 `addServerPlugin`。这一条警告容易被违反，因为它违反之后在开发时可能看起来是好的：

> 你必须在你自己的插件文件里显式地从 `nitropack/runtime` 导入 `defineNitroPlugin`。同样的要求也适用于 `useRuntimeConfig` 这类的工具函数。

也就是说 runtime 插件文件顶部要写 import，不能像应用代码那样裸用。注册侧只需要一个路径：

```ts
import { addNitroPlugin, createResolver, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  setup () {
    const { resolve } = createResolver(import.meta.url)
    addNitroPlugin(resolve('./runtime/plugin.ts'))
  },
})
```

插件文件里 `export default defineNitroPlugin((nitroApp) => { ... })`，文档示例挂了三个运行时钩子：`request`、`beforeResponse`、`afterResponse`。这三个钩子各自的参数与语义在讲 Nitro 运行时钩子的那篇逐项拆过，这里不重复。

## 把模块选项接到预渲染上

`addPrerenderRoutes` 加要预渲染的路由，官方给的示例正好是"模块元信息 + 默认值 + 选项分支"三件事凑在一起的最小完整形状：

```ts
import { addPrerenderRoutes, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  meta: {
    name: 'nuxt-sitemap',
    configKey: 'sitemap',
  },
  defaults: {
    sitemapUrl: '/sitemap.xml',
    prerender: true,
  },
  setup (options) {
    if (options.prerender) {
      addPrerenderRoutes(options.sitemapUrl)
    }
  },
})
```

读这段要看清三层：`meta.configKey` 决定用户在 `nuxt.config` 里用哪个键配置（这里是 `sitemap`）；`defaults` 让 `options.prerender` 有值可判断；`addPrerenderRoutes` 接受一个字符串或字符串数组。用户把 `prerender` 设为 `false`，这条路由就不会被预渲染。

## 这一层做完之后的检查

- 需要监听框架事件的，全部走钩子而不是 `setup` 里的直接执行；一次性初始化的放 `onInstall`。
- 有 watcher、有句柄、有子进程的，注册 `close`。
- 对外提供扩展点的自定义钩子，触发放 `modules:done`。
- 任何碰 Nitro 实例的代码放在 `ready` 之后；纯客户端场景用 `tryUseNitro` 并写好没有实例的分支。
- runtime 的服务端插件文件顶部有显式 import。

下一篇是最后一段：把测试和发布接上——E2E 的五个步骤、`npm pack` 的外部项目验证，以及 `nuxt build-module` 那六个开关各自改变什么。

## 下一篇

下一篇 692：实战模块（四）：E2E 五步、npm pack 手测与 release 的六步

## 本文依据

- https://nuxt.com/docs/4.x/guide/modules/recipes-advanced
- https://nuxt.com/docs/4.x/api/kit/nitro
