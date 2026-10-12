---
url: /posts/5d286c05e7ee77643ad3e071193daf95/
title: 内幕两个接口：nuxt 是构建核心、nuxtApp 是运行时核心
date: 2027-06-05T10:00:00+08:00
updated: 2027-06-05T10:00:00+08:00
author: cmdragon

summary:
  摘要：Nuxt 只有两个上下文对象。**`nuxt`** 在你跑 `nuxt dev` 或 `nuxt build` 时创建，装着归一化后的选项、一些内部状态，以及一个由 unjs/hookable 驱动的钩子系统——文档称之为 **Builder Core**；**`nuxtApp`** 在浏览器或服务端渲染一个页面时创建，装着 Vue 实例、运行时钩子和 ssrContext、payload 这些水合用的状态——文档称之为 **Runtime Core**。两者**被刻意隔离**：官方原话是它们不该共享状态、代码或上下文，**唯一例外是运行时配置**。这条隔离还决定了一个限制：一个进程里只允许运行一个 Nuxt 实例。

categories:
  - 前端开发

tags:
  - nuxt4
  - 内部机制
  - 构建上下文
  - 运行时上下文
  - 钩子
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## `nuxt`：Builder Core 里装了什么

页面给的定位：用 `nuxt dev` 启动开发模式，或用 `nuxt build` 构建生产应用时，会创建一个共享上下文，内部就叫 `nuxt`。它持有三样东西：

- **归一化后的选项**，并且已经与各 `nuxt.config` 文件合并过
- 一些内部状态
- 一个由 [unjs/hookable](https://github.com/unjs/hookable) 驱动的**钩子系统**，用于让不同组件彼此通信

最后一句是理解 Kit 与模块的关键：**这个上下文是全局可用的，Nuxt Kit 的组合式函数就是靠它工作的，因此一个进程里只允许运行一个 Nuxt 实例**。

这也解释了 `useNuxt()` 与 `tryUseNuxt()` 的行为差别（一个抛错一个返回 null）——它们读的就是这个全局唯一槽位。要扩展这个接口、挂在构建流程的不同阶段上，官方给的手段是**模块**。细节指向 `packages/nuxt/src/core/nuxt.ts`。

## `nuxtApp`：Runtime Core 的八个成员

渲染页面时（浏览器或服务端都算）会创建一个共享上下文，叫 `nuxtApp`。它保存 Vue 实例、运行时钩子，以及 `ssrContext` 和 payload 这类内部状态。

页面把接口形状整个贴了出来，逐行落地：

```ts
interface NuxtApp {
  vueApp // the global Vue application

  versions // an object containing Nuxt and Vue versions

  // These let you call and add runtime NuxtApp hooks
  hooks
  hook
  callHook

  // Only accessible on server-side
  ssrContext: {
    url
    req
    res
    runtimeConfig
    noSSR
  }

  // This will be stringified and passed from server to client
  payload: {
    serverRendered: true
    data: {}
    state: {}
  }

  provide: (name: string, value: any) => void
}
```

注释里那两句是文档原意，各自划定了一件事：`vueApp` 就是那个全局 Vue 应用；`versions` 是一个装着 Nuxt 与 Vue 版本的对象；`hooks`/`hook`/`callHook` 三个是**运行时**钩子的读取与注册入口；**`ssrContext` 只有服务端能访问**（里面是 `url`、`req`、`res`、`runtimeConfig`、`noSSR` 五项）；**`payload` 会被序列化，从服务端传给客户端**，形状是 `serverRendered`、`data`、`state` 三个键；`provide` 是给命名值做注入的方法。

`payload` 那一句正好解释了 `experimental.payloadExtraction` 在改什么：它决定这份数据是内联在 HTML 里还是抽成 `_payload.json`。

## 两个上下文的取用规则不同

`nuxt` 通过 Kit 函数（模块里）取；`nuxtApp` 通过 `useNuxtApp()` 取，**可用范围被写明是 Nuxt 插件、`<script setup>` 与 Vue 组合式函数**。

服务端有一条额外的安全限制，页面写得很直接：**全局用法在浏览器里可以，在服务端不行，目的是避免在不同用户之间共享上下文**。

还有一条取舍规则：`useNuxtApp` 在上下文不可用时会**抛异常**；如果你的组合式函数并非总是需要 `nuxtApp`，应该改用 `tryUseNuxtApp`，**它返回 `null` 而不是抛错**。

要扩展 `nuxtApp` 接口、挂进不同阶段或访问上下文，官方给的手段是**插件**（不是模块）。

## Runtime Context 与 Build Context 是被隔离的

这一节的原文值得整段读：Nuxt 用 Node.js 构建与打包项目，但也有一侧是运行时。两侧都能扩展，**但运行时上下文与构建期是隔离的，因此它们不应该共享状态、代码或上下文——只有运行时配置例外。**

于是分工被写成一句话：`nuxt.config` 与模块用来扩展**构建上下文**，插件用来扩展**运行时**。

这条隔离解释了三个常见困惑：

- 为什么模块的选项在运行时读不到，得用 `runtimeConfig` 显式暴露出去（配方页那句"模块不是应用运行时的一部分，所以模块选项也不是"）。
- 为什么 `nuxt/kit` 不能在组件、页面、插件或 server routes 里 import。
- 为什么 `experimental.asyncContext` 值得存在——它处理的就是"在异步函数里还能不能拿到运行时上下文"。

## 生产构建不带着这两套扩展走

最后一句是关于交付物的：

> 为生产构建应用时，`nuxt build` 会在 `.output` 目录里生成一个**独立的构建产物，与 `nuxt.config` 和 Nuxt 模块无关**。

这句话把"构建期"与"部署产物"的边界定死了：模块与配置只在**生成**那一步起作用，`.output` 一旦产出就不再回读它们。这也意味着你在部署环境里改 `nuxt.config` 不会影响已经产出的 `.output`。

至于构建期内部到底按什么顺序跑——虚拟应用怎么生成、`build:before` 与 `build:done` 之间发生了什么、构建产物契约的六个键怎么把产物交给服务端运行时——那是构建器那一章的内容，本系列已在讲 builder 契约的那篇逐项拆过。

## 三个自检问题

- 你写的这段代码跑在哪一侧？构建期（模块、Kit、钩子）还是运行时（插件、`nuxtApp`、组合式函数）？
- 需要跨两侧传值吗？只有 `runtimeConfig` 这一条官方许可的通道。
- 需要拿上下文但可能拿不到吗？服务端别用全局取法；`useNuxtApp` 会抛错，条件性需求用 `tryUseNuxtApp`。

下一篇讲 nightly 发布频道：什么情况下该把版本指到每夜构建，配置里那行 `npm:@nuxt/kit-nightly@latest` 式的写法从哪来。

## 本文依据

- https://nuxt.com/docs/4.x/guide/going-further/internals
- https://nuxt.com/docs/4.x/guide/going-further/builders
