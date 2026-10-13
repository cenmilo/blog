---
url: /posts/9d3e1c11879a48a8e03d543e9b0df182/
title: 565 - server 节只有一个键：服务端构建器与两种简写
date: 2027-04-18T10:00:00+08:00
updated: 2027-04-18T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：`server` 这一节的定性是 **"Configuration for Nuxt's server builder."**，下面**只有 `builder` 一个子项**：类型 `string | { bundle: (nuxt: Nuxt) => Promise<void> }`，默认 **`"@nuxt/nitro-server"`**，说明句是 "By default, Nuxt uses `@nuxt/nitro-server`, which provides standalone Nitro integration."。两个简写 **`"nitro"` 与 `"vite"`** 分别对应 `@nuxt/nitro-server` 与 `@nuxt/vite-server`，而后者被官方写成"实验性、只用 Vite 实现"，并带一串后果：**`ssr: false` 时产出静态 SPA；开 SSR 时构建出的服务端入口导出 web 标准的 `{ fetch }`（`.output/server/index.mjs` 在其前面服务静态输出，`nuxt preview` 跑它）；所有依赖被打进产物，因此部署不需要 `node_modules`；`nuxt generate` 在构建末尾爬这个服务器写出静态 `.output/public`；但它不附带服务端运行时——所以服务端路由、除 `prerender` 以外的路由规则、内联样式与服务端组件都不可用，需要服务端的模块也不工作。** 整节还挂了一条 `::callout` 警告：这是内部用途、API 未定型。

categories:
  - 前端开发

tags:
  - nuxt4
  - 服务端构建器
  - nitro
  - vite
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

客户端构建器可以换成 webpack 或 rspack，那服务端呢？

有一节管这件事，但它自己先声明"这不是给你用的"。

## server.builder 的记载

`## server` 一句定性：**"Configuration for Nuxt's server builder."**；其下只有 `### builder`：

> Specify the server builder to use for bundling the server part of your application.
>
> By default, Nuxt uses `@nuxt/nitro-server`, which provides standalone Nitro integration. This architecture allows for different Nitro integration patterns, such as using Nitro as a Vite plugin (with the Vite Environment API).

- **类型**：`string | { bundle: (nuxt: Nuxt) => Promise<void> }`
- **默认**：`"@nuxt/nitro-server"`

然后是一整段关于 `"vite"` 简写的说明，这段是这一节真正的信息量所在：

> `"nitro"` and `"vite"` are shorthands for `@nuxt/nitro-server` and `@nuxt/vite-server`. The latter is experimental and implemented with Vite alone: with `ssr: false` it produces a static SPA, and with SSR enabled it builds a server from the Nuxt SSR renderer whose entry exports a web-standard `{ fetch }` (`.output/server/index.mjs` serves the static output in front of it, and `nuxt preview` runs it). Every dependency is bundled into that output, so it deploys without `node_modules`. `nuxt generate` crawls that server at the end of the build and writes a static `.output/public`. It ships no server runtime, so server routes, route rules other than `prerender`, inlined styles and server components are unavailable, and modules that need a server will not work.

## 把这段拆成七条

| # | 事实 | 为什么值得单独记 |
| --- | --- | --- |
| 1 | `"nitro"` ＝ `@nuxt/nitro-server`；`"vite"` ＝ `@nuxt/vite-server` | 简写与包名等价，写法可短 |
| 2 | 后者**是实验性的**、只用 Vite 实现 | 不是"另一种可选生产方案" |
| 3 | `ssr: false` 时它产出静态 SPA | 与 `ssr` 开关联动 |
| 4 | 开 SSR 时，服务端入口导出 web 标准的 `{ fetch }` | 形态与 Nitro 那套不同 |
| 5 | `.output/server/index.mjs` 在它前面服务静态输出，`nuxt preview` 跑的就是它 | 部署入口没换，换的是里面跑的东西 |
| 6 | 所有依赖被打进产物，因此**部署不需要 `node_modules`** | 与"standalone dist"是同方向的两条表述 |
| 7 | `nuxt generate` 在构建末尾爬这个服务器写 `.output/public` | 静态生成路径也变了 |

