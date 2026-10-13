---
url: /posts/f0d29139b9dce92c543bfbf1322e40d5/
title: 312 - shared 目录：一份代码两端可用的规则
date: 2027-01-24T09:00:00+08:00
updated: 2027-01-24T09:00:00+08:00
author: cmdragon

summary:
  摘要：有些纯函数（格式化、校验、常量、类型）前端和后端都要用，放 app/ 服务端拿不到、放 server/ 前端拿不到——这就是 shared/ 的位置。这一篇讲它的核心规则与限制：shared/ 里的代码两端都能用，但有一条硬约束"shared 里的代码不能 import 任何 Vue 或 Nitro 代码"，因为 Nuxt 把应用和服务端打成两个独立 bundle、跑在不同上下文；类型可以跨边界 import（是编译期擦除），但两端都要用的类型应放 shared/types。再讲扫描规则——只有 shared/utils 与 shared/types 顶层自动导入，其余文件用 #shared 别名手动 import。

categories:
  - 前端开发

tags:
  - nuxt4
  - shared
  - 自动导入
  - 两端共用
  - shared-types
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上一篇（311）把 app 与 server 的自动导入分界讲了，那条"不许互 import"的墙留了个问题：那**两端都要用的代码放哪**？答案是 `shared/`。这一篇讲它的规则和它的限制。

## shared 是干嘛的

官方定位：

> The `shared/` directory allows you to share code that can be used in both the Vue app and the Nitro server.

一个 `capitalize` 这种纯函数、一份 API 响应的类型定义，前端组件要用、后端 handler 也要用——放这里，两边都能拿。版本门槛：

> The `shared/` directory is available in Nuxt v3.14+.

## 一条硬约束：shared 里不许 import Vue 或 Nitro 代码

这条是 shared 的核心限制，官方放在 important：

> Code in the `shared/` directory cannot import any Vue or Nitro code.

为什么？官方 "Why You Cannot Mix Vue and Nitro Code" 给了原理：

> Nuxt builds two separate bundles: the Vue app (client and server-side rendering) and the Nitro server (API routes, server middleware, server plugins). They are bundled independently and run in different contexts. Code in the `shared/` directory is used in both bundles, so it cannot import from either.

拆成人话：Nuxt 出两个各自独立打包、跑在不同环境的 bundle——Vue 应用、Nitro 服务端。shared 的代码**同时进这两个 bundle**，所以它不能依赖其中任何一个的运行时：

- **不能在 shared 里 import Vue 组件/组合式**（它们要 Vue 运行时、要 Nuxt 上下文 `useNuxtApp()`/`useRoute()`，Nitro 里根本没有）。官方警告这样 import 会"build 或运行时报错、还可能把 Vue 依赖拖进服务端包"。
- **不能在 shared 里 import 服务端专用代码**（Node API、Nitro 工具、server handler）。官方警告这样"会弄坏客户端构建、在浏览器里运行时报错、或把服务端逻辑泄露进客户端包"。

所以 shared 里放的应该是**环境无关的东西**：纯函数、常量、类型。这条限制和 311 那道墙是同一个原理的两面。

## 类型是例外：import type 可以跨

跨边界有个特例，官方 "Type-Only Imports"：

> `import type` is erased at compile time and does not pull runtime code into the other bundle, so importing only types across the boundary may appear to work. Even so, keep shared types (such as API response types) in `shared/types/`, where they are auto-imported in both contexts.

也就是 `import type` 编译期被擦掉、不带运行时，跨边界 import 类型"看起来能跑"；但官方**仍建议把两端都要用的类型放 `shared/types/`**（两边都自动导入），而不是零散地 `import type` 跨来跨去——这样边界清晰、也避免日后不小心把类型 import 变成值 import。类型上下文按用途放：

- `app/types/` → 只 Vue 应用侧自动导入。
- `server/types/` → 只服务端自动导入（301）。
- 两端都要 → `shared/types/`。

## 用法与扫描规则

shared 里的工具支持具名导出或默认导出两种：

```ts [shared/utils/capitalize.ts]
export const capitalize = (input: string) => {
  return input[0] ? input[0].toUpperCase() + input.slice(1) : ''
}
```

之后 app 和 server 都能直接用：

```vue [app/app.vue]
<script setup lang="ts">
const hello = capitalize('hello')
</script>
```

```ts [server/api/hello.get.ts]
export default defineEventHandler((event) => {
  return { hello: capitalize('hello') }
})
```

扫描规则是关键，官方给了带注释的结构图：

```bash [Directory Structure]
-| shared/
---| capitalize.ts        # Not auto-imported
---| formatters
-----| lower.ts           # Not auto-imported
---| utils/
-----| lower.ts           # Auto-imported
-----| formatters
-------| upper.ts         # Not auto-imported
---| types/
-----| bar.ts             # Auto-imported
```

> Only files in the `shared/utils/` and `shared/types/` directories will be auto-imported. Files nested within subdirectories of these directories will not be auto-imported unless you add these directories to `imports.dirs` and `nitro.imports.dirs`.

抓两点：

- **只有 `shared/utils/` 和 `shared/types/` 里的（顶层）文件自动导入**；shared 根下的、子目录里的、以及这两个目录再往下的嵌套文件，都**不**自动导入。
- 要让嵌套子目录也自动导入，得**同时**把它们加进 `imports.dirs`（app 侧）和 `nitro.imports.dirs`（server 侧）——两边都得配，因为两端各读一套配置。

官方还点明：这套扫描方式和 `app/composables/`、`app/utils/` 完全一致（"identical to"）。

## 没自动导入的：用 #shared 别名手动 import

不落在自动导入范围内的 shared 文件，官方给了 `#shared` 别名手动引：

```ts
import capitalize from '#shared/capitalize'
import lower from '#shared/formatters/lower'
import upper from '#shared/utils/formatters/upper'
```

`#shared` 由 Nuxt 自动配好，保证不管你从哪个文件引、路径都一致。

## 自查

- 两端都要用的**纯函数/常量/类型**放 shared；别在 shared 里 import Vue 组件/组合式，也别 import 服务端专用代码（会构建失败或拖坏另一个 bundle）。
- 两端都要用的**类型**放 `shared/types/`（两边自动导入），别零散 `import type` 跨边界。
- 只有 `shared/utils/`、`shared/types/` 顶层自动导入；嵌套要同时配 `imports.dirs` 与 `nitro.imports.dirs`；其余用 `#shared` 别名手动引。
- v3.14+ 才有 shared/。

## 下一篇

300-339 服务端引擎与 API 层继续。下一篇 313 讲服务端配置的下一环（见索引），编号与日期连续推进。

## 本文依据

- shared 目录（`directory-structure/shared`）：定位句 "The `shared/` directory allows you to share code that can be used in both the Vue app and the Nitro server."；note v3.14+；important "Code in the `shared/` directory cannot import any Vue or Nitro code."；Why You Cannot Mix（"Nuxt builds two separate bundles ... bundled independently and run in different contexts ... cannot import from either"、Vue App Code in Nitro 与 Nitro Code in the Vue App 两段后果、Type-Only Imports "import type is erased at compile time" 及建议放 shared/types）；类型上下文归属（app/types 仅 app、server/types 仅 server、shared/types 两端）；Usage 具名/默认导出与 app/server 使用示例；How Files Are Scanned（"Only files in the `shared/utils/` and `shared/types/` directories will be auto-imported"、嵌套需 `imports.dirs`+`nitro.imports.dirs`、"identical to app/composables and app/utils"、目录扫描注释图）；`#shared` 别名三条 import 示例。
