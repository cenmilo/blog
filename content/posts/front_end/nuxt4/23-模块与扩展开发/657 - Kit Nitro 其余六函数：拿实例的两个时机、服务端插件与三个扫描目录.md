---
url: /posts/29645b0536ce926faea1a2716ce5e7f4/
title: 657 - Kit Nitro 其余六函数：拿实例的两个时机、服务端插件与三个扫描目录
date: 2027-05-19T09:00:00+08:00
updated: 2027-05-19T09:00:00+08:00
author: cmdragon

summary:
  摘要：Nitro 那一节剩下六个函数分三组。**取实例**：`useNitro` **只能在 `ready` 钩子之后调用**，而且有一条 note 说**改动 Nitro 实例的配置不会生效**；`tryUseNitro` 返回 `undefined`，因为有两种情况确实没有实例——**`ready` 之前**，以及**配置的 `server.builder` 根本不用 Nitro（例如输出纯客户端 SPA 的 builder）**，那种情况下**服务端路由、路由规则与预渲染三样全都不存在**。**服务端插件** `addNitroPlugin` 在 v4.6 之前叫 `addServerPlugin`，并有一条警告要求**在插件文件里显式从 `nitropack/runtime` 导入 `defineNitroPlugin`**（`useRuntimeConfig` 这类助手同理）。**三个扫描类**：`addServerImports` 的八个字段与 `addImports` 逐字相同、`addServerImportsDir` 管自动导入目录、`addServerScanDir` 只按四个子目录注册（api、routes、middleware、utils）。

categories:
  - 前端开发

tags:
  - nuxt4
  - Nitro实例
  - 服务端插件
  - 服务端自动导入
  - NuxtKit
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上一篇讲了怎么注册处理程序，这一篇讲剩下六件：拿 Nitro 实例、注册服务端插件、以及让服务端的代码也能"不写 import 就用"。

## 六件事各归哪组

| 组 | 函数 |
| --- | --- |
| 取实例 | `useNitro`、`tryUseNitro` |
| 服务端插件 | `addNitroPlugin` |
| 服务端自动导入与扫描 | `addServerImports`、`addServerImportsDir`、`addServerScanDir` |

## `useNitro`：时机与一条"改了没用"

```ts
function useNitro (): Nitro
```

定性一句：**返回 Nitro 实例**。两条提示把它限制了很窄：

- `::warning`：**只能在 `ready` 钩子之后调用 `useNitro()`**。
- `::note`：**对 Nitro 实例配置的改动不会被应用**。

官方示例就是这两条的落地——**注册在 `ready` 里**：

```ts
import { defineNuxtModule, useNitro } from '@nuxt/kit'

export default defineNuxtModule({
  setup (options, nuxt) {
    nuxt.hook('ready', () => {
      const nitro = useNitro()
      // Do something with Nitro instance
    })
  },
})
```

第二条 note 的含义要说白：**拿到实例不等于能改它的配置**。要改配置得走 `nitro:config` 钩子那类时机（这一节没写，但把"改配置不生效"与"只能在 ready 后调用"并排放，就是在告诉读者这两条路不是同一条）。

## `tryUseNitro`：两种"根本没有实例"

```ts
function tryUseNitro (): Nitro | undefined
```

这一节写清了什么时候会是 `undefined`，一共两种情形：

> There is no Nitro instance before the `ready` hook has run, and none at all when the configured `server.builder` does not use Nitro, such as a builder that outputs a client-only SPA.

> Use this rather than `useNitro()` for anything that should keep working without a server: server routes, route rules and prerendering are all absent in that case.

三处要点：**`ready` 之前没有**、**`server.builder` 不用 Nitro 时永远没有（举的例子是输出纯客户端 SPA 的 builder）**、**这种情况下服务端路由、路由规则、预渲染三样都不存在**。

所以判据是：**你的模块在没有服务端时是否还应该正常工作**——是就用 `tryUseNitro` 并准备一条早退分支：

```ts
nuxt.hook('ready', () => {
  const nitro = tryUseNitro()
  if (!nitro) {
    // no server: skip anything that would only run there
    return
  }
})
```

这句注释（"跳过只在服务端跑的那部分"）就是它的全部用法。**它对应的是上一篇 `addPrerenderRoutes` 那类调用在纯客户端构建下根本无从生效**。

## `addNitroPlugin`：改名过，而且必须显式导入

定性：**添加插件以扩展 Nitro 的运行时行为**。一条 `::note` 是历史信息：**这个名字在 Nuxt v4.6 之前叫 `addServerPlugin`**——在旧模块源码里看到 `addServerPlugin` 就是它。

```ts
function addNitroPlugin (plugin: string | { nitro2?: string, nitro3?: string }): void
```

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `plugin` | `string \| { nitro2?: string, nitro3?: string }` | `true` | 插件路径；**为每个 Nitro 大版本各准备一份实现时，按 major 各给一个路径**。插件**必须导出一个默认函数，它接收 Nitro 实例作为参数** |

注意这里**只有 `nitro2` 与 `nitro3` 两个键，没有通用的 `nuxt` 那一份**——这与服务端兼容性那一章说的一致：**`nuxt/server` 不覆盖插件领域**。

最关键的是那条 `::warning`：

> It is necessary to explicitly import `defineNitroPlugin` from `nitropack/runtime` within your plugin file. The same requirement applies to utilities such as `useRuntimeConfig`.

**服务端插件文件里必须显式从 `nitropack/runtime` 导入 `defineNitroPlugin`，`useRuntimeConfig` 这类助手也一样**。这正是"发布模块的 runtime 不启用自动导入"那条规则的具体落点。

