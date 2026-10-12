---
url: /posts/cd7ce788442be1bcf72aa3af4d4e64c6/
title: 服务端中间件：server 下的 middleware 跑在每个请求且不许返回
date: 2027-01-17T09:00:00+08:00
updated: 2027-01-17T09:00:00+08:00
author: cmdragon

summary:
  摘要：server/middleware 和 app/middleware 名字像、机制相反，是两码事。服务端中间件跑在 Nitro 那一侧，每个请求在任何服务端路由处理之前都执行——加头、查头、记日志、扩展 event.context。它最关键的一条约束是"不该返回任何东西（也不该关闭或响应请求）"，只能检视/扩展请求上下文或抛错；这和前端路由中间件"靠返回值指令导航"正好相反。这一篇讲它的作用域、这条禁止返回的规则，以及它到底能拦什么（连 /api 都跑，正是补前端中间件够不到的那块）。

categories:
  - 前端开发

tags:
  - nuxt4
  - 服务端中间件
  - server-middleware
  - h3
  - event-context
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上一篇（290）结尾留了个钩子：路由中间件不作用在 `/api`，要守服务端请求得用**服务端中间件**。这一篇就是它。它和前端中间件虽然都叫 middleware，机制几乎是相反的——最容易搞错的就是这条"不许返回"。

## 它在哪、跑多勤

server 那页 "Server Middleware" 一节：

> Nuxt will automatically read in any file in the `~~/server/middleware` to create server middleware for your project.
> Middleware handlers will run on every request before any other server route to add or check headers, log requests, or extend the event's request object.

拆两点：

- **位置**：文件放 `server/middleware/`，自动注册，不用登记。
- **作用域**：**on every request**、**before any other server route**。也就是每个进来的请求（页面 SSR、`/api/*`、`/routes/*` 都算）在任何具体服务端路由处理之前，这些中间件先按序跑一遍。这正是它能补前端中间件够不到的地方——前端路由中间件明确不跑 `/api`，服务端中间件跑**所有**请求。

用途官方列了三类：加或查请求头、记日志、扩展 `event` 的请求对象（往下一个处理环节带数据）。

## 最硬的规则：不该返回任何东西

这条和前端中间件完全相反，单拎出来记。server 那页的 note：

> Middleware handlers should not return anything (nor close or respond to the request) and only inspect or extend the request context or throw an error.

对比一下两端的"返回值语义"：

| | 前端路由中间件（290） | 服务端中间件（本篇） |
| --- | --- | --- |
| 靠返回值干嘛 | **靠返回值指令导航**：navigateTo / abortNavigation / 不返回=放行 | **不该返回**任何值 |
| 能拦什么 | 页面导航 | 每个服务端请求 |
| 出错方式 | 返回 abortNavigation(error) | throw 一个错误 |
| 想传递数据 | 不适用 | 写进 `event.context`，给下游 handler 读 |

服务端中间件的合法动作只有三种：**检视请求**、**扩展 `event.context`**、**抛错**。它不返回响应、不关闭请求、不"return 一个值表示什么"——那是 h3 handler 的写法，不是 middleware 的。想给后续处理塞点东西，是挂到 `event.context` 上。

## 两个官方示例

记日志（检视请求，不返回）：

```ts [server/middleware/log.ts]
export default defineEventHandler((event) => {
  console.log('New request: ' + getRequestURL(event))
})
```

给下游塞鉴权信息（扩展 context，不返回）：

```ts [server/middleware/auth.ts]
export default defineEventHandler((event) => {
  event.context.auth = { user: 123 }
})
```

第二个就是"扩展 event 的请求对象"的标准形态：把 `auth` 挂到 `event.context.auth`，之后的服务端路由能用 `event.context.auth` 读到。真要拦，是在这里 `throw createError({ status: 401 })`（292 会接上会话鉴权那条）。

## 别和前端 Vue 代码混

服务端中间件里不要 import 前端应用代码。server 那页开头那条 important 划的就是这条边界：

> Do not import Vue app code (components, composables, or other app-only utilities) in your server routes or utilities, and do not import server-only code in your app.

这和 288 讲 `nuxt/server` "Server Code Only" 是同一层规矩的两面：app 与 server 两个运行环境不许互相 import。所以 `server/middleware` 里那些 helper（`getRequestURL` 等）是 h3/服务端面的，不是应用组合式。

## 一张"该用哪个中间件"的判断

- 守某个**页面**的访问、要给用户跳转到登录页 → 前端路由中间件（`app/middleware`）。
- 守**每个服务端请求/接口**、要检查头、记日志、往 event.context 塞服务端才有的数据 → 服务端中间件（`server/middleware`）。
- 前端中间件只是体验，真正保护数据要在服务端做——这条 259 那句已经立过（"Client-side middleware is helpful for the user, but without server-side protection your data can still be accessed."）。

## 自查

- 在 `server/middleware` 里 `return` 了一个值？这是错的——服务端中间件不该返回任何东西，只能检视、扩展 `event.context`、或 throw。
- 想拦 API 请求 / 每个请求都跑一遍？路由中间件不作用在 `/api`，要用服务端中间件。
- 要往后续 handler 传数据？挂 `event.context.xxx`，不是靠返回值。
- 别在 `server/middleware` 里 import 前端组件/组合式（app 与 server 代码不许混）。

## 下一篇

下一篇 292 把 259 的会话配方和中间件合起来：客户端中间件守页面、服务端 `requireUserSession` 守数据，官方那份鉴权配方的完整落点。

## 本文依据

- server 目录（`directory-structure/server`）：Server Middleware 一节（"automatically read in any file in the `~~/server/middleware`"、"Middleware handlers will run on every request before any other server route to add or check headers, log requests, or extend the event's request object."、note "Middleware handlers should not return anything (nor close or respond to the request) and only inspect or extend the request context or throw an error."、`log.ts` 与 `auth.ts`（`event.context.auth = { user: 123 }`）示例）；开头 important "Do not import Vue app code ... and do not import server-only code in your app."。
- Routing（`getting-started/routing`）：important 路由中间件不跑 `/api/*`、要守服务端请求改用 server middleware。
- Sessions and Authentication（`guide/recipes/sessions-and-authentication`，259 已引）："Client-side middleware is helpful for the user, but without server-side protection your data can still be accessed."