最后那句限制清单最该抄进规范：**它不附带服务端运行时，所以服务端路由、除 `prerender` 之外的路由规则、内联样式、服务端组件都不可用，需要服务端的模块也不工作。**

也就是说：**选了 `server.builder: 'vite'`，等于放弃 `server/` 目录那一层能力。** 这与 `nuxt/server` 那套导入（只能在被服务端构建器打包的代码里用）是同一件事的两端。

## 一条警告：这是内部用途

`### builder` 末尾挂着 `::callout`：

> This option is intended for internal use and the API is not finalized. Please open an issue before relying on the current implementation.

**官方在这里要的不是"你用它"，而是"你别依赖它"**——并给了"要依赖请先开 issue"的路径。与客户端那个 `builder` 的待遇完全不同：`## builder` 那节大大方方给了三种写法、并说明换 webpack/rspack 要自己装包；`server.builder` 则是内部键。

这也解释了配置分布：**`nitro` 一节（Nitro 的配置入口）是官方承认的透传面，而"换掉整个服务端构建器"不是。**

## 三个容易混的 builder

| 键 | 管什么 | 官方态度 |
| --- | --- | --- |
| `builder` | 客户端（Vue 那一侧）构建器，默认 `@nuxt/vite-builder` | 正常支持，可换 webpack/rspack/自定义 |
| `server.builder` | 服务端构建器，默认 `@nuxt/nitro-server` | 内部用途、API 未定 |
| `typescript.builder` | 生成哪种构建器的**类型环境**（默认按 `builder` 推断，可 `false` 或 `'shared'`） | 正常配置项 |

`typescript.builder` 那节还有一句对模块作者的提示：**`'shared'` 是给要支持多种构建器的模块作者推荐的**。三者名字相同、层级完全不同，改之前先确认自己动的是哪一个。

## 自查

- `## server` 只有一个子项 `builder`，默认 `@nuxt/nitro-server`；`"nitro"`/`"vite"` 是两个简写，后者实验性且只用 Vite。
- 用 `vite` 服务端构建器时的形态：`ssr:false` 出静态 SPA；开 SSR 时入口导出 web 标准 `{ fetch}`；产物含全部依赖故不需要 `node_modules`；`nuxt generate` 在构建末爬它写 `.output/public`。
- **代价是官方明列的四类能力没了**：服务端路由、除 `prerender` 外的路由规则、内联样式、服务端组件（以及需要服务端的模块）。
- 该键被官方标为内部用途、API 未定型，要依赖请先开 issue。
- 与 `builder`（客户端）、`typescript.builder`（类型环境）是三件不同的事，名字相近。

## 下一篇

下一篇 566 读 `serverHandlers` 与 `devServerHandlers`：前者五个选项（`handler`/`route`/`method`/`middleware`/`lazy`）与那条"三个目录里的文件本来就自动注册"的说明，后者只有两句——"Nitro development-only server handlers"，没有示例也没有选项清单。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## server` 与 `### builder` 全部记载：定性句、默认 `@nuxt/nitro-server` 与 standalone Nitro integration／Vite Environment API 那句、`"nitro"`/`"vite"` 简写段全文、类型与默认值、`::callout` 内部用途警告；对照的 `## builder` 与 `### typescript.builder`）
- https://nuxt.com/docs/4.x/guide/going-further/server-imports（`nuxt/server` 只能用于被服务端构建器打包的代码，用于说明"没有服务端运行时"的影响面）
- https://nuxt.com/docs/4.x/guide/concepts/server-engine 与 https://nuxt.com/docs/4.x/api/commands/preview（Nitro 产出 standalone dist、`nuxt preview` 起的是构建后的服务器）
