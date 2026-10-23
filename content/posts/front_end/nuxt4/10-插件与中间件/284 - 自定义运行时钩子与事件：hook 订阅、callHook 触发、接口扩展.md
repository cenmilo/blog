---
url: /posts/4ac382652a1068b0e8102f5855da3ba2/
title: 284 - 自定义运行时钩子与事件：hook 订阅、callHook 触发、接口扩展
date: 2027-01-14T11:00:00+08:00
updated: 2027-01-14T11:00:00+08:00
author: cmdragon

summary:
  摘要：框架内置的钩子（app:created、vue:error 那些）是 Nuxt 自己触发的；这一篇讲反过来自己造——用同一套 hook/callHook 做一个解耦的事件系统。官方 events 页给了完整配方：hook('名字', cb) 当监听、await callHook('名字', payload) 触发，payload 是按引用传的所以监听者能改它把数据回传给发起方。要让自定义钩子名有类型，就去扩展 RuntimeNuxtHooks / NuxtHooks / NitroRuntimeHooks 三个接口。DevTools 的 Hooks 面板可以观察所有事件。

categories:
  - 前端开发

tags:
  - nuxt4
  - 自定义事件
  - callHook
  - 钩子
  - hookable
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


前面几篇订阅的都是 Nuxt 内置的钩子时刻（app:created、page:start、vue:error）。这一篇把方向反过来：**你自己定一个事件名、自己触发、让别人订阅**——用它来解耦代码。官方 events 那一节给的就是这套。

## 为什么用它：解耦

官方的动机例子很具体：

> Using events is a great way to decouple your application and allow for more flexible and modular communication between different parts of your code. Events can have multiple listeners that do not depend on each other. For example, you may wish to send an email to your user each time an order has shipped. Instead of coupling your order processing code to your email code, you can emit an event which a listener can receive and use to dispatch an email.

订单发货要发邮件——但你不希望"处理订单的代码"直接依赖"发邮件的代码"。于是订单处理只发一个事件，谁想发邮件谁去监听，加个"发短信"的监听也不用改订单代码。事件可以有多个互不依赖的监听者。

底层机制官方点明了：

> The Nuxt event system is powered by unjs/hookable, which is the same library that powers the Nuxt hooks system.

也就是说"事件系统"和"钩子系统"是同一个东西（都来自 hookable）。这就是为什么本篇用的 API 还是 `hook` 和 `callHook`——275、276 讲的那对。

## 订阅：用 hook

自己起个事件名，用 `nuxtApp.hook` 监听：

```ts
const nuxtApp = useNuxtApp()

nuxtApp.hook('app:user:registered', (payload) => {
  console.log('A new user has registered!', payload)
})
```

`'app:user:registered'` 是你编的名字，`payload` 是触发时传进来的数据。

## 触发：用 callHook

反过来，用 `callHook` 发事件、通知所有监听者：

```ts
const nuxtApp = useNuxtApp()

await nuxtApp.callHook('app:user:registered', {
  id: 1,
  name: 'John Doe',
})
```

这正好把 276 讲的 `callHook` 用起来了——`await` 它，等于等所有 `'app:user:registered'` 的监听跑完。

## payload 按引用传：监听者能改它、把数据回传给发起方

这是官方专门给的一段，也是最容易被忽略的能力：

> You can also use the payload object to enable two-way communication between the emitter and listeners. Since the payload is passed by reference, a listener can modify it to send data back to the emitter.

官方例子：一个监听往 payload 上挂了 `message` 字段，触发方拿到的那个对象上就有这个字段了：

```ts
const nuxtApp = useNuxtApp()

nuxtApp.hook('app:user:registered', (payload) => {
  payload.message = 'Welcome to our app!'
})

const payload = {
  id: 1,
  name: 'John Doe',
}

await nuxtApp.callHook('app:user:registered', {
  id: 1,
  name: 'John Doe',
})

// payload.message will be 'Welcome to our app!'
```

