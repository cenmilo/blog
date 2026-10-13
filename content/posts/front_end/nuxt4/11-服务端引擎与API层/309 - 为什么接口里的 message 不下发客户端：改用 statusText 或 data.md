---
url: /posts/5e4591a3a7ee944a1ff0099dbcc933f3/
title: 309 - 为什么接口里的 message 不下发客户端：改用 statusText 或 data
date: 2027-01-23T09:00:00+08:00
updated: 2027-01-23T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：一个常见困惑：接口里 createError({ message: '...' })，前端 useFetch 的 error 里却读不到那句 message。官方 create-error 页把原因写死了——在 API 路由里传给 createError 的 message 不会传到客户端。要让前端拿到信息，官方给两条：用一个短的 statusText（它能被客户端访问），或者用 data 属性带结构化数据（配合 useFetch 时，自定义数据在 error.value.data.data 这个位置）。这一篇把这条下发规则和两个替代写清，并重申那条安全口径。

categories:
  - 前端开发

tags:
  - nuxt4
  - createError
  - statusText
  - data
  - useFetch
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

接口里抛了个 `createError({ status: 400, message: '邮箱格式不对' })`，前端 `useFetch` 拿到 `error`，点开却发现那句中文 `message` 不见了、只剩状态码。这不是 bug，是官方定的下发规则。这一篇讲清"为什么"和"那我要给前端带信息该怎么办"。

## 规则：API 路由里的 message 不下发

官方 create-error 页 "In API Routes" 那段，把这条讲得毫不含糊：

> In API routes, using `createError` by passing an object with a short `statusText` is recommended because it can be accessed on the client side. Otherwise, a `message` passed to `createError` on an API route will not propagate to the client.

三句拆开：

1. **推荐**给 API 路由的 createError 传一个带**短 `statusText`** 的对象——因为 `statusText` 能在客户端访问到。
2. 你不那么做、只给 `message` → **"a `message` passed to `createError` on an API route will not propagate to the client."**——API 路由里的 message 不会传到客户端。
3. 也就是说：`message` 是"服务端本地看/记日志用"的，`statusText` 才是"下发给前端"的那条短文本。

这就是那个困惑的直接答案。你在服务端 handler 里写的 `message` 被有意挡在客户端之外。

## 要带结构化数据：用 data

短文本用 `statusText`，但如果你要下发的是**结构化的错误信息**（比如哪个字段错了、错误列表），官方给的是 `data`：

> Alternatively, you can use the `data` property to pass data back to the client. When handling the error with `useFetch`, the custom data is available at `error.value.data.data`.

这里那个取用路径 `error.value.data.data` 有两个 `data`，容易看晕，拆一下为什么长这样：

- 外层 `error.value.data` 是 **NuxtError 对象上的 `data` 字段**（错误对象本身的结构，`message/status/statusText/data` 这些是 NuxtError 的属性，见 308 的参数表）。
- 内层 `.data` 是你**自己塞进 `createError({ data: ... })` 的那个 data**。

所以你 `throw createError({ status: 400, data: { field: 'email' } })`，在 `useFetch` 侧读自定义数据是 `error.value.data.data` → `{ field: 'email' }`。两个 `data` 不是写错，是"错误对象的 data 里，装着你传的 data"。

## 为什么这么设计：安全

这条规则不是随手的，它接的是 308 末尾那句安全提醒：

> always consider avoiding to put dynamic user input to the message to avoid potential security issues.

`message` 常常来自底层异常的原文（可能含内部信息、SQL、堆栈片段），Nuxt 把它挡在服务端不下发，是一种默认收口；你要**主动、有意识地**通过 `statusText`（短、可控）或 `data`（结构化、你亲手组）把该给前端的信息放出去。换句话说：**下发内容应该是你显式挑选过的，而不是 handler 里那句随手写的 message 自动漏出去。**

## 三选一速查

| 你要给前端的 | 放哪 | 前端怎么读（useFetch） |
| --- | --- | --- |
| 一条短状态文本 | `statusText` | 推荐、客户端可访问 |
| 结构化错误数据 | `data` | `error.value.data.data` |
| 只给服务端看/记日志 | `message` | **不下发到客户端** |

## 自查

- 接口 `createError` 的 `message` 不会传到客户端——别指望前端读到它，那是服务端本地用的。
- 要给前端一句短信息 → 用短的 `statusText`；要给结构化错误 → 用 `data`，配合 useFetch 时读 `error.value.data.data`。
- 下发内容应当是你显式挑过的，别把动态用户输入或内部异常原文塞进会下发的字段（安全）。
- 两端别混：这是"API 路由里的 createError"的下发规则；页面里 throw 触发的全屏错误页是 308 讲的另一回事。

## 下一篇

下一篇 310 讲 `showError`：在应用侧主动抛一个可被渲染成错误页的错误，以及它和 createError、app:error 钩子的关系。

## 本文依据

- createError（`api/utils/create-error`）：In API Routes 一节原句——"using `createError` by passing an object with a short `statusText` is recommended because it can be accessed on the client side. Otherwise, a `message` passed to `createError` on an API route will not propagate to the client. Alternatively, you can use the `data` property to pass data back to the client. When handling the error with `useFetch`, the custom data is available at `error.value.data.data`."；安全句 "avoiding to put dynamic user input to the message to avoid potential security issues."；参数表 `{ cause, data, message, name, stack, status, statusText, fatal }`（NuxtError 字段，见 308）。
- Error Handling（`getting-started/error-handling`）：NuxtError 结构（见 056/90-109 记录，message/status/statusText/data 为错误对象字段）。
