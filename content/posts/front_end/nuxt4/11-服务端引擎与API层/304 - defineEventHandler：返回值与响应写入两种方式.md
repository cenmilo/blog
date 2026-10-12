---
url: /posts/4fe272547a45dcd46c210101710a8624/
title: defineEventHandler：返回值与响应写入两种方式
date: 2027-01-21T10:00:00+08:00
updated: 2027-01-21T10:00:00+08:00
author: cmdragon

summary:
  摘要：server 目录下每个 handler 文件都要 default 导出一个 defineEventHandler（别名 eventHandler）包的函数。这一篇讲这个函数体里"怎么把响应交出去"的两条路：一是直接 return（对象/数组自动成 JSON、返回 Promise 会被 await），二是用 event.node.res.end() 自己把响应写完。还列清 handler 能合法返回的三类值（可 JSON 序列化的值、Response、ReadableStream）。顺带点一个类型层面的连带后果：只有 return 值、而不是 res.end()，Nitro 才给这条路由生成接口类型。

categories:
  - 前端开发

tags:
  - nuxt4
  - defineEventHandler
  - h3
  - 响应
  - 类型化接口
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

第一篇 server 相关的代码（`server/api/hello.ts`）长这样：

```ts [server/api/hello.ts]
export default defineEventHandler((event) => {
  return { hello: 'world' }
})
```

这一篇把这两行里藏的规则拆开讲：函数怎么定义、return 什么算合法、以及"直接 return"和"自己写响应"两条路的区别。

## 定义：default 导出 defineEventHandler

官方对每个 handler 文件的要求：

> Each file should export a default function defined with `defineEventHandler()` or `eventHandler()` (alias).

两个名字是同一个东西（`eventHandler` 是别名）。函数接收当前请求事件 `event`，你从它读请求、往外发响应。

## 路一：直接 return

server 那页给了返回值语义，server-imports 那页把"handler 可以返回什么"列成了三类，合并看：

> A handler may return:
> - A JSON-serializable value, sent as JSON with the status and headers set on `event.res`.
> - A `Response`, sent with its own status. Headers set on `event.res.headers` are sent with it, and `set-cookie` values from both are kept.
> - A `ReadableStream`, streamed as the body.

逐条：

1. **可 JSON 序列化的值**：对象、数组、字符串、数字……Nuxt/h3 自动把它当 JSON 发（就是 `return { hello: 'world' }`），状态码和响应头从 `event.res` 上取。这是 99% 接口该走的路。
2. **`Response`**：你直接返回一个标准 `Response`，用它的状态码；`event.res.headers` 上设的头会一起发、两边的 `set-cookie` 都保留。
3. **`ReadableStream`**：作为响应体流式发出。

另外一条 server 那页的原文补充：

> The handler can directly return JSON data, a `Promise`, or use `event.node.res.end()` to send a response.

返回 **Promise 会被 await**（这就是 `async (event) => {...}` 里 `await` 一个异步结果再 return 能正常工作的原因）。

## 路二：用 event.node.res.end() 自己写响应

不想走"返回值自动转响应"，就自己把响应用 node 的原始 API 写完：

```ts
export default defineEventHandler((event) => {
  event.node.res.end()
})
```

这是"我已经手动处理了响应，别再替我序列化返回值"的模式。h3 的 `res.end()`、`next()` 官方也声明支持（server 那页 "Handlers can return promises, which will be awaited (`res.end()` and `next()` are also supported)"）。

两条路的取舍：**多数接口 return 值就行，简单、还能带类型（见下）；只有在要完全控制底层响应（自己写状态/流/头、或非 JSON 输出）时，才用 `event.node.res.end()`。**

## 一个类型层面的连带后果

这条很重要，和 300 讲的"类型化接口"接上。官方 server-engine 那页：

> When using API routes (or middleware), Nitro will generate typings for these routes as long as you are returning a value instead of using `res.end()` to send a response.

也就是说：**`return` 值 → Nitro 为这条路由生成返回类型**，你在前端 `useFetch('/api/hello')` / `$fetch('/api/hello')` 处能拿到准确的返回类型推断；**用 `res.end()` → 类型就丢了**。这是"我的接口在前端怎么没有类型、变成 unknown 了"的一个常见原因。所以只要不是非走原始 API 不可，尽量 `return` 值——既简单又保住类型。

server-imports 那页还有个相关 tip：

> `defineEventHandler` preserves your handler's return type, which is what types `$fetch` and `useFetch` calls to the route. Annotate the value you return rather than the handler.

要显式标注类型，就标"你返回的那个值"，而不是标 handler 本身。

## 别在 server 里 import 前端代码

顺带把边界再钉一次（server 那页开头 important）：handler 里不要 import Vue 组件/组合式等 app-only 代码。服务端 handler 和前端是两个 bundle、两个上下文，跨界 import 会构建失败或报错（300、312 都展开过这道墙）。

## 自查

- handler 必须 default 导出 `defineEventHandler()`（或别名 `eventHandler()`）包的函数。
- 优先 `return` 值：对象/数组自动 JSON、Promise 会被 await、也能给路由保住返回类型。
- 只有要完全控制响应（自写状态/流/头、非 JSON）才用 `event.node.res.end()`，代价是接口类型不再自动生成。
- 合法返回值三类：可 JSON 序列化值 / `Response` / `ReadableStream`。
- 别在 handler 里 import 前端 app 代码。

## 下一篇

下一篇 305 讲 handler 怎么读请求数据：`readBody` 读体、`getQuery`/`getValidatedQuery` 读查询串，以及校验失败返回 400 的行为。

## 本文依据

- server 目录（`directory-structure/server`）："Each file should export a default function defined with `defineEventHandler()` or `eventHandler()` (alias)."、"The handler can directly return JSON data, a `Promise`, or use `event.node.res.end()` to send a response."、`hello.ts` 示例、开头 important（别在 server 里 import Vue app 代码）。
- Server Imports（`guide/going-further/server-imports`）：Handlers 一节三类返回值（JSON-serializable/Response/ReadableStream 的原文描述、"res.end() and next() are also supported"）；tip "defineEventHandler preserves your handler's return type ... Annotate the value you return rather than the handler."
- Server Engine（`guide/concepts/server-engine`）：Typed API Routes "generate typings ... as long as you are returning a value instead of using res.end()"（见 300）。
