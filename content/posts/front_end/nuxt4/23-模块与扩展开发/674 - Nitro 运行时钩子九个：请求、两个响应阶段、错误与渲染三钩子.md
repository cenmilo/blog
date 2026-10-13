---
url: /posts/ba131175e4c5f77743e686f0ff57ea7f/
title: 674 - Nitro 运行时钩子九个：请求、两个响应阶段、错误与渲染三钩子
date: 2027-05-24T11:00:00+08:00
updated: 2027-05-24T11:00:00+08:00
author: cmdragon

summary:
  摘要：第三张表是服务端运行时的九个钩子，**挂载方式是 `nitroApp.hooks.hook(...)`（两层）**，而**这一节开头就写着"全部钩子请看 Nitro 文档"——这九条只是 Nuxt 侧列出的那部分**。九行分成四类：**一个请求入口 `request`（参数 `event`）与一个错误口 `error`（参数 `error, { event? }`）**；**`beforeResponse` 与 `afterResponse` 成对**（都带 `event, { body }`）；**渲染三钩子 `render:response`、`render:html`、`render:island`**（参数分别是 `response, { event }`、`html, { event }`、`islandResponse, { event, islandContext }`）；再加 `close` 和 `dev:ssr-logs`。两处必须知道：**`render:response` 与 `beforeResponse` 的描述逐字相同**（都写"在发送响应之前调用"）；**`dev:ssr-logs` 那一行在本表里列位错乱**，"Server" 落到了描述列上。

categories:
  - 前端开发

tags:
  - nuxt4
  - 服务端钩子
  - Nitro
  - 渲染流程
  - 中间件
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

想在服务端给每个请求记一笔、或者往最终 HTML 里塞一段内容，位置就在这九行里。它们的触发点都在服务端进程内，**挂法与另外两张表不同**。

## 先解决"怎么挂"

服务端这一族要通过 Nitro 插件注册，官方示例给的就是这个形状：

```ts [~~/server/plugins/test.ts]
export default defineNitroPlugin((nitroApp) => {
  nitroApp.hooks.hook('render:html', (html, { event }) => {
    console.log('render:html', html)
    html.bodyAppend.push('<hr>Appended by custom plugin')
  })

  nitroApp.hooks.hook('render:response', (response, { event }) => {
    console.log('render:response', response)
  })
})
```

三件事在这一段里：**取到插件入口要 `defineNitroPlugin`**；**注册要写 `nitroApp.hooks.hook(...)`，比应用侧多一层**；**`html.bodyAppend.push(...)` 是往 body 末尾追加内容的官方做法**。

而这个文件要靠 `addNitroPlugin` 注册进项目，那条页的警告写着：**插件文件里必须显式从 `nitropack/runtime` 导入 `defineNitroPlugin`**（示例里恰好没写那行导入，**照抄会坏**）。

## 九行照录

这张表有**四列：Hook、Arguments、Description、Types**——**Types 是它独有的一列**，指向类型定义的源码位置。

| 钩子 | 参数 | 描述 |
| --- | --- | --- |
| `dev:ssr-logs` | `{ path, logs }` | 见下文，本行列位错乱 |
| `render:response` | `response, { event }` | 在发送响应之前调用 |
| `render:html` | `html, { event }` | 在构造 HTML 之前调用 |
| `render:island` | `islandResponse, { event, islandContext }` | 在构造 island 的 HTML 之前调用 |
| `close` | - | 当 Nitro 被关闭时调用 |
| `error` | `error, { event? }` | 当发生错误时调用 |
| `request` | `event` | 当收到一个请求时调用 |
| `beforeResponse` | `event, { body }` | 在发送响应之前调用 |
| `afterResponse` | `event, { body }` | 在发送响应之后调用 |

Types 一列给出的位置很具体：`response`、`html`、`islandResponse` 指向 Nuxt 渲染器源码的三处不同行，**`event` 一律指向 h3 的类型文件同一行**，`error` 指向 Nitro 的类型文件；`beforeResponse` 与 `afterResponse` 的第二个类型位置写作 unknown。

