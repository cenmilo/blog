---
url: /posts/2be0420ae6007013b7793fbbae4104df/
title: 375 - useHydration：把 DOM 差异变成可控状态
date: 2027-02-14T09:00:00+08:00
updated: 2027-02-14T09:00:00+08:00
author: cmdragon

summary:
  摘要：374 说水合不匹配根子是"两端算出的 HTML 不一样"，`useState` 是靠 payload 让两端拿同一份值。`useHydration` 是同一条同步通道更底层的手动挡：`useHydration(key, get, set)` 把 `get()` 在服务端（SSR 渲染完成时）跑一次、结果按 `key` 塞进 `nuxtApp.payload`，客户端（初始 Vue 实例创建时）用 `set(data)` 接回去——省掉重复计算和重复请求。官方点明它是高级组合式，主要在插件里用、多被模块作者用；要"全局响应式状态"该用 `useState` 而不是它。它的无 `useHydration` 等价写法，正好就是订 `app:rendered` 与 `app:created` 两个钩子（371）。

categories:
  - 前端开发

tags:
  - nuxt4
  - useHydration
  - payload
  - 水合
  - 插件
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

374 里让两端用同一份值，靠的是 `useState`。`useHydration` 是这条"服务端写、水合读"通道的**手动挡**——你自己给出"服务端怎么取、客户端怎么写回"，Nuxt 负责把它塞进 payload 再取出来。它给非响应式的外部状态（自己 new 出来的 store、类实例）做水合同步特别合适。

## 它做什么

官方定义：

> `useHydration` is a built-in composable that provides a way to set data on the server side every time a new HTTP request is made and receive that data on the client side. This way `useHydration` allows you to take full control of the hydration cycle.

机制句在 Usage 里：`get` 在服务端返回的数据，按第一个参数给的唯一 `key` 存进 `nuxtApp.payload`；水合时这数据在客户端被取回，**避免重复计算或重复请求**。

两条 `::note` 划定了它的定位：

- "This is an advanced composable, primarily designed for use within plugins, mostly used by Nuxt modules." —— 高级组合式，主要放插件里、多被模块作者用。
- "`useHydration` is designed to **ensure state synchronization and restoration during SSR**. If you need to create a globally reactive state that is SSR-friendly in Nuxt, `useState` is the recommended choice." —— 它管的是 SSR 期间的状态同步与还原；要**全局响应式**状态请用 `useState`。

## 签名与参数

```ts
export function useHydration<T> (key: string, get: () => T, set: (value: T) => void): void
```

| 参数 | 类型 | 说明（依文档） |
| --- | --- | --- |
| `key` | `string` | 标识这份数据的唯一键 |
| `get` | `() => T` | **只在服务端执行**（SSR 渲染完成时调用）设置初始值 |
| `set` | `(value: T) => void` | **只在客户端执行**（初始 Vue 实例创建时调用）接收数据 |

返回值：官方明写 "This composable does not return any value."（返回 `void`）。

`get`/`set` 的"只在某一端执行"和两端触发时机，正是 371 那两个钩子——服务端 `app:rendered`（SSR 渲染完）、客户端 `app:created`（实例建好）。

## 用起来

官方给的一组对照很说明问题。用 `useHydration`：

```ts
export default defineNuxtPlugin((nuxtApp) => {
  const myStore = new MyStore()

  useHydration(
    'myStoreState',
    () => myStore.getState(),
    data => myStore.setState(data),
  )
})
```

不用它、手写等价——你会发现它就是订那两个钩子：

```ts
export default defineNuxtPlugin((nuxtApp) => {
  const myStore = new MyStore()

  if (import.meta.server) {
    nuxt.hooks.hook('app:rendered', () => {
      nuxtApp.payload.myStoreState = myStore.getState()
    })
  }

  if (import.meta.client) {
    nuxt.hooks.hook('app:created', () => {
      myStore.setState(nuxtApp.payload.myStoreState)
    })
  }
})
```

`useHydration(key, get, set)` 把这段"服务端 `app:rendered` 往 `payload[key]` 写、客户端 `app:created` 从 `payload[key]` 读回"的样板收成一个调用。这就是它"完全掌控水合周期"的含义，也是它多被需要精确控制水合的模块作者用的原因。

## 什么时候不该用它

两条 `::note` 合起来就是选择依据：

- 要**响应式**的全局状态 → `useState`（374 用它替 `Math.random()` 那种）。`useHydration` 不做响应式，它只搬运一次数据。
- 只是想把某个非响应式对象（外部 store、类实例、`Map`/`Set` 之类）在服务端算好、水合时原样还原 → 用 `useHydration`。

## 自查

- `useHydration(key, get, set)`：服务端（`app:rendered`）跑 `get` 按 key 存进 payload，客户端（`app:created`）用 `set` 取回，避免重复计算/请求；返回 void。
- `get` 只服务端跑、`set` 只客户端跑——时机对应 371 的钩子。
- 它是高级组合式、主要在插件里、多被模块作者用。
- 要全局响应式状态用 `useState`，不是 `useHydration`；它只负责一次性搬运+还原非响应式状态。

## 下一篇

下一篇 376 讲 `<ClientOnly>` 与它的 `#fallback` 骨架屏：让一块内容干脆跳过服务端渲染、水合后才出现，从根上避开那些没法两边对齐的场景。

## 本文依据

- `useHydration` 定义、"set data on the server side every time a new HTTP request is made and receive that data on the client side"、两条 `::note`（advanced / plugins / modules；useState 更合适做响应式）、Usage 机制句（get 结果按 key 存 payload、水合取回避免重复）、签名 `useHydration<T>(key, get, set): void`、Parameters 表（get 只服务端 app:rendered 时、set 只客户端 app:created 时）、"does not return any value"、with/without useHydration 代码组（`api/composables/use-hydration`）。
