---
url: /posts/76655fe80e10de7d4914388eef449cc4/
title: 310 - showError：在应用侧抛出可渲染的错误页
date: 2027-01-23T10:00:00+08:00
updated: 2027-01-23T10:00:00+08:00
author: cmdragon

summary:
  摘要：showError 是"不通过 throw、直接命令式地弹一个全屏错误页"的工具。它只能在 Nuxt 上下文里调，参数可以是字符串、Error 或一个 Partial 对象（status/statusText/data 等）。它内部用 useError() 把错误写进一个响应式、SSR-friendly 的共享错误状态，并会调用 app:error 钩子。这一篇讲它的定位与两个官方示例，再给一条取舍：官方其实更推荐用 throw createError()，showError 是"我就想直接弹一页"时的便捷出口。

categories:
  - 前端开发

tags:
  - nuxt4
  - showError
  - 错误页
  - useError
  - app-error
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


上一篇讲了"throw 一个 createError"。这一篇讲它的命令式版本：**`showError`——不 throw、直接调用，就弹一个全屏错误页。** 官方定位：

> Nuxt provides a quick and simple way to show a full screen error page if needed.

## 在哪能调、参数长什么样

限定和 307 那条一样，绑在上下文上：

> Within the [Nuxt context] you can use `showError` to show an error.

参数：

> `error`: `string | Error | Partial<{ cause, data, message, name, stack, status, statusText }>`

三选一——一个字符串、一个现成的 `Error`、或一个 Partial 对象（字段和 308 createError 那批一致）。两个官方示例：

```ts
showError('😱 Oh no, an error has been thrown.')
showError({
  status: 404,
  statusText: 'Page Not Found',
})
```

第一个传字符串（当 message），第二个传对象带 status/statusText——和 createError 的参数习惯一脉相承。

## 它做了两件事：写共享错误状态 + 调 app:error

官方对 `showError` 背后机制有两句，正好把它和前面几篇串起来。

**一、把错误写进一个 SSR-friendly 的共享状态：**

> The error is set in the state using `useError()` to create a reactive and SSR-friendly shared error state across components.

也就是 `showError` 不是"直接打印"，而是把错误塞进 `useError()` 那个**响应式、跨组件共享**的错误状态里；错误页、其它组件读 `useError()` 都能拿到同一份。这跟 234 讲的 `useError` 是同一个东西。

**二、它会调用 app:error 钩子：**

> `showError` calls the `app:error` hook.

这句把它接回了 285 那张错误分流表——`app:error` 正是"启动期/致命错误"那条钩子。你在插件里 `nuxtApp.hook('app:error', ...)` 订阅的，就是这类被 showError/createError 触发的致命错误。所以 `showError` = "写共享错误态 + 触发 app:error + 渲染全屏错误页" 的打包出口。

## 什么时候用 showError，什么时候 throw createError

两者最终都能弹全屏错误页，但官方给了一条明确取舍。error-handling 那页 "Error Page" 之后原话：

> It is recommended instead to use `throw createError()`.

也就是**优先 `throw createError()`**，`showError` 是次选。差异在心智模型：

- `throw createError(...)` → 走 JS 的异常流（中断当前执行、往上传播被捕获），符合"出错了就抛出"的直觉，也方便在中间件/setup 里以 throw 结束流程。
- `showError(...)` → 命令式调用、不抛异常，适合"我已经在一个回调里、不想用 throw 中断，只想主动亮一个错误页"的场景。

308 也补过客户端的差别：客户端 `throw createError` 默认是 non-fatal，要全屏得 `fatal: true`；而 `showError` 本身就是奔着"显示全屏错误页"去的。

## 自查

- 要"直接弹一屏错误页"又在一个不想 `throw` 的位置 → `showError('...' | {status, statusText, ...})`。
- 它只能在 Nuxt 上下文里调；参数可以是字符串、Error 或 Partial 对象。
- 机制上它写了 `useError()` 的共享错误态、并触发 `app:error`（285 那张表）——所以订阅 app:error 能捕获到它。
- 官方口径是优先 `throw createError()`；showError 留给"命令式亮错误页"这种确有其事的场合。

## 下一篇

下一篇 311 讲 `server/utils` 与服务端自动导入的边界：哪些服务端 helper 自动可用、哪些必须显式 import、以及 `#server` 别名的作用范围。