示例给出的三个钩子名可以直接当清单用：

```ts [module.ts]
import { addNitroPlugin, createResolver, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  setup () {
    const { resolve } = createResolver(import.meta.url)
    addNitroPlugin(resolve('./runtime/plugin.ts'))
  },
})
```

```ts [runtime/plugin.ts]
export default defineNitroPlugin((nitroApp) => {
  nitroApp.hooks.hook('request', (event) => {
    console.log('on request', event.path)
  })

  nitroApp.hooks.hook('beforeResponse', (event, { body }) => {
    console.log('on response', event.path, { body })
  })

  nitroApp.hooks.hook('afterResponse', (event, { body }) => {
    console.log('on after response', event.path, { body })
  })
})
```

**这一处文档自相矛盾**：警告说要显式导入 `defineNitroPlugin`，而这段示例里它**没有被导入**。实际写插件时按警告来（补上那行 import），别按示例抄。

## `addServerImports`：与 `addImports` 同表

```ts
function addServerImports (dirs: NuxtImport | NuxtImport[]): void
```

定性：**给服务端添加导入，让它们在 Nitro 里可用而不必手动 import**。

八个字段与自动导入那一节的 `addImports` **逐字相同**（`name`、`from` 必填，`priority`、`disabled`、`meta`、`type`、`typeFrom`、`as` 可选）。用法也一样：

```ts
const names = [
  'useStoryblok',
  'useStoryblokApi',
  'useStoryblokBridge',
  'renderRichText',
  'RichTextSchema',
]

names.forEach(name =>
  addServerImports({ name, as: name, from: '@storyblok/vue' }),
)
```

这一节有一条长度少见的 `::warning`，讲的是**跨上下文共享同一个函数时的三个要求**：

> If you want to provide a utility that works in both server and client contexts and is usable in the `shared/` directory, the function must be imported from the same source file for both `addImports` and `addServerImports` and should have identical signature.

> That source file should not import anything context-specific (i.e., Nitro context, Nuxt app context) or else it might cause errors during type-checking.

拆成三条：**同一个源文件**（两边都指向它，不能一个指实现、一个指 re-export）、**签名一致**、**那个源文件不能导入任何上下文相关的东西（Nitro 上下文或 Nuxt 应用上下文），否则类型检查会报错**。

顺带一处原文瑕疵：**签名的参数名是 `dirs`，而下面的参数说明写的是 `imports`**——同一个函数里两个称呼，照 `imports` 理解才对（它接的是导入条目，不是目录）。

## 两个"目录"函数的分工

```ts
function addServerImportsDir (dirs: string | string[], opts: { prepend?: boolean }): void
function addServerScanDir (dirs: string | string[], opts: { prepend?: boolean }): void
```

| 函数 | 定性与边界 |
| --- | --- |
| `addServerImportsDir` | **添加一个由 Nitro 扫描来做自动导入的目录**；`opts.prepend` 为 `true` 时把该目录**加到扫描列表的开头** |
| `addServerScanDir` | **添加由 Nitro 扫描的目录；它会检查子目录，并按 `~~/server` 那样注册**；带一条 `::note`：**只有 `~~/server/api`、`~~/server/routes`、`~~/server/middleware`、`~~/server/utils` 会被扫描** |

`addServerScanDir` 那条 note 是这一组里最容易被忽略的限制：**你注册的目录里只有这四个子目录名有意义**，放别的名词（例如 `runtime/server/composables`）不会被当路由或工具注册。

```ts
addServerImportsDir(resolve('./runtime/server/composables'))
```

```ts
addServerScanDir(resolve('./runtime/server'))
```

两个示例之后各接一段服务端代码，展示"不用 import 就能用"：**`runtime/server/api/hello.ts` 里直接调 `useApiSecret()`／`hello()`**，而它们的实现分别在 `runtime/server/composables/index.ts` 与 `runtime/server/utils/index.ts` 里导出。

```ts [runtime/server/composables/index.ts]
export function useApiSecret () {
  const { apiSecret } = useRuntimeConfig()
  return apiSecret
}
```

```ts [runtime/server/utils/index.ts]
export function hello () {
  return 'Hello from server utils!'
}
```

**注意 `useApiSecret` 里那个 `useRuntimeConfig()` 也没有 import**——与上面 `addNitroPlugin` 那条警告对照着读：**这些示例演示的是"自动导入生效之后"的样子，不是发布模块时该照抄的写法**。

还有一个签名与表格不一致的地方：两个函数的类型里 `opts` **没有 `?`（看起来是必填）**，而参数表把它标成 `Required: false`。

## 三个 addServer 的分工

| 你要的 | 用哪个 |
| --- | --- |
| 点名几个第三方导出，让服务端代码不用 import | `addServerImports` |
| 一个放工具/组合式函数的目录，整体进服务端自动导入 | `addServerImportsDir` |
| 一整套服务端结构（API 路由、中间件、utils），按 `~~/server` 的规则注册 | `addServerScanDir` |

这一节没写的还有：`useNitro` 之后哪些字段可读、`addServerScanDir` 与项目自己的 `server/` 目录同名路由冲突时谁赢、以及 `prepend` 对扫描顺序的实际影响（只有"加到扫描列表开头"一句，没有说这意味着什么优先级）。

下一篇讲路径解析那四个函数，先从那条"不查嵌套依赖"的限制说起。

## 下一篇

下一篇 658：Kit 路径解析四函数：resolvePath 的五选项与那条不查嵌套依赖

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/nitro
