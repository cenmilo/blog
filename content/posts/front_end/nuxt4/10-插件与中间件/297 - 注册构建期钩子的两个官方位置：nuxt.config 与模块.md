---
url: /posts/1e6c1e7b8b794e496db6801fe971b749/
title: 297 - 注册构建期钩子的两个官方位置：nuxt.config 与模块
date: 2027-01-19T09:00:00+08:00
updated: 2027-01-19T09:00:00+08:00
author: cmdragon

summary:
  摘要：294 分了构建期与运行时两类钩子，这一篇专讲构建期钩子怎么注册。官方给的位置就两个：nuxt.config.ts 里挂一个顶层 hooks 节（对象，键是钩子名、值是回调），以及模块 setup 里调 nuxt.hook(name, cb)。两者订的是同一批 Nuxt Hooks (Build Time) 名字（ready、close、pages:extend、nitro:config、build:done 这些）。这一篇把两个位置的示例、常见名字、以及"构建期钩子跑在 build/dev 阶段、主要给模块用"这条边界讲清，避免有人误把它当运行时钩子订。

categories:
  - 前端开发

tags:
  - nuxt4
  - 构建期钩子
  - nuxt.config
  - 模块
  - pages-extend
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

运行时钩子在插件/组件里订（275、283）。构建期钩子（294 说的那一类，`pages:extend`、`build:done` 这些）不在那一侧，官方给的是另外两个位置：`nuxt.config.ts` 里的 `hooks` 节，和模块 setup 里的 `nuxt.hook(...)`。这一篇把这两个入口讲全。

## 位置一：nuxt.config.ts 的 hooks 节

官方 hooks 那页 "Within `nuxt.config.ts`" 给的最小形状：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  hooks: {
    close: () => { },
  },
})
```

结构很简单：顶层一个 `hooks` 对象，键 = 构建期钩子名，值 = 回调。它订的就是 Nuxt Hooks (Build Time) 那批名字（下一节列）。这种方式适合"在自己的项目里挂一条构建期逻辑"，不写模块也能用。

## 位置二：模块 setup 里的 nuxt.hook(...)

写 Nuxt 模块时，官方给的是在 `setup` 里调 `nuxt.hook`：

```js
import { defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  setup (options, nuxt) {
    nuxt.hook('close', async () => { })
  },
})
```

两点：

- 这里 `nuxt.hook` 的第一个参数就是钩子名（和 `nuxt.config` 里 `hooks` 的键同名同一批）。
- 模块 setup 拿到的是 `nuxt`（构建期的 Nuxt 实例），不是 `nuxtApp`（运行时实例，295）。别把两者混：`nuxt.hook(...)` 订构建期，`nuxtApp.hook(...)` 订运行时。名字像，所属阶段完全不同。

## 订的是哪些名字

Nuxt Hooks (Build Time) 表里挑最常用的几个（都在 `api/advanced/hooks` 那页）：

| 钩子 | 参数 | 何时触发 |
| --- | --- | --- |
| `ready` | `nuxt` | Nuxt 初始化完成、可用时 |
| `close` | `nuxt` | Nuxt 实例优雅关闭时 |
| `pages:extend` | `pages` | 从文件系统扫完页面路由后 |
| `pages:resolved` | `pages` | 路由被补上元数据之后 |
| `build:before` / `build:done` | - | 打包前 / 打包后 |
| `nitro:config` | `nitroConfig` | 初始化 Nitro 前，可改它的配置 |
| `imports:dirs` / `components:dirs` | `dirs` | 扩展导入目录 / 组件扫描目录 |

前两行 `ready`/`close` 就是官方示例里用的名字。`pages:extend` 你在 293（构建期设中间件）见过、在 130 一带也提过——它是模块/配置改路由表的主入口。

> 提醒：这张表是节选。官方在 Nuxt Hooks 那节写的是 "Check the schema source code for all available hooks"，穷举要回 `packages/schema/src/types/hooks.ts`。

## 一条边界：构建期钩子跑在 build/dev，不在运行时

为什么普通应用代码很少订这些？因为它们的触发时机在"构建 / 开发热重载"这一段：`ready`（实例就绪）、`pages:extend`（扫路由）、`nitro:config`（装配服务端）……这些一次构建就过了，不会"每次用户请求"都跑。294 那条时间线里它在最前面那段。

所以判据还是那两个问题（294）：

- 你要介入的是**构建过程**（改路由表、改 Vite/Nitro 配置、加模板）→ 构建期钩子，位置就是本篇这两个（`nuxt.config` hooks 节 / 模块 `nuxt.hook`）。
- 你要介入的是**页面渲染/水合/导航**（`app:created`、`vue:error`、`page:start`）→ 运行时钩子，位置在插件/组件（275、283）。

把 `build:done` 想拿去"每次请求记一条日志"是不行的——构建期钩子根本不在请求路径上跑。

## 自查

- 注册构建期钩子就两个位置：`nuxt.config.ts` 的顶层 `hooks` 对象、或模块 setup 里的 `nuxt.hook(name, cb)`。
- 分清新老同名：`nuxt.hook`（构建期，订 `NuxtHooks`）≠ `nuxtApp.hook`（运行时，订 `RuntimeNuxtHooks`，275）。
- 构建期钩子跑在 build/dev，不在每次请求；要响应导航去订运行时钩子。
- 表是节选，全量查 schema 源文件。

## 下一篇

下一篇 298 把运行时那两组讲清分工：App Hooks（应用渲染生命周期，插件/组件订）与 Server Hooks（Nitro 服务端生命周期，服务端插件订），以及它们各在哪一侧跑。

## 本文依据

- Lifecycle Hooks 指南（`guide/going-further/hooks`）：Nuxt Hooks (Build Time) 一节（"These hooks are available for Nuxt modules and build context."、"Within `nuxt.config.ts`" 的 `hooks:{close:()=>{}}` 示例、"Within Nuxt Modules" 的 `defineNuxtModule` + `nuxt.hook('close', ...)` 示例、read-more 指向 api/advanced/hooks#nuxt-hooks-build-time）。
- Lifecycle Hooks API（`api/advanced/hooks`）：Nuxt Hooks (Build Time) 表若干行（`ready`/`close`/`pages:extend`/`pages:resolved`/`build:before`/`build:done`/`nitro:config`/`imports:dirs`/`components:dirs` 的参数与描述）；"Check the schema source code for all available hooks."
