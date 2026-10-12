---
url: /posts/cfe3e0b47aad2bb956023214e61c7681/
title: 模块定义文件长什么样：两种写法、五个键与 wrapper 的八条保证
date: 2027-05-11T09:00:00+08:00
updated: 2027-05-11T09:00:00+08:00
author: cmdragon

summary:
  摘要：打开 starter 生成的项目，模块本体就是 **`src/module.ts` 这一个文件**。官方在这一页先给你**最低层的写法**——一个接收 `inlineOptions` 与 `nuxt` 的（可异步）函数——然后立刻说**不推荐这种**，推荐带 `meta` 的对象写法，尤其是要发到 npm 的模块。对象形态里这一页出现了**五个键**：`meta`、`defaults`、`hooks`、`moduleDependencies`、`setup`。`defineNuxtModule` 返回的仍然是那个低层签名的包装函数，它替你做了**八件事**：合并选项、类型推断、用 `meta.name` 或 `meta.configKey` 算出的唯一键**保证只装一次**、自动注册钩子、自动查兼容性、暴露 `getOptions` 与 `getMeta`、保证前后兼容、对接 module builder。运行时代码放 `src/runtime/`，但这一页有条警告：**发布出去的模块在自己的 runtime 里用不了自动导入**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 模块定义
  - defineNuxtModule
  - runtime目录
  - 模块作者
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模块有两种来源，但**写法只有一种**。npm 上装来的算一种，写在自己项目里的算另一种——官方原话是"in either case, they work in the same way"。这一页讲的就是那个共同形状。

## 先定位那个文件

`::note` 第一句就给位置：**用 starter 时，模块定义在 `src/module.ts`**。

定义文件是模块的入口点——**当 Nuxt 在配置里看到这个模块被引用时，加载的就是它**。

## 最低层的写法（官方不推荐）

低层来看，一个模块定义就是一个**简单的、可能是异步的函数**，接收用户的内联选项和一个 `nuxt` 对象用来与 Nuxt 交互：

```ts
export default function (inlineOptions, nuxt) {
  // You can do whatever you like here..
  console.log(inlineOptions.token) // `123`
  console.log(nuxt.options.dev) // `true` or `false`
  nuxt.hook('ready', (nuxt) => {
    console.log('Nuxt is ready')
  })
}
```

这三行示例把参数怎么用说全了：**选项从第一个参数读**（`inlineOptions.token` 打出 `123`，对应配置数组里元组那种带选项的写法）、**`nuxt.options` 是只读快照式的配置**（`nuxt.options.dev` 是 `true` 或 `false`）、**钩子通过 `nuxt.hook` 挂**。

想要类型提示，可以用 Nuxt Kit 提供的 `defineNuxtModule` 包一层，但仍然写成函数：

```ts
import { defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule((options, nuxt) => {
  nuxt.hook('pages:extend', (pages) => {
    console.log(`Discovered ${pages.length} pages`)
  })
})
```

然后官方明确表态：**不推荐用这种低层函数定义**。推荐的形态是**带 `meta` 属性的对象写法**，尤其是准备发到 npm 的时候。给的理由是：这个 helper 实现了模块常用的许多模式、**保证未来兼容性**、同时改善模块作者和模块用户两边的体验。

## 推荐的形态：五个键

```ts
import { defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  meta: {
    // Usually the npm package name of your module
    name: '@nuxtjs/example',
    // The key in `nuxt.config` that holds your module options
    configKey: 'sample',
    // Compatibility constraints
    compatibility: {
      // Semver version of supported nuxt versions
      nuxt: '>=3.0.0',
    },
  },
  // Default configuration options for your module, can also be a function returning those
  defaults: {},
  // Shorthand sugar to register Nuxt hooks
  hooks: {},
  // Configuration for other modules - this does not ensure the module runs before
  // your module, but it allows you to change the other module's configuration before it runs
  moduleDependencies: {
    'some-module': {
      // You can specify a version constraint for the module. If the user has a different
      // version installed, Nuxt will throw an error on startup.
      version: '>=2',
      // By default moduleDependencies will be added to the list of modules to be installed
      // by Nuxt unless `optional` is set.
      optional: true,
      // Any configuration that should override `nuxt.options`.
      overrides: {},
      // Any configuration that should be set. It will override module defaults but
      // will not override any configuration set in `nuxt.options`.
      defaults: {},
    },
  },
  // The function holding your module logic, it can be asynchronous
  setup (moduleOptions, nuxt) {
    // ...
  },
})
```

五个键的分工：

