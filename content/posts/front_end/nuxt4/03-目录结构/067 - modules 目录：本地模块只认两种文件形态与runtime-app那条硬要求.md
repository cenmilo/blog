---
url: /posts/8831d4d6ce53a25cb230285136cb8c90/
title: 067 - modules 目录：本地模块只认两种文件形态，以及 runtime/app 那条硬要求
date: 2026-11-03T10:00:00+08:00
updated: 2026-11-03T10:00:00+08:00
author: cmdragon

summary:
  摘要：modules/ 目录用来在你的应用里自动注册本地模块，官方说它"是放置你在构建应用过程中开发的任何本地模块的好地方"，而自动注册只认两种文件形态：modules/*/index.ts 与 modules/*.ts，其它形态不会被自动注册，你也无需把这些本地模块另外加进 nuxt.config.ts。官方给的那段注释纠正了一个常见误会：nuxt/kit 是一个"辅助子路径导入"，在你定义本地模块时可以直接用，这意味着你不需要把 @nuxt/kit 加进项目的依赖。这一节最容易被忽略、也最该遵守的是那条 note：所有本来该放在你 app/ 目录里的组件、页面、组合式函数以及其它文件，都必须放在 modules/your-module/runtime/app/ 下面，因为这样才能保证它们被正确类型检查。执行顺序官方给的是两步——先加载在 nuxt.config.ts 里定义的模块，然后执行 modules/ 目录里找到的模块且它们按字母序加载；要改本地模块的顺序，就在每个目录名前面加一个数字（1.first-module/index.ts 与 2.second-module.ts）。

categories:
  - 前端开发

tags:
  - nuxt4
  - 本地模块
  - kit
  - 执行顺序
  - 目录结构

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

045 讲的是装别人的模块。这一篇讲在你自己项目里写模块，`modules/` 目录。它只有三条规则，但其中一条不遵守就会类型报错。

定位与那两种文件形态。使用 modules/ 目录来在你的应用里自动注册本地模块。这里是一个好地方，用来放置你在构建应用过程中开发的任何本地模块。

自动注册的文件匹配模式是，`modules/*/index.ts` 与 `modules/*.ts`。你不需要把这些本地模块另外加进你的 `nuxt.config.ts`。

这两条模式把所有可能的形态钉死了，目录加 `index.ts`，或者一个平铺的 `.ts` 文件。对照 063、066 那条只扫顶层的规则，这里的取舍是同样的味道，别的形态，比如 `modules/hello/main.ts`，不会被注册。这一句是我按官方只列这两种模式得出的判断。

无需再加进 `nuxt.config` 这句，与 061 讲插件时那句所有插件自动注册、不需要额外加进 nuxt.config，是同一条设计。框架这一层的约定就是，目录里有等于生效。

`nuxt/kit` 不用装，那条注释很值钱。官方在这个例子的第一行放了一段注释，我单独拎出来，因为它回答了一个非常常见的犹豫。

```ts [modules/hello/index.ts]
// `nuxt/kit` is a helper subpath import you can use when defining local modules
// that means you do not need to add `@nuxt/kit` to your project's dependencies
import { addComponentsDir, addServerHandler, createResolver, defineNuxtModule } from 'nuxt/kit'

export default defineNuxtModule({
  meta: {
    name: 'hello',
  },
  setup () {
    const resolver = createResolver(import.meta.url)

    // Add an API route
    addServerHandler({
      route: '/api/hello',
      handler: resolver.resolve('./runtime/api-route'),
    })

    // Add components
    addComponentsDir({
      path: resolver.resolve('./runtime/app/components'),
      pathPrefix: true, // Prefix your exports to avoid conflicts with user code or other modules
    })
  },
})
```

```ts [modules/hello/runtime/api-route.ts]
export default defineEventHandler(() => {
  return { hello: 'world' }
})
```

官方说，启动 Nuxt 时，`hello` 模块会被注册，而 `/api/hello` 路由将可用。

这段里有四处值得学。

一，导入源写的是 `nuxt/kit`，不是 `@nuxt/kit`。官方注释明说这是一个辅助子路径导入，并给出结论，你不需要把 `@nuxt/kit` 加进项目依赖。对比 062 那节库作者示例里写的是 `@nuxt/kit`，那是发布成 npm 包的情形，两个导入源分别对应两种场合，官方这两节各写了一处。

二，`createResolver(import.meta.url)` 加 `resolver.resolve('./runtime/...')`。015 讲过 ESM 里没有 `__dirname`，051 的 alias 示例用的也是 `fileURLToPath(new URL(...))`，这一节给的是模块内部的官方写法。

三，`meta.name`。模块要有名字，对照 044 里 `.nuxtrc` 那套 setups 记账需要 `meta.name` 与 `meta.version` 的机制，630 到 699 会展开。

四，`addComponentsDir` 里那行注释，`pathPrefix: true`，Prefix your exports to avoid conflicts with user code or other modules。这是官方替模块作者写的冲突预防提示，组件名带上前缀，免得和使用者代码或别的模块撞名。062 那个 `prefix: 'awesome'` 是同一个目的的另一种写法。

那条 note，应用侧文件必须住进 `runtime/app/`。这一节最容易被忽略、也最该守的硬要求，请注意，所有组件、页面、组合式函数以及其它本来会放在你 `app/` 目录里的文件，都需要放在 `modules/your-module/runtime/app/` 下面。这保证了它们能够被正确地进行类型检查。

注意官方给的理由，不是为了能运行，而是为了能被正确类型检查。这条与 031、036 那套类型按上下文分四份 tsconfig 是同一件事，应用侧代码要落在 app 上下文里，`tsconfig.app.json` 才会覆盖到它。

所以模块内部的目录形状是固定的一套。

```bash
modules/hello/
  index.ts                 # 模块本体（会被自动注册）
  runtime/
    api-route.ts           # 服务端处理器（用 addServerHandler 挂）
    app/
      components/          # ← 组件必须在这里，而不是 runtime/components/
```

`runtime/app/components` 这个路径不是随便起的，它就是上面那段示例代码里 `addComponentsDir` 的 `path`。

执行顺序，`nuxt.config` 里的先跑，目录里的按字母序。官方原话，模块按以下顺序执行，首先，加载在 `nuxt.config.ts` 里定义的模块，然后，执行在 `modules/` 目录里找到的模块，且它们按字母序加载。

这一条对调试很关键，本地模块永远跑在你显式声明的那些模块之后。如果你的本地模块依赖某个第三方模块先完成初始化，它在 `nuxt.config` 的 `modules` 数组里时顺序是安全的，如果你把它放进 `modules/` 目录，它就退到后面去了。

官方接着说，你可以通过在每个目录名前面加一个数字来改变本地模块的顺序。

```bash [Directory structure]
modules/
  1.first-module/
    index.ts
  2.second-module.ts
```

注意官方这里给的编号是 `1.`、`2.`，没有补零。但 060 与 061 两处讲字母序编号时都同一条 note，文件名按字符串排序，`10.` 会排在 `2.` 之前，官方给个位数加 `0` 正是为此。这一节自己没重复那条 note，本地模块一旦过十个，我会照 061 的建议补零，这是我的做法。

三种模块的位置，别再混。把这一串读过的篇目收成一张表。

`nuxt.config` 的 `modules` 数组里放的是已安装的第三方模块，045 装的，注册方式是你在配置里点名。`modules/`，本项目，放的是本地模块，自动注册，两种模式，顺序在数组之后。`layers/` 放的是图层，可复用的代码与配置整包，用 `extends` 或目录自动注册，050 说的这个目录里的图层会被自动注册。`app/plugins/` 放的是插件，自动注册，运行在应用创建时，061 讲过。

009 那篇我讲过模块、插件、图层各是什么，这张表是它在目录层面的落地。

## 自查

- [ ] `modules/hello/main.ts` 会被自动注册吗？官方认的两种模式是什么？
- [ ] 本地模块要不要写进 `nuxt.config` 的 `modules` 数组？
- [ ] 写本地模块时，`@nuxt/kit` 要不要装进依赖？导入源该写什么？
- [ ] 模块里的组件该放在哪个路径下？官方给的理由是什么（运行还是类型）？
- [ ] 你的本地模块与 `nuxt.config` 里声明的模块，谁先执行？
- [ ] `addComponentsDir` 里 `pathPrefix: true` 那行注释说的目的是什么？

## 下一篇

068 讲 `layers/`：官方给的优先级顺序是什么、为什么每一层都必须有自己的 `nuxt.config.ts`，以及「未在 `extends` 里点名的层会降到哪里」。
