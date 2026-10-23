---
url: /posts/dc69c3b6017dcf62740be853483cb40f/
title: 299 - 自己定义钩子：Adding Custom Hooks 的完整写法
date: 2027-01-19T11:00:00+08:00
updated: 2027-01-19T11:00:00+08:00
author: cmdragon

summary:
  摘要：284 用自定义事件讲过 hook/callHook 那套机制，这一篇专补"给自定义钩子名做类型声明"这一件事。官方 Adding Custom Hooks 一节给的就是：按钩子属于哪一类，去 declare module 扩展三个不同接口——应用运行时钩子扩 RuntimeNuxtHooks、构建期钩子扩 NuxtHooks（这俩都在 #app）、Nitro 服务端钩子扩 NitroRuntimeHooks（在 nitropack/types），返回值类型用 HookResult。这一篇把三个接口分别对应哪类钩子写清，再补一条 289 讲过的放置规则：增补文件要落在类型能被扫到的上下文目录里。

categories:
  - 前端开发

tags:
  - nuxt4
  - 自定义钩子
  - 类型增补
  - RuntimeNuxtHooks
  - HookResult
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


284 里你已经会用 `hook('app:user:registered', cb)` 和 `callHook('app:user:registered', payload)` 造自己的事件了——名字是自由字符串，机制上没问题。但 TypeScript 不认识这个名字，你在 `hook('...')` 处没有补全、也没有签名检查。这一篇就是给这些自定义名字**补类型**，官方的 "Adding Custom Hooks" 一节讲的就是这件事。

## 官方给的一段：三个接口

官方原话与完整代码块：

> You can define your own custom hooks support by extending Nuxt's hook interfaces.

```ts
import type { HookResult } from '@nuxt/schema'

declare module '#app' {
  interface RuntimeNuxtHooks {
    'your-nuxt-runtime-hook': () => HookResult
  }
  interface NuxtHooks {
    'your-nuxt-hook': () => HookResult
  }
}

declare module 'nitropack/types' {
  interface NitroRuntimeHooks {
    'your-nitro-hook': () => void
  }
}
```

关键就一句话：**扩展哪个接口，取决于你的钩子是哪一类**。这一段一口气给了三个，对应 294/297/298 讲过的三组。

## 三个接口分别归哪一类

把 284 那句"你起的事件名要类型"落到具体位置，对照三组钩子（294 的时间线）：

| 你定义的钩子 | 扩哪个接口 | 它在哪个模块下 | 对应钩子类别 |
| --- | --- | --- | --- |
| 应用运行时（插件/组件 `nuxtApp.hook` / `useRuntimeHook`） | `RuntimeNuxtHooks` | `#app` | App Hooks (Runtime) |
| 构建期（模块/`nuxt.config` `nuxt.hook`） | `NuxtHooks` | `#app` | Nuxt Hooks (Build Time) |
| 服务端运行时（Nitro 插件 `nitroApp.hooks.hook`） | `NitroRuntimeHooks` | `nitropack/types` | Server Hooks (Runtime) |

对照本篇示例代码就是：

- `'your-nuxt-runtime-hook'` 进了 `RuntimeNuxtHooks` → 它是应用运行时钩子，`nuxtApp.hook('your-nuxt-runtime-hook', ...)`、`useRuntimeHook('your-nuxt-runtime-hook', ...)` 会有补全。
- `'your-nuxt-hook'` 进了 `NuxtHooks` → 构建期钩子，`nuxt.hook('your-nuxt-hook', ...)` 有补全。
- `'your-nitro-hook'` 进了 `nitropack/types` 的 `NitroRuntimeHooks` → 服务端钩子，`nitroApp.hooks.hook(...)` 有补全。

284 里那个 `'app:user:registered'` 是在 `nuxtApp.hook`/`callHook` 上用的，所以它要补类型应归 `RuntimeNuxtHooks`（第一行那种）。

## 返回值类型：HookResult

注意前两个例子签名里的返回是 `HookResult`：

> import type { HookResult } from '@nuxt/schema'

`HookResult` 是构建期/运行时钩子回调的统一返回类型。Nitro 那个例子里官方写的是 `() => void`——不同类别的官方示例返回标注不同（Nitro 用 void，Nuxt 两类用 HookResult），照着对应类别抄即可。

签名里的参数位官方示例都是空的（`() => ...`），你要传 payload 就在参数位补上，比如给 284 那个事件写：

```ts
declare module '#app' {
  interface RuntimeNuxtHooks {
    'app:user:registered': (payload: { id: number, name: string }) => HookResult
  }
}
```

（参数形状跟着你 callHook 传的 payload 定，284 强调过它是按引用传的。）

## 放置规则：增补文件要落对上下文目录

这一条和 289 讲类型化 provide 时完全一样，这里再钉一次，因为它是"声明写了却不生效"的头号原因。typescript 那页 "Augmenting Types with Project References" 的规则：

> if you want to augment types for the `app` context, the augmentation file should be placed in the `app/` directory.
> - For the `server` context, place the augmentation file in the `server/` directory.
> - For types that are shared between the app and server, place the file in the `shared/` directory.

落到本篇：

- `RuntimeNuxtHooks` / `NuxtHooks`（应用与构建侧）——增补 `.d.ts` 落在能被 app 上下文扫到的位置（如 `app/`）。
- `NitroRuntimeHooks`（服务端）——落 `server/`（或 `shared/` 若两端都要）。

还有个通用要求：**每个 augment 文件末尾要有 `export {}`**（289 讲过），否则它是全局脚本、`declare module` 变覆盖而不是扩展。类型只有在跑过 `nuxt dev`/`nuxt build`/`nuxt prepare` 重新生成后才刷新，刚写完不生效先重跑 prepare。

## 自查

- 自定义钩子名补类型：按类别选接口——应用运行时 `RuntimeNuxtHooks`、构建期 `NuxtHooks`、Nitro 服务端 `NitroRuntimeHooks`。
- 前两个 declare 在 `#app` 下，Nitro 那个在 `nitropack/types` 下；返回类型对齐官方示例（Nuxt 两类 `HookResult`、Nitro 那个 `void`）。
- 参数位照你 callHook 传的 payload 写；payload 按引用传（284）。
- 增补文件放对上下文目录（app 侧放 `app/`、服务端放 `server/`、共享放 `shared/`）且带 `export {}`；不生效先 `nuxt prepare`。

## 下一篇

270-299 插件与中间件到此收尾。下一篇 300 开 300-339 服务端引擎与 API 层：先讲 Nitro 是什么、它自带哪些能力（跨平台、serverless、API 路由、混合模式、独立产物）。