机制就一句 "the payload is passed by reference"（按引用传递）：监听者拿到的不是副本，是同一个对象，所以它改的属性，触发方事后也能在**它自己那个对象**上读到——于是事件不只是"单向广播"，还能变成"发起方问、监听方答"的双向通信。

注意官方这段示例里一个小坑：它声明了 `const payload = { id: 1, name: 'John Doe' }`，但真正传给 `callHook` 的是**又一个新写的字面量**，而不是那个 `payload` 变量。严格按这段抄的话，`payload.message` 未必挂得到你手上那个对象——要演示"回传给发起方"，正确写法是把同一个 `payload` 变量传给 `callHook`。我按官方意图给你这条判据：**要用双向通信，就得让 callHook 传入的对象和你后面要读的那个对象是同一个引用。**

## 给自定义钩子补类型：扩展三个接口

自己起的事件名默认没有类型。官方 hooks 那一节的 "Adding Custom Hooks" 给了扩展口子——按用途 declare 三个不同接口：

```ts
import type { HookResult } from '@nuxt/schema'

declare module '#app' {
  interface RuntimeNuxtHooks {
    'your-nuxt-runtime-hook': () => HookResult
  }
  interface NuxtHooks {
    'your-nuxt-hook': () => HookResult
  }
}

declare module 'nitropack/types' {
  interface NitroRuntimeHooks {
    'your-nitro-hook': () => void
  }
}
```

三个接口对应三类钩子，别放错：

- `RuntimeNuxtHooks`（在 `#app`）：应用运行时钩子（就是 `nuxtApp.hook` / `useRuntimeHook` 用的那套，本篇的 `'app:user:registered'` 归这儿）。
- `NuxtHooks`（也在 `#app`）：构建期钩子（模块/nuxt.config 那套，275 里提过别和运行时混）。
- `NitroRuntimeHooks`（在 `nitropack/types`）：服务端 Nitro 运行时钩子。

补上之后，你自定义的名字在 `hook`/`callHook`/`useRuntimeHook` 处就有自动补全和签名校验。

## 三类钩子的位置（顺带对齐）

官方 hooks 那一节把钩子系统分成三层，本篇讲的是中间的"App Hooks (Runtime)"这一层的自定义用法。另两层留个坐标，别混：

- **Nuxt Hooks (Build Time)**：给模块和构建上下文用，`nuxt.config.ts` 里 `hooks: { close: () => {} }` 或模块里 `nuxt.hook('close', ...)`。
- **App Hooks (Runtime)**：插件/组合式用，`nuxtApp.hook('page:start', ...)`——本篇的事件就挂在这层。
- **Server Hooks (Runtime)**：服务端插件用，`defineNitroPlugin` 里 `nitroApp.hooks.hook('render:html', ...)`。

## 可观测性

官方给了一句 tip：

> You can inspect all events using the **Nuxt DevTools** Hooks panel.

调试自定义事件时，DevTools 的 Hooks 面板能看到事件的流动，不用全靠 console。

## 自查

- 想解耦"发生了某事、让别人各自响应" → 自定义事件：`hook(name, cb)` 订阅、`await callHook(name, payload)` 触发。
- 要让监听方把数据回传给发起方 → 传同一个 payload 对象进去（按引用），别像官方示例那样另写字面量。
- 自定义名字要类型 → 运行时钩子扩 `RuntimeNuxtHooks`、构建期扩 `NuxtHooks`、Nitro 扩 `NitroRuntimeHooks`。
- 观察事件流用 DevTools 的 Hooks 面板。

## 下一篇

下一篇 285 讲插件里抛出的错误怎么走：`vue:error` 钩子与 `vueApp.config.errorHandler` 接客户端错误、`app:error` 接启动期（含插件抛错）错误，以及"插件抛错后要清错误才会重跑"那条坑。
