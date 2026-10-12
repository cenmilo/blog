---
url: /posts/1ef8c79d7c59166b600f5bba4fa731ee/
title: server 目录全景：api、routes、middleware、plugins、utils
date: 2027-01-20T10:00:00+08:00
updated: 2027-01-20T10:00:00+08:00
author: cmdragon

summary:
  摘要：server/ 目录是 Nuxt 应用里所有服务端代码的家。这一篇把官方 server 那页的六个子目录一次铺开：api/（自动加 /api 前缀的接口）、routes/（不加前缀的服务端路由）、middleware/（每个请求都跑、不该返回的服务端中间件）、plugins/（注册成 Nitro 插件、订服务端生命周期）、utils/（自定义服务端 helper）、types/（只在服务端上下文自动导入的类型）。再点两条通用规则：每个 handler 文件 default 导出 defineEventHandler、以及 app 代码与 server 代码不许互相 import。

categories:
  - 前端开发

tags:
  - nuxt4
  - server目录
  - api路由
  - Nitro插件
  - h3
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上一篇（300）说服务端跑在 Nitro 上。这一篇落到目录：**你在项目里写的服务端代码，几乎全在 `server/` 这个文件夹下，按子目录自动分类注册。** 官方 server 那页给了这张结构图：

```bash [Directory Structure]
-| server/
---| api/
-----| hello.ts      # /api/hello
---| routes/
-----| bonjour.ts    # /bonjour
---| middleware/
-----| log.ts        # log all requests
```

三个示例目录只是开头，整页其实铺了六个各有分工的子目录。逐个讲。

## 通用规则：handler 怎么定义

所有路由/中间件文件都遵守同一条：默认导出一个用 `defineEventHandler()`（别名 `eventHandler()`）定义的函数；handler 可以直接返回 JSON 数据、返回 Promise，或用 `event.node.res.end()` 发响应。

> 一个必须记住的边界（server 页开头的 important）：
> Do not import Vue app code (components, composables, or other app-only utilities) in your server routes or utilities, and do not import server-only code in your app.

也就是 app 与 server 是两个运行环境，代码不许互相 import。这条在 288 从 `nuxt/server` 那侧也讲过，是同一道墙。

## 一、api/：自动带 /api 前缀的接口

> Files inside the `~~/server/api` are automatically prefixed with `/api` in their route.

`server/api/hello.ts` → 访问路径是 `/api/hello`。最小 handler：

```ts [server/api/hello.ts]
export default defineEventHandler((event) => {
  return {
    hello: 'world',
  }
})
```

返回的对象自动变成 JSON 响应（就是 300 说的 h3 那条"直接返回对象即 JSON"）。前端 `useFetch('/api/hello')` 就能拿。

## 二、routes/：不加前缀的服务端路由

> To add server routes without `/api` prefix, put them into `~~/server/routes` directory.

`server/routes/hello.ts` → 路径是 `/hello`（不带 `/api`）：

```ts [server/routes/hello.ts]
export default defineEventHandler(() => 'Hello World!')
```

api 和 routes 的唯一区别就是**要不要那层 `/api` 前缀**。官方还有一条 note 值得记：server routes 的动态路由能力**不如 pages 完整**（"server routes do not support the full functionality of dynamic routes as pages do"）。`/api` 前缀的来龙去脉下一篇（302）专门讲。

## 三、middleware/：每个请求都跑，且不该返回

291 已经细讲，这里只归位：

> Nuxt will automatically read in any file in the `~~/server/middleware` to create server middleware for your project.
> Middleware handlers will run on every request before any other server route ...

规则还是那条最关键的——**服务端中间件不该返回任何东西**，只检视请求、扩展 `event.context`、或抛错。示例 `log.ts`（记日志）、`auth.ts`（`event.context.auth = { user: 123 }`）。

## 四、plugins/：注册成 Nitro 插件

> Nuxt will automatically read any files in the `~~/server/plugins` directory and register them as Nitro plugins. This allows extending Nitro's runtime behavior and hooking into lifecycle events.

```ts [server/plugins/nitroPlugin.ts]
export default defineNitroPlugin((nitroApp) => {
  console.log('Nitro plugin', nitroApp)
})
```

