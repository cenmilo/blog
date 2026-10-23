---
url: /posts/146b5ac6fa1e2dbf2bfd799dcf5a182c/
title: 086 - serverDir 与 server 节：服务端目录归哪个基准、改名会牵动什么
date: 2026-11-09T11:00:00+08:00
updated: 2026-11-09T11:00:00+08:00
author: cmdragon

summary:
  摘要：服务端目录由顶层配置项 serverDir 决定，官方给了三句话——它存放 Nitro 的 routes、middleware 和 plugins；相对路径以 rootDir 为基准而不是 srcDir；类型 string、默认值 /<rootDir>/server。这带来一个新手最容易踩空的地方：dir 那一节的九个键（app、assets、layouts、middleware、modules、pages、plugins、public、shared）默认全是 app/... 或根下的目录，压根没有 server，所以你只改 srcDir 时 server/ 并不会跟着挪，官方 server 页面里也因此一律把它写成 ~~(这里指 rootDir)/server/api/hello.ts。改名会牵动五样东西：四类自动注册的文件、/api 前缀、中间件"不得返回或响应"的限制、只能服务端内用的 #server 别名，以及 server/types 的自动导入和 tsconfig 里覆盖 server 文件夹的 project reference。而 server 这一节只有一个子项 builder：默认 @nuxt/nitro-server（提供独立的 Nitro 集成），"nitro" 与 "vite" 是 @nuxt/nitro-server 和 @nuxt/vite-server 的简写，后者仍是实验性的纯 Vite 实现，且"不带任何服务端运行时"，server routes、除 prerender 外的 route rules、内联样式与 server components 都不可用，官方还明确警告这个选项供内部使用、API 未定稿。

categories:
  - 前端开发

tags:
  - nuxt4
  - serverdir
  - nitro
  - 构建器
  - 目录结构

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

前面讲 `server/` 里该放什么。这一篇讲它归谁管：目录名能不能改、改完会牵连到哪几处，以及 `nuxt.config` 里那个只有一格的 `server` 节到底能做什么。

`serverDir` 官方就三句话。

| 官方原话的要点 | 内容 |
| --- | --- |
| 它是什么 | 定义应用的服务端目录，Nitro 的 routes、middleware 和 plugins 放在这里 |
| 相对谁 | 如果给的是相对路径，它相对于你的 `rootDir` |
| 类型与默认 | `string`，默认 `/<rootDir>/server` |

就这些。没有别名、没有第二个候选位置、也没有跟随 srcDir 的说法。

最容易被忽略的差别：app 侧认 srcDir，server 侧认 rootDir。`dir` 这一节的九个键，默认值长这样。

| 键 | 默认值 | 说明 |
| --- | --- | --- |
| `dir.app` | `"app"` | 应用目录 |
| `dir.assets` | `"app/assets"` | 构建里别名为 `~assets` |
| `dir.layouts` | `"app/layouts"` | 每个文件自动注册为布局 |
| `dir.middleware` | `"app/middleware"` | 每个文件自动注册为中间件 |
| `dir.modules` | `"modules"` | 每个文件自动注册为模块 |
| `dir.pages` | `"app/pages"` | 用来自动生成页面路由 |
| `dir.plugins` | `"app/plugins"` | 每个文件自动注册为插件 |
| `dir.public` | `"public"` | 静态文件目录，生成时直接拷进 dist |
| `dir.shared` | `"shared"` | 应用与服务端共用的目录 |

这九格里没有 server。服务端目录是独立的顶层项 `serverDir`，基准是 `rootDir`。

这就是为什么官方写文档时，app 侧的路径是 `app/pages/index.vue`，而 server 侧全写成 `~~/server/api/hello.ts`，`~~` 指的就是 `rootDir`。

它带来的实际后果很直白。

```ts
export default defineNuxtConfig({
  srcDir: 'sources',
})
```

这样写之后，页面在 `sources/pages/`，但服务端代码仍然在项目根的 `server/` 下，一点没动。想让它们一起搬走，必须两个都写。

```ts
export default defineNuxtConfig({
  srcDir: 'sources',
  serverDir: 'sources/server',
})
```

改名会牵动五样东西。把服务端目录从 `server/` 改成别的名字，不是换个文件夹这么简单。下面每一项都是官方明确挂在这个目录上的。

一，四类自动注册的文件。

```bash
-| server/
---| api/
-----| hello.ts      # /api/hello
---| routes/
-----| bonjour.ts    # /bonjour
---| middleware/
-----| log.ts        # log all requests
```

`api/`、`routes/`、`middleware/`、`plugins/` 里的文件由 Nuxt 自动扫描注册并带 HMR，另外 `utils/` 用来放自己的助手函数。这些子目录全在 `serverDir` 之下，改名后一个都不能少地跟过去。

二，`/api` 前缀。`~~/server/api` 里的文件，路由会自动加上 `/api` 前缀，不想带前缀的路由放进 `~~/server/routes`。改名不改这条规则，但改完最好真访问一次确认前缀还在。