## 两个描述完全相同

`render:response` 与 `beforeResponse` 在表里的描述**逐字相同**，都是 "Called before sending the response."。

从名字与参数能看出它们不是一件事（一个拿到 `response` 对象，一个拿到 `event, { body }`），**但"渲染响应"与"发送响应"这两个阶段在这张表里没有用文字区分开**——**要确定先后只能实测**，这是文档的缺口，不要当成有依据的顺序。

同类情况还有一处：`close` 这个名字**在构建期表里也有一行**（"当 Nuxt 实例正在优雅关闭时调用"），**两张表各有一个 `close`**。

## 四类钩子各自能做什么

**请求与错误**：`request`（收到请求时）与 `error`（发生错误时，且 **`event` 是可选的**——`{ event? }` 意味着**错误可能发生在没有请求上下文的时候**）。这两个是"每个请求都过一遍"的位置，与中间件的区别是**它们不决定是否继续处理请求**（表里没说可以中断；描述只有"被调用"）。

**响应两阶段**：`beforeResponse`／`afterResponse`，参数形状一致（`event, { body }`），**`body` 出现在两者里，说明这是读响应体的位置**。

**渲染三钩子**：`render:html`（构造 HTML 之前）、`render:island`（构造 island 的 HTML 之前）、`render:response`（发送响应之前）。**`render:island` 是这九行里参数最多的**：它额外带一个 `islandContext`，与岛屿渲染的上下文对应。

**`dev:ssr-logs`**：`{ path, logs }` 两个键一起给，描述（按内容读）是"在一个请求周期结束时，带一组服务端日志被调用"。它对应的是运行时表里那条客户端侧的同名钩子——**一条讲"服务端产出了这组日志"，一条讲"客户端收到了这组日志"**，中间那一步就是 dev 日志转发。

**`close`**：无参数，Nitro 被关闭时调用——**这里是清理服务端资源的位置**，与模块侧"打开/持有/watcher 要在 `close` 里关掉"是同族要求。

## 这张表不是全部

节标题下面第一句就是：**全部可用钩子请看 Nitro 文档**。

所以九行的定位是"Nuxt 侧列出并链了类型的那几个"，**Nitro 自己还有更多运行时钩子不在这张表里**（例如各类编译期/构建期钩子）。**要判断某个钩子存不存在，以 Nitro 文档为准**，不能因为这张表里没有就断定没有。

另一处边界：这些钩子属于服务端运行时，而**"可移植写法"那一章给出的对应入口是 `useServerHooks()`**（`nuxt/server` 提供，用于在服务端代码里挂运行时钩子），并且同一章明确说**服务端插件、存储、缓存这几块 `nuxt/server` 不覆盖**——所以要用这张表的钩子，通常就得走 `nitro2`/`nitro3` 那一侧的实现。

## 三条实测建议

1. 先打印 `request` 与 `beforeResponse`、`afterResponse` 三者的顺序，**把 `render:response` 与 `beforeResponse` 的先后确认下来**——文档没给。
2. 想看 `error` 在没有 event 时是否真的会触发，**构造一次启动期错误**（例如服务端插件里抛错）比构造请求期错误更接近它的可选参数设计。
3. `render:html` 的可改字段以示例给过的 `bodyAppend` 为起点，其余追加位置**页面没有列**（示例只用到这一个）。

这一组未记载的还有：`html` 对象的完整形状、`islandContext` 有哪些字段、这些钩子能否被 `addServerHandler` 的中间件影响顺序，以及 `dev:ssr-logs` 那行错位是否意味着还有一个未列的 "Environment" 概念被漏掉。

下一篇换一页：`import.meta` 上那些能判断"代码跑在哪"的字段。

## 下一篇

下一篇 675：import.meta 在 Nuxt 里的八个运行时字段与两个构建期字段

## 本文依据

- https://nuxt.com/docs/4.x/api/advanced/hooks
- https://nuxt.com/docs/4.x/guide/going-further/hooks