注意包装函数是 `defineNitroPlugin`（不是 `defineNuxtPlugin`）——这层是服务端运行时插件，订的是 298 那张 Server Hooks 表（`render:html`、`request`、`error` 等）。它和 `app/plugins/`（Nuxt 应用插件）是两套系统，287 对齐过。

## 五、utils/：自定义服务端 helper

> You can add more helpers yourself inside the `~~/server/utils` directory.

server 路由本身由 h3 提供一批现成 helper（读 body、处理 cookie、重定向、读头等，300 列过）。官方例子演示了"包一层自定义 handler 工具"——`server/utils/handler.ts` 里写一个 `defineWrappedResponseHandler`，把原 handler 包起来、前后加操作，再在 `server/api/hello.get.ts` 里用它。属于"把重复的服务端逻辑抽成可复用函数"的位置。

## 六、types/：只在服务端自动导入的类型

> Types placed in `~~/server/types/` are auto-imported in the server context only, so you can reference them in server routes, middleware, plugins, and utilities without importing them.

```ts [server/types/todo.ts]
export interface Todo {
  id: string
  title: string
  completed: boolean
}
```

在 `server/api/todos.get.ts` 里能直接用 `Todo`，不用 import。两条边界记清：

- **只扫顶层**：`Only files directly in server/types/ are scanned; files in nested subdirectories are not auto-imported`（和 `shared/types/` 一样的规则）。
- **app 也要用的类型别放这**：要 Vue 应用侧也能引用，放 `shared/types/`（300 之前 066 讲过 shared）。

## 附赠：#server 别名（v4.3）

深嵌套时不想写 `../../../utils/...`，可以用 `#server` 指向 `server/` 目录：

```ts [server/api/users/[id]/profile.ts]
import { formatUser } from '#server/utils/formatUser'
```

边界 note：**`#server` 只能在 `server/` 目录内用，客户端代码里 import 它会报错。**

## 关于 tasks：这一版文档没有

有旧教程会让你建 `server/tasks/`。**但当前 4.x 的 server 那页没有 tasks 这一节**——它的子目录只到 api / routes / middleware / plugins / utils / types。别按旧结构写 `server/tasks`，那是 4.x 文档里没记载的东西。

## 自查

- 要接口带 `/api` 前缀放 `server/api/`、要干净路径（如 `/webhook`）放 `server/routes/`。
- `server/middleware/` 只加/查头、记日志、扩展 event.context，不返回值（291）。
- 要订服务端生命周期（改 HTML、响应）→ `server/plugins/` 的 `defineNitroPlugin`，别和 `app/plugins/` 混。
- 服务端类型放 `server/types/`（只扫顶层），app 也要用的放 `shared/types/`。
- 服务端代码别 import Vue app 代码，反之亦然；客户端别用 `#server`。
- 别凭空建 `server/tasks/`——4.x server 文档没有。

## 下一篇

下一篇 302 讲 `/api` 前缀这件事本身：为什么 `server/api` 里的文件自动带 `/api`、而 `server/routes` 不带，两种目录该怎么选。

## 本文依据

- server 目录（`directory-structure/server`）：结构图（api/hello.ts→/api/hello、routes/bonjour.ts→/bonjour、middleware/log.ts）；`defineEventHandler()`/`eventHandler()` 别名与"直接返回 JSON/Promise/event.node.res.end()"；开头 important（app 与 server 代码不许互 import）；Server Routes（"Files inside the `~~/server/api` are automatically prefixed with `/api`"、"To add server routes without `/api` prefix, put them into `~~/server/routes`"、note 动态路由不如 pages 完整、`hello.ts` 两示例）；Server Middleware（见 291）；Server Plugins（"automatically read any files in `~~/server/plugins` ... register them as Nitro plugins ... hooking into lifecycle events"、`defineNitroPlugin` 示例）；Server Utilities（"add more helpers yourself inside the `~~/server/utils`"、`defineWrappedResponseHandler` 示例）；Server Types（"`~~/server/types/` are auto-imported in the server context only"、`Todo` 示例、"Only files directly in server/types/ are scanned"、app 需要的放 shared/types）；Server Alias（`#server` 示例、note 只能在 server 目录内用、客户端 import 报错）。
