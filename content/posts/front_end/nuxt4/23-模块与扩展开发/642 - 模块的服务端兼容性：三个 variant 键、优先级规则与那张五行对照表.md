---
url: /posts/20e144e461ca188bb6a32bd15cd22e9b/
title: 642 - 模块的服务端兼容性：三个 variant 键、优先级规则与那张五行对照表
date: 2027-05-14T09:00:00+08:00
updated: 2027-05-14T09:00:00+08:00
author: cmdragon

summary:
  摘要：模块里 `server` 那些文件要同时服务两代运行时——**Nuxt 3 与 4 用 Nitro v2（`nitropack`）配 h3 v1，Nuxt 5 默认 Nitro v3（`nitro`）配 h3 v2**——解法是把同一份逻辑注册成多个 variant：**键只有三个，`nuxt`、`nitro2`、`nitro3`**。选哪个由 Nuxt 决定，规则是硬的：**在 `nitropack` v2 宿主上 `nitro2` 永远优先于 `nuxt`，在 Nitro v3 上 `nitro3` 永远优先于 `nuxt`**，所以想验 `nuxt` 那份实现，得把 fixture 的 `server.builder` 设成非 Nitro 的（如 `@nuxt/vite-server`）。三条版本线要记住：**handler 的对象形态需要 `@nuxt/kit@^4.6` 进 `dependencies`**、**开发处理器路由以 `/**` 结尾在 v2 宿主需要 `^4.6.1`**、**要类型检查 `nuxt` 那份文件需要把 `nuxt@>=4.6` 装成 dev 依赖**。最后，`nuxt/server` 不覆盖插件、存储与缓存，**这三类只能按 `nitro2` 注册**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 服务端兼容
  - Nitro
  - h3
  - 模块作者
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模块的服务端代码是最容易被版本换代打断的部分：同一个 `defineEventHandler`，在两代 Nitro 里来自不同的包、给的 event 也不一样。这一节给的方案不是"二选一"，而是**同一件事注册两份实现，让框架按宿主挑**。

## 前提与一句豁免

页面开场的版本关系是三句：**Nuxt 3 与 4 面向 Nitro v2（`nitropack`），它用 `h3` v1**；**Nuxt 5 默认 Nitro v3（`nitro`），它用 `h3` v2**；此外还可以用**基于 Web API 的自定义服务端运行时，那种到处都兼容（`nuxt/server`）**。

一条 `::note` 先把范围缩小：**如果你的模块没有服务端代码，这里全部内容都不适用**。

推荐路径是 `nuxt/server`：它是**基于 Web API 编写无关服务端代码的导入面，随 Nuxt 4.6 及以上提供**，用它写的代码能跑在 `nitropack` v2 上、跑在 Nitro v3 上，也跑在其他 server builder（例如 `@nuxt/vite-server`）下。

## 三个 variant 键

| 键 | 这个文件导入什么 | 它运行在哪 |
| --- | --- | --- |
| `nuxt` | **只导入 `nuxt/server`** | 任何 server builder，从 Nuxt 4.6 起 |
| `nitro2` | `h3`、`nitropack/runtime`、`#imports` | `nitropack` v2 |
| `nitro3` | `nitro`、`nitro/h3`，可单独用也可与 `nuxt/server` 一起 | Nitro server builder，v3 |

注册时给 `handler` 传一个对象而不是字符串：

```ts [module.ts]
import { addServerHandler, createResolver, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  meta: { name: 'my-module' },
  setup () {
    const { resolve } = createResolver(import.meta.url)

    addServerHandler({
      route: '/api/my-module/status',
      handler: {
        nuxt: resolve('./runtime/server/status'),
        nitro2: resolve('./runtime/server/status.legacy'),
      },
    })
  },
})
```

两份文件的差别不只在导入面，**还在于凡是 h3 v1 的助手或行为与 `nuxt/server` 处理不同的地方**：

```ts [runtime/server/status.ts]
import { defineEventHandler, useRuntimeConfig } from 'nuxt/server'

export default defineEventHandler(() => ({
  version: useRuntimeConfig().public.myModule.version,
}))
```