| 键 | 这一页给的说法 | 值得注意的点 |
| --- | --- | --- |
| `meta` | 标识你的模块 | 注释把三个子键各自的角色写清了：`name` 通常是 npm 包名、`configKey` 是 `nuxt.config` 里放你选项的那个键、`compatibility.nuxt` 是 semver 约束 |
| `defaults` | 模块的默认配置 | 注释里多写了半句："**也可以是一个返回这些值的函数**" |
| `hooks` | 注册 Nuxt 钩子的简写糖 | 与在 `setup` 里调 `nuxt.hook` 是两条路，指向同一个钩子表 |
| `moduleDependencies` | 别的模块的配置 | 注释先划了一道：**它不保证那个模块跑在你的模块之前**，只是允许你在它运行前改它的配置 |
| `setup` | 装你的逻辑的函数 | 可以是异步 |

`moduleDependencies` 里那四个字段（`version`、`optional`、`overrides`、`defaults`）在这页只给了注释级说明，专门的一页有各自的定义——`version` 不满足时是**启动直接报错**，`optional` 不设时**默认会被加进要安装的模块列表**，`overrides` 覆盖 `nuxt.options`，`defaults` 只覆盖模块默认、**不覆盖用户在 `nuxt.options` 里写过的值**。

## wrapper 替你做的八件事

`defineNuxtModule` 返回的还是一个低层签名的 `(inlineOptions, nuxt)` 函数，只不过这个 wrapper 在调用你的 `setup` 之前会先套用默认值和其他必要步骤。这一页列了八条：

- 支持 `defaults` 与 `meta.configKey`，**自动合并模块选项**
- 类型提示与自动类型推断
- **用从 `meta.name` 或 `meta.configKey` 算出的唯一键，保证模块只被安装一次**
- 自动注册 Nuxt 钩子
- 基于模块 meta **自动检查兼容性问题**
- 为 Nuxt 的内部使用暴露 `getOptions` 与 `getMeta`
- 只要模块用的是最新版 `@nuxt/kit` 的 `defineNuxtModule`，就**保证向后与向上的兼容**
- 与 module builder 工具链集成

第三条是这八条里唯一带"如果你自己写就一定会做"性质的：自己写低层函数时，同一个模块被配置数组和分层各引用一次会真的跑两遍。**唯一键的来源是 `meta.name` 或 `meta.configKey`**，所以这两个至少得写一个——这一页没写"都不给会怎样"，只写了算键的两个来源。

## 运行时代码放哪

`::note` 同样先给位置：**用 starter 时，运行时目录是 `src/runtime/`**。

关键的那句前提：**模块和配置里的其他东西一样，不会被打进你的应用运行时**。但模块往往要往宿主应用里提供或注入运行时代码，runtime 目录就是为这件事存在的。

这一页按三组列出能放的东西：

| 目标 | 能放的资产 |
| --- | --- |
| Nuxt 应用侧 | Vue 组件、Composables、Nuxt 插件 |
| 服务端引擎 Nitro | API 路由、中间件、Nitro 插件 |
| 其他想注入的 | 样式表、3D 模型、图片、等等 |

放进去之后，**从模块定义里把这些资产注入应用**（具体注入函数在配方页，本系列的 635 逐节讲）。

## 那条警告只针对发布出去的模块

> Published modules cannot leverage auto-imports for assets within their runtime directory. Instead, they have to import them explicitly from `#imports` or alike.

原因是同一节的后半句：**`node_modules` 里的文件不启用自动导入，出于性能考虑**——而发布模块最终就住在那儿。

这解释了一种很烦的现象：**本地模块里 `useRuntimeConfig()` 直接写就能用，同样的文件发到 npm 装回来就报未定义**。不是配置丢了，是这个文件换了位置、自动导入不再覆盖它。starter 的 playground 里跑得好，不等于发出去还能跑。

## 这一页没写的两件事

- **完整的参数表在 Kit 那一页**：这页只出现五个键，而 Kit 的 `defineNuxtModule` 参数表列了八个（多出 `schema`、`onInstall`、`onUpgrade`）。要知道全部键得翻 Kit 页。
- **runtime 目录的文件怎么被解析成路径**：这页只说"从模块定义注入"，具体是 `createResolver(import.meta.url)` 加 `resolver.resolve(...)`，在配方页的每个示例里都是这两行。

按顺序做的自查：先看 `src/module.ts` 里对象有没有 `meta.configKey`；再看 `defaults` 是不是需要写成函数（依赖 `nuxt` 实例时）；然后看 `src/runtime/` 里的每个文件**是否都显式 import 了它用到的东西**。

下一篇讲选项合并：用户在配置里写的那些值，怎么和你 `defaults` 里的默认值合到一起。

## 本文依据

- https://nuxt.com/docs/4.x/guide/modules/module-anatomy
- https://nuxt.com/docs/4.x/api/kit/modules