三，中间件那条限制。`~~/server/middleware` 会被自动读成服务端中间件，它在任何其他服务端路由之前对每个请求运行，用来添加或检查请求头、记录请求、扩展事件对象。官方对它的要求是硬性的：中间件处理器不该返回任何东西，也不关闭或响应请求，只能检查或扩展请求上下文，或者抛错。

```ts [server/middleware/auth.ts]
export default defineEventHandler((event) => {
  event.context.auth = { user: 123 }
})
```

四，`#server` 别名。这是 v4.3 加上的：可以用 `#server` 导入 `server/` 目录里的任何文件，与发起导入的文件位置无关。

```ts
// Instead of relative paths like this:
// import { formatUser } from '../../../utils/formatUser'

// Use the #server alias:
import { formatUser } from '#server/utils/formatUser'
```

并且官方注明 `#server` 只能在 `server/` 目录内使用，在客户端代码里从 `#server` 导入会直接报错。改了目录名之后，别名能不能照样解析、指向新目录还是旧目录，官方没有逐项承诺，动完先在一处真实文件里 import 一次，别靠猜。

五，类型。`~~/server/types/` 里的类型只在服务端上下文自动导入，可以在服务端路由、中间件、插件和工具里直接引用而不必 import，Vue 应用也需要的那些类型应该放进 `shared/types/`。另外 Nuxt 4 默认生成的 `tsconfig.json` 里，有一条覆盖 `server/` 文件夹的 project reference 来保证类型准确。目录改名后，这条引用和 `server/types` 的自动导入是最容易悄悄失效的两处。

`server` 节里只有一个键：`builder`。官方给这节的定义是配置 Nuxt 的服务端构建器，唯一的子项 `builder` 用来说明打包应用服务端部分时用哪个构建器。

默认值 `@nuxt/nitro-server`，官方说明它提供独立的 Nitro 集成，而正是这套架构让不同的 Nitro 集成方式成为可能，比如把 Nitro 当作 Vite 插件来用（配合 Vite Environment API）。

`"nitro"` 和 `"vite"` 是 `@nuxt/nitro-server` 与 `@nuxt/vite-server` 的简写。后者的官方描述信息量很大，逐条拆开。

| 官方结论 | 含义 |
| --- | --- |
| 实验性，且只用 Vite 实现 | 不是稳定接口 |
| `ssr: false` 时 | 产出静态 SPA |
| 开启 SSR 时 | 从 Nuxt 的 SSR 渲染器构建出一个 server，其入口导出 web 标准的 `{ fetch }` |
| `.output/server/index.mjs` | 在这个 fetch 前面服务静态产物，`nuxt preview` 运行的就是它 |
| 依赖全部打进产物 | 部署时不需要 `node_modules` |
| `nuxt generate` | 在构建末尾爬这个 server，写出静态的 `.output/public` |
| 它不携带任何服务端运行时 | 于是 server routes 不可用，除 `prerender` 之外的 route rules 不可用，内联样式不可用，server components 不可用，需要服务端的模块不能工作 |

类型：`string | { bundle: (nuxt: Nuxt) => Promise<void> }`。默认：`"@nuxt/nitro-server"`。

官方在这一项下面挂了一条警告：这个选项是为内部使用准备的，API 尚未定稿，在依赖当前实现之前请先开 issue。

两个 builder 不是一回事。这是最容易写错的一处，同一个词，位置不同，管的东西完全不同。

| 你写的键 | 官方定义 | 默认值 | 能换成什么 |
| --- | --- | --- | --- |
| `builder` | 打包应用 Vue 那一侧 的构建器 | `"@nuxt/vite-builder"` | `vite` / `webpack` / `rspack` / 自定义（下一篇） |
| `server.builder` | 打包应用服务端那一侧 的构建器 | `"@nuxt/nitro-server"` | `"nitro"` / `"vite"`（实验性） |

到底该不该动这两个。

| 你想做的事 | 该动哪个 | 结论 |
| --- | --- | --- |
| 服务端代码换个目录名 | `serverDir` | 可以，但要同时核对五处牵动 |
| 只把 srcDir 改掉，希望 server 跟着挪 | 没有这种联动 | 必须显式再写 `serverDir` |
| 想要更轻的纯静态产物 | `server.builder: 'vite'` | 实验性 + 官方声明接口未定稿，别上生产 |
| 需要 server routes / route rules / server components | `server.builder` | 只能用默认的 nitro 那条路 |

对绝大多数项目，`serverDir` 保持默认、`server.builder` 一行都不写，才是官方在 `dir` 那节给出的建议姿态，最好坚持默认值，除非确实需要。

## 自查

- [ ] `serverDir` 的相对路径基准是 `srcDir` 还是 `rootDir`？官方原话是哪一句？
- [ ] 为什么官方在文档里把服务端路径写成 `~~/server/api/hello.ts`，而页面写成 `app/pages/`？
- [ ] 服务端目录改名后，你会去核对哪五处？
- [ ] `server.builder` 设成 `"vite"` 时，官方明说哪几样东西不可用？
- [ ] `builder` 与 `server.builder` 的默认值分别是什么？

## 下一篇

087 讲 `buildDir`：`.nuxt` 这个名字为什么被那么多工具假定成隐藏目录、换成不带点的位置会牵动什么，以及 `nuxt cleanup` 与它的关系。