```ts [runtime/server/status.legacy.ts]
// @ts-expect-error `#imports` is typed for the app, not the server build
import { useRuntimeConfig } from '#imports'
import { defineEventHandler } from 'h3'

export default defineEventHandler(event => ({
  version: useRuntimeConfig(event).public.myModule.version,
}))
```

对照着看这两份，最实际的差异是 **`useRuntimeConfig()` 在 `nuxt/server` 里不需要传 event，而 h3 v1 那份必须 `useRuntimeConfig(event)`**；legacy 文件还带了一条 `@ts-expect-error`，因为 `#imports` 是为 app 而不是服务端构建生成类型的。

**保留旧 handler、在旁边加一份可移植的、把两份都注册**，是页面给的做法——Nuxt 会注册应用能跑的那份实现：Nitro v3 与其他 builder 下用可移植文件，还在跑 `nitropack` v2 的宿主上用 v2 文件。注册了宿主跑不了的 handler 时，**它会被跳过并给出警告**。

`::tip` 划了一条边界：**只有确实需要 Nitro v3 独有 API 时才用 `nitro3`，用 `nuxt/server` 会让代码在将来更可移植**。

## 一条版本线：对象形态要 kit 在 dependencies 里

`::important` 这条很容易踩：**`handler` 的对象形态需要你的模块把 `@nuxt/kit@^4.6` 写进 `dependencies`**（module starter 就是这么配的）。

反面情形也写了：如果 `@nuxt/kit` 只是 `devDependency` 并且在构建时被留作 external，或者它是 `peerDependency`，**那么实际运行的是应用那侧的版本**，而**低于 4.6 的版本会在对象 `handler` 上失败**。

类型侧还有两条：要类型检查 `nuxt` 文件，把 **`nuxt@>=4.6` 装成 devDependency**；要检查 `nitro3` 文件，**还要装 `nitro`**。

## 优先级与测试陷阱

在 `nitropack` v2 宿主上，**Nuxt 更偏好 `nitro2` 那份，所以 `nuxt` 那份只有在你只注册了它时才会在那里跑**。想验 `nuxt` 那份，页面说的是**在 Nuxt 5 上测**。

两条补充：如果你的模块自己的别名在两个 Nitro 大版本上都能解析到正确代码，**可以给两个键注册同一个文件**（`{ nitro2: file, nitro3: file }`），或者去设 `meta.compatibility.server`。**在 Nuxt 5 的 playground 与测试 fixture 里，h3 自动导入的 `defineEventHandler` 是不可用的，要从 `nuxt/server` 导入**。

## 开发处理器：同样的键，但跑在 Nuxt 进程里

`addDevServerHandler` 接受同样的键，**差别是每个 variant 是你模块代码里的一个函数，运行在 Nuxt 进程而不是服务端构建里**。因为 `nuxt/server` 只用于服务端代码，**不要从模块代码导入它，即使懒加载也不要**。

正确写法是**对着 Web 的 `Request` 与 `Response` 写逻辑，从 `nuxt.options` 读配置，再各自适配到两边的 API**：

```ts [module.ts]
import { addDevServerHandler, defineNuxtModule } from '@nuxt/kit'
import { defineEventHandler, toWebRequest } from 'h3'

export default defineNuxtModule({
  meta: { name: 'my-module' },
  setup (_options, nuxt) {
    const greeting = nuxt.options.runtimeConfig.public.myModule.greeting

    function greet (request: Request) {
      const name = new URL(request.url).searchParams.get('name')
      if (!name) {
        return new Response('Missing name', { status: 400 })
      }
      return Response.json({ message: `${greeting}, ${name}!` })
    }

    addDevServerHandler({
      route: '/_my-module/greet',
      handler: {
        nuxt: event => greet(event.req),
        nitro2: defineEventHandler(event => greet(toWebRequest(event))),
      },
    })
  },
})
```

**`nuxt` 那份收到的是 `RequestEvent`、读它的 `event.req`；`nitro2` 那份要用 `toWebRequest(event)` 把 h3 v1 的 event 转换**。因为模块导入了 h3 v1，页面要求**把 `h3@^1` 加进模块的 `dependencies`**。

