---
url: /posts/168f373f77e6fc79122dc480a66abfd7/
title: 621 - Bridge 下的插件与中间件：一条内存泄漏警告与 definePageMeta 的两个键
date: 2027-05-07T09:00:00+08:00
updated: 2027-05-07T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：`bridge/plugins-and-middleware` 三个小节，各带一条硬性限制。插件侧：**新格式只接一个参数 `nuxtApp`**，而且**想在插件里用 `useNuxtApp`、`useRuntimeConfig` 这类新组合式函数，就必须用 `defineNuxtPlugin` 包那一层**；同节一条 `::warning` 点名风险——**虽然 `nuxtApp.vueApp` 提供了兼容接口，但用它注册插件、指令、mixins 或组件时，不加"确保不被重复安装"的逻辑就可能造成内存泄漏**。中间件侧：**`defineNuxtRouteMiddleware` 在 `app/middleware` 目录之外不被支持**（`::important`），中间件参数是 `(to, from)` 两个。`definePageMeta` 侧：要先用 `bridge.macros.pageMeta: true` 打开，而且**在 Bridge 下只支持 `middleware` 与 `layout` 两个键**。

categories:
  - 前端开发

tags:
  - nuxt4
  - bridge
  - 插件
  - 路由中间件
  - 升级迁移
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

Bridge 项目里最费解的一种现象是：本地跑得好好的，内存却一点点涨；或者某个中间件在某处就是不生效。这一节的两条警告正好各解释一种。

## 插件：换形状，也换注册方式

> You can now migrate to the Nuxt 3 plugins API, which is slightly different in format from Nuxt 2.
>
> Plugins now take only one argument (`nuxtApp`). You can find out more in the docs.

给出的形状是：

```ts [app/plugins/hello.ts]
export default defineNuxtPlugin((nuxtApp) => {
  nuxtApp.provide('injected', () => 'my injected function')
  // now available on `nuxtApp.$injected`
})
```

两点可取：**`nuxtApp.provide('injected', …)` 的结果挂在 `nuxtApp.$injected` 上**（注释原文如此）；**只接一个参数**，Nuxt 2 那套 `(ctx, inject)` 不再适用。

紧接一条 `::note` 是判断标准：

> If you want to use the new Nuxt composables (such as `useNuxtApp` or `useRuntimeConfig`) within your plugins, **you will need to use the `defineNuxtPlugin` helper for those plugins.**

也就是说：**想在插件里调新组合式函数，`defineNuxtPlugin` 这层包装不是风格问题，而是必要条件**——没包就拿不到上下文。

然后就是这一节最重的一条 `::warning`，原话要点是：**"Although a compatibility interface is provided via `nuxtApp.vueApp` you should avoid registering plugins, directives, mixins or components this way without adding your own logic to ensure they are not installed more than once, or this may cause a memory leak."**

拆开读是三件事：`nuxtApp.vueApp` 是**存在的**、它是**兼容通道**、**默认用法有风险**。重复安装是泄漏的成因，所以"要不要自己加一次性守卫"这件事，官方明确留给你。至于哪些注册必须走这条路，这一节没说。

## 中间件：两个参数，一条目录限制

> Middleware now take two argument (`to`, `from`). You can find out more in the docs.

示例只用了 `to`：

```ts [app/middleware/auth.ts]
export default defineNuxtRouteMiddleware((to) => {
  if (to.path !== '/') {
    return navigateTo('/')
  }
})
```

（官方这块代码没标文件名，上面这个路径只是示意——**它强调的限制恰恰与目录有关**。）

`::important` 就是这一节唯一硬性边界：

> Use of `defineNuxtRouteMiddleware` **is not supported outside of the `app/middleware` directory.**

这条能解释一种很具体的现象：**把中间件写在别处（例如随组件放的文件里、或从插件里引用一个非 `app/middleware` 下的文件）时它不生效**——不是注册问题，是**目录外的这种写法在 Bridge 下不被支持**。

与迁移文档那一节对照着记更清楚：**`defineNuxtRouteMiddleware` 是新版写法**、全局中间件在 Bridge 之后仍靠 `.global` 这类约定，而**引用它的方式**从组件选项变成了 `definePageMeta`。

## definePageMeta：先开宏，只有两个键可用

> You can also use `definePageMeta` in Nuxt Bridge.
>
> It can be enabled with the `macros.pageMeta` option in your configuration file

```ts [nuxt.config.ts]
import { defineNuxtConfig } from '@nuxt/bridge'

export default defineNuxtConfig({
  bridge: {
    macros: {
      pageMeta: true,
    },
  },
})
```

这一节最后一条 `::note` 把它收窄得很厉害：

> But only for `middleware` and `layout`.

**在 Bridge 阶段，`definePageMeta` 只有这两个键有效。** 所以不要指望 `key`、`transition`、`keepalive`、`validate` 这些在这个阶段就能用——那批要等到真正上 3 之后（`definePageMeta` 的完整键集在 API 那一节，而迁移文档里 `key`/`transition`/`keepalive` 各另有去处）。

`bridge.macros.pageMeta` 与 Bridge 配置页里那一项是同一个开关：**它属于"要显式打开"的那四个之一**，不开就用不了宏。

## 自查

- 插件：**只接 `nuxtApp` 一个参数**；`provide` 的结果在 `nuxtApp.$injected`；**要用新组合式函数就必须包 `defineNuxtPlugin`**。
- **`nuxtApp.vueApp` 是兼容通道，用它注册东西要自己保证不重复安装，否则可能内存泄漏**。
- 中间件：参数 `(to, from)`；**`defineNuxtRouteMiddleware` 只在 `app/middleware` 目录内被支持**。
- `definePageMeta` 需 `bridge.macros.pageMeta: true`，**且只支持 `middleware` 与 `layout` 两个键**。
- 想要 key/transition/keepalive/validate 那批宏键，得等真正迁到 3，不是 Bridge 阶段的事。

## 下一篇

下一篇 622 把 Bridge 剩下的两节并起来读：meta 那一节要求把配置里的 `head` 改名成 **`app.head`**（对象不再有 `hid` 去重键）、`useHead` 底层走的是 `@unhead/vue` 而不是 `vue-meta`、**不建议与 Nuxt 2 原生 `head()` 混用**，函数版 `titleTemplate` 则要挪到布局里；runtime config 那一节只有一条前置警告——**用 `runtimeConfig` 之前 `nitro` 必须已经配好**，然后把 `publicRuntimeConfig`/`privateRuntimeConfig` 合成一个 `runtimeConfig`，读的时候要走 `$config.public.xxx`。

## 本文依据

- https://nuxt.com/docs/4.x/bridge/plugins-and-middleware（`## New Plugins Format` 两句定性与示例（含 `nuxtApp.$injected` 注释）、`::note`（要用新 composables 必须用 `defineNuxtPlugin`）、`::warning`（`nuxtApp.vueApp` 与重复安装、内存泄漏）；`## New Middleware Format` 的参数句与示例代码块、`::important` 目录限制原句；`## definePageMeta` 两句、`macros.pageMeta` 配置示例与"but only for `middleware` and `layout`"的 `::note`）
- https://nuxt.com/docs/4.x/bridge/configuration（`macros: { pageMeta: true }` 属于四个 opt-in 项之一，用于对照本节开关）
- https://nuxt.com/docs/4.x/migration/plugins-and-middleware（Nuxt 3 侧的对照：插件只接 `nuxtApp`、`provide` 的两种写法、中间件用 `definePageMeta` 引用、全局中间件 `.global` 后缀）
