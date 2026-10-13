---
url: /posts/089383a6f229450b8cc8d32b5795bc6d/
title: 307 - setResponseStatus：设置状态码与那句浏览器里不生效
date: 2027-01-22T10:00:00+08:00
updated: 2027-01-22T10:00:00+08:00
author: cmdragon

summary:
  摘要：想给一个响应设 HTTP 状态码（不只默认 200），Nuxt 给的工具是 setResponseStatus(event, code, statusText?)。这一篇讲它的两个限定：一是只能在 Nuxt 上下文里调（官方 important），二是那句关键的"在浏览器里没有任何效果"——因为状态码是服务端发响应时才有的东西，纯客户端拿不到 event、也改不了已经发出去的响应。用法上配合 useRequestEvent()，在浏览器 event 会是 undefined，要判一下再设。

categories:
  - 前端开发

tags:
  - nuxt4
  - setResponseStatus
  - 状态码
  - 上下文
  - SSR
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上一篇（306）读请求，这一篇讲怎么把响应"标上正确的状态码"。默认响应是 200；要返回 404、202 这种，用 `setResponseStatus`。

## 它是什么

官方定位：

> `setResponseStatus` sets the status (and optionally the statusText) of the response.

一个函数，收 `event`、状态码、可选的 statusText。

## 用法：配 useRequestEvent，且要判 event

官方示例：

```ts
const event = useRequestEvent()

// event will be undefined in the browser
if (event) {
  // 为自定义 404 页设状态码 404
  setResponseStatus(event, 404)

  // 同时也设状态信息
  setResponseStatus(event, 404, 'Page Not Found')
}
```

这段把两个要点一起演示了：

1. `setResponseStatus` 需要一个 `event`。在组件/页面里，这个 `event` 通过 `useRequestEvent()` 取。
2. 注释那句 **"event will be undefined in the browser"**——纯客户端没有这次服务端请求的 event，所以取出来是 `undefined`，因此官方示例先 `if (event)` 再设。

## 两条硬限定

**限定一：只能在 Nuxt 上下文里调。** 官方 important：

> `setResponseStatus` can only be called in the [Nuxt context].

也就是它跟 295 讲的那四类合法位置绑定（setup、插件、中间件、钩子回调）。不在上下文里调 → 拿不到 nuxtApp/event，就是 E1001 那一类问题（277）。

**限定二：在浏览器里没有效果。** 官方 note 写死了：

> In the browser, `setResponseStatus` will have no effect.

原因就在状态码这件事本身——HTTP 状态码是**服务端发响应那一刻**写进响应头的东西。客户端拿到响应之后，没有"改自己已经收到的状态码"这回事；SSR 阶段服务端发的响应才有状态码可设。所以 `setResponseStatus` 只在服务端那一趟真正生效，客户端调它静默无效果（也呼应上面那句 `event` 在浏览器是 undefined）。

## 和 createError 的分工

设状态码有两条路，别混：

- 你**正常返回内容、只是想给它标个状态码**（比如返回一个自定义 404 页但状态码要是 404、或返回 202 Accepted）→ `setResponseStatus(event, code)` + 正常 return 内容。
- 你要**抛一个错误**（让错误处理/错误页接管）→ `createError({ status })`（308 讲）。

官方 server 那一节 "Status Codes" 给的正是第一条路的典型用法：

```ts [server/api/validation/[id].ts]
export default defineEventHandler((event) => {
  setResponseStatus(event, 202)
})
```

返回 `202 Accepted`。这是在服务端 handler 里设码，`event` 是 handler 的入参、本来就存在。

## 自查

- 想给"正常返回的内容"标状态码 → `setResponseStatus(event, code, statusText?)`；想抛错误走错误处理 → `createError({ status })`。
- 在组件里用要先 `useRequestEvent()` 且 `if (event)` 判空——浏览器里 event 是 undefined。
- 它只能在 Nuxt 上下文调（295）；在浏览器调用没有效果，别指望客户端能改状态码。
- 服务端 handler 里 event 是现成入参，直接 `setResponseStatus(event, 202)` 就行。

## 下一篇

下一篇 308 讲服务端错误的正规姿势：`createError` 的参数 status/statusText/body/data/fatal，以及字符串参数和对象参数的差别。

## 本文依据

- setResponseStatus（`api/utils/set-response-status`）：定位句；important "setResponseStatus can only be called in the Nuxt context."；`useRequestEvent()` + `// event will be undefined in the browser` + `if (event)` + `setResponseStatus(event, 404)` / `setResponseStatus(event, 404, 'Page Not Found')` 示例；note "In the browser, setResponseStatus will have no effect."
- server 目录（`directory-structure/server`）：Status Codes 一节（`setResponseStatus(event, 202)` 返回 202 Accepted，`[id].ts` 示例）；Nuxt context 定义（见 295）。