### 路由匹配在两边不一样

这一段的差异最容易埋 bug：**在 `nitropack` v2 宿主上，每个 variant 都按路径前缀匹配路由（Nitro v2 就是这么做的），所以 `/_my-module` 也会匹配 `/_my-module/inter.woff2`；在 Nitro v3 上，`nuxt` 与 `nitro3` variant 是精确匹配**。

要让所有 variant 都覆盖某个路由下的子路径，**路由以 `/**` 结尾**：

```ts [module.ts]
addDevServerHandler({
  route: '/_my-module/assets/**',
  handler: {
    nuxt: event => serveAsset(event.req),
    nitro2: defineEventHandler(event => serveAsset(toWebRequest(event))),
  },
})
```

这时**它匹配 `/_my-module/assets` 以及其下的每个路径，所以你不必再注册一次基路径**；但在 `nitropack` v2 宿主上，这需要 **`@nuxt/kit@^4.6.1`** 进 `dependencies`。

还有一条读取路径的差异：**在 v2 宿主上 handler 读到的 `event.path` 是相对它自己路由基的**，因为 Nitro v2 会把基剥掉——请求 `/_my-module/assets/inter.woff2` 时 `event.path` 是 `/inter.woff2`。完整 URL 仍然能拿到：`toWebRequest()` 里有，`nuxt` 那份从 `event.req.url` 读。

## 工具函数、插件与其他指向服务端代码的地方

**暴露出去、接收 event 的工具函数**（通过别名、`addServerImports` 或类型模板提供的），参数要标成 `nuxt/server` 的 `RequestEvent`，或者标成它实际读的那一小块，例如 `Pick<RequestEvent, 'req' | 'context'>`。**标成 `H3Event` 的参数只接受某一个 h3 大版本的 event，于是这个工具函数没法从 `nuxt/server` 的 handler 里调用**。另外 `getNitroVersion` 报的是 **Nitro 的大版本，不是宿主在跑哪个 server API**——`server.builder` 不同的宿主一个 Nitro 的都不跑。

服务端插件也有 variant，但它们是 Nitro 专有的，**没有通用的 `nuxt` 那份可注册**；`::tip` 给了一个省事的角度：**`defineNitroPlugin` 是恒等函数，也许可以用一个类型注解替代它，从而写出 `nitro2` 与 `nitro3` 都能用的代码**。

```ts [module.ts]
addNitroPlugin({
  nitro2: resolve('./runtime/server/plugin.legacy'),
  nitro3: resolve('./runtime/server/plugin'),
})
```

其他一切指向服务端代码的地方（别名、类型模板、或读服务端 event 的 app 插件）用 `resolveServerVariant`，**它在宿主一个都跑不了时返回 `undefined`**；`addServerImports` 的 `from` 也接受同样的 variants：

```ts [module.ts]
const someServerFile = resolveServerVariant({
  nuxt: resolve('./runtime/server'),
  nitro2: resolve('./runtime/server.legacy'),
})

if (someServerFile) {
  nuxt.options.alias['#my-module/server'] = someServerFile
}

addServerImports({
  name: 'verifyToken',
  from: { nuxt: resolve('./runtime/verify'), nitro2: resolve('./runtime/verify.legacy') },
})
```

优先级在这里又说了一遍并且给了测法：**跑 `nitropack` v2 时 `nitro2` 恒优先于 `nuxt`，跑 `nitro` v3 时 `nitro3` 恒优先于 `nuxt`；所以要测 `nuxt` 那份，就把测试 fixture 的 `server.builder` 设成一个不是 Nitro 的 builder，例如 `@nuxt/vite-server`**。

## 声明那些推断不出来的

Nuxt 会**读注册文件的 import 来判断它用的是哪套 API**，所以通常不需要你声明。不够用时设 `meta.compatibility.server`：

```ts [module.ts]
export default defineNuxtModule({
  meta: {
    name: 'my-module',
    compatibility: { server: 'nuxt' },
  },
})
```

最后一句是关键：**判断不出来时，代码会被当作 `nitro2` 处理**。

## 类型与 runtimeConfig

要给模块加的 runtime config 键上类型，就 **augment `nuxt/schema` 里的 `RuntimeConfig`（或 `PublicRuntimeConfig`）**，这样键在 `nuxt`、`nitro2`、`nitro3` 三种文件里都有类型：

```ts [module.ts]
declare module 'nuxt/schema' {
  interface RuntimeConfig {
    myModule: { apiKey: string }
  }
}
```

builder 专有的类型——例如 `useRequestEvent()` 的 `H3Event`、`nitro:config` 钩子里那份完整的 Nitro 配置——**来自生成的 `.nuxt/nuxt.d.ts`**；没有它时会退化成与 builder 无关的类型（`RequestEvent` 和一份精简的 Nitro 配置）。**接受 event 的类型工具要按 `RequestEvent` 标，才能支持任何 server builder**。

还有一条范围限制：**建立在服务端插件、存储或缓存上的模块没有 `nuxt` variant，因为 `nuxt/server` 不覆盖这些领域**。做法是先按 `nitro2` 注册文件，移植时再补 `nitro3`。

## 哪些必须留在 Nitro 自己的导入上

`nuxt/server` 覆盖的是请求与响应相关的工作：处理器、错误、路由参数、读取与校验 query 和 body、响应头、CORS、cookie、会话、重定向、路由规则、runtime config、app config、用 `serverFetch()` 的进程内取数、用 `useServerHooks()` 的服务端运行时钩子。**Tasks 同样不被 `nuxt/server` 支持**。

需要别的领域时，就得待在 Nitro 自己的导入上，**并且用与该文件对应的那一版 specifier**——页面写得很直接：**导入 `nitro/*` 的文件会被读成 Nitro 3 代码，所以 `nitro2` 文件必须改导入 `h3` 与 `nitropack/*`**。

| 领域 | 在 `nitro2` 文件里 | 在 `nitro3` 文件里 |
| --- | --- | --- |
| 存储：`useStorage()` 与其驱动配置 | `nitropack/runtime` | `nitro/storage` |
| 缓存：`defineCachedEventHandler()`、`defineCachedFunction()` | `nitropack/runtime` | `nitro/cache` |
| 服务端插件 | 从 `nitropack/runtime` 取 `defineNitroPlugin()` | 从 `nitro` 取 `definePlugin()` |
| Nitro 自有的运行时钩子，例如 `request` 与 `error` | `useNitroApp().hooks`，来自 `nitropack/runtime` | `useNitroHooks()`，来自 `nitro/app` |
| 懒加载处理器：`lazyEventHandler()` | `h3` | `nitro/h3` |

最后关于 Nitro v3 的行为变化，这一节只给了一句：**如果你只发 `nitro2` 那份文件，它会通过 Nitro v2 兼容层继续在 Nuxt 5 上运行**，而那些随 Nitro v3 改掉的行为记在 Nuxt 5 的模块服务端兼容性指南里——**这一节没有列出它们**。

## 动手顺序

1. 先分清你的模块有没有服务端代码；没有就停在这里。
2. 有事件处理器的，保留现有实现、旁边加一份只导入 `nuxt/server` 的，**两份都注册**。
3. 确认 `dependencies` 里有 `@nuxt/kit@^4.6`（用到 `/**` 的开发路由要 `^4.6.1`），用到 h3 v1 适配时加 `h3@^1`。
4. 存储、缓存、插件、Tasks 这四类**别指望 `nuxt` variant**，按 `nitro2` 注册并计划移植。
5. 在 Nuxt 5 与一个非 Nitro 的 `server.builder` 下各跑一次，**否则你验的其实只是 `nitro2` 那份**。

下一篇把模块这块收口，进入实验特性与机制内幕。

## 下一篇

下一篇 643：Nuxt Kit 定位：模块专用、esm-only、装它要注意的两条版本线

## 本文依据

- https://nuxt.com/docs/4.x/guide/modules/server-compatibility
