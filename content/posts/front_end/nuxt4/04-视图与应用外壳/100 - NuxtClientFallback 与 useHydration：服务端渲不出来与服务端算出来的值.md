---
url: /posts/ee99cb753e901a333ab450fbcf045077/
title: 100 - NuxtClientFallback 与 useHydration：服务端渲不出来与服务端算出来的值
date: 2026-11-14T10:00:00+08:00
updated: 2026-11-14T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇讲两件都在"服务端渲染与水合"这条线上、但方向相反的机制。第一件是 `<NuxtClientFallback>`：官方的定义是"在其任何子节点在 SSR 中触发错误时，把内容渲染到客户端"，它是**实验特性**——官方明确写着"这个组件是实验性的，要使用它你必须在 `nuxt.config` 里启用 `experimental.clientFallback` 选项"，实验特性页的对应说法是"启用实验性的 `<NuxtClientFallback>` 组件，以便在 SSR 中出错时在客户端渲染内容"。它对外有三格 prop（`placeholderTag`/`fallbackTag` 类型 `string`、默认 `div`；`placeholder`/`fallback` 类型 `string`；`keepFallback` 类型 `boolean`、默认 `false`）、一个事件（`@ssr-error`，官方注明"这只会在服务端触发"）和一个插槽（`#fallback`，"指定在插槽于服务端渲染失败时要显示的内容"）。这里有一条安全警告必须整句记住：**`placeholder` 和 `fallback` 这两个 prop 是按原始 HTML 渲染内容的，不要把不可信的用户输入传给它们，否则可能造成 XSS 漏洞；需要正确转义的动态内容请改用 `#fallback` 或 `#placeholder` 插槽。** 第二件是 `useHydration`：官方定义是"允许完全控制水合周期，以从服务端设置和接收数据"，签名是 `useHydration<T>(key: string, get: () => T, set: (value: T) => void): void`——**没有返回值**，参数表里 `get` 是"**只在服务端执行**（在 SSR 渲染完成时调用）以设置初始值"、`set` 是"**只在客户端执行**（在创建初始 Vue 实例时调用）以接收数据"，服务端 `get` 返回的数据会以第一个参数给的唯一 key 存进 `nuxtApp.payload`，水合时在客户端被取回，从而避免冗余的计算或 API 调用。官方还给了两条定位：这是**高级**组合式函数，主要在插件内使用、大多由 Nuxt 模块使用；它的设计目标是**在 SSR 期间保证状态同步与恢复**，如果你需要的是 SSR 友好的全局响应式状态，官方推荐的选择是 `useState`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 水合
  - clientfallback
  - usehydration
  - 实验特性

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


服务端渲染有两个方向的麻烦：**渲不出来的内容**怎么办，以及**渲出来的数据**怎么交给浏览器。前者对应 `<NuxtClientFallback>`，后者对应 `useHydration`。这两个都在水合这条线上，但一个是实验特性、一个是高级接口，用法差别很大。

## 一、`<NuxtClientFallback>`：SSR 出错时把这一块交给客户端

官方定义：**Nuxt 提供 `<NuxtClientFallback>` 组件，在其任何子节点在 SSR 中触发错误时，把内容渲染到客户端。**

它和 `<NuxtErrorBoundary>` 的分工正好相反，这张表要用错一次就会白折腾：

| | `<NuxtErrorBoundary>` | `<NuxtClientFallback>` |
| --- | --- | --- |
| 处理的错误发生在 | **客户端**（默认插槽内） | **SSR 阶段**（其子节点触发错误） |
| 兜底方式 | 渲染 `#error` 插槽 | 把那一块内容交给客户端渲染 |
| 是否要开关 | 直接可用 | **实验特性，必须先启用** |
| 事件 | `@error` | `@ssr-error`（只在服务端触发） |

### 先开开关，否则组件不生效

官方在组件页的第一条 note 就是门槛："**这个组件是实验性的，为了使用它，你必须在你的 `nuxt.config` 中启用 `experimental.clientFallback` 选项。**"

```ts
export default defineNuxtConfig({
  experimental: {
    clientFallback: true,
  },
})
```

实验特性页对这一项的描述与组件页一致："启用实验性的 `<NuxtClientFallback>` 组件，以便在 SSR 中出错时在客户端渲染内容。"

### 基本用法

官方的示例把"会坏的组件"和"正常的组件"放在一起，说明它是**按块兜底**，不是整页降级：

```vue [app/pages/example.vue]
<template>
  <div>
    <Sidebar />
    <!-- this component will be rendered on client-side -->
    <NuxtClientFallback fallback-tag="span">
      <Comments />
      <BrokeInSSR />
    </NuxtClientFallback>
  </div>
</template>
```

### 三格 prop 与一条 XSS 警告

| prop | 官方说明 | 类型 | 默认 |
| --- | --- | --- | --- |
| `placeholderTag` 或 `fallbackTag` | 指定插槽在服务端渲染失败时要渲染的后备标签 | `string` | `div` |
| `placeholder` 或 `fallback` | 指定插槽渲染失败时要渲染的后备内容 | `string` | —— |
| `keepFallback` | 如果它在服务端渲染失败，保留后备内容 | `boolean` | `false` |

官方在这一节后面挂了一条安全警告，值得逐字读：

> `placeholder` 和 `fallback` prop 会把内容按**原始 HTML** 渲染。不要把不可信的用户输入传给这些 prop，它可能导致 XSS 漏洞。对于需要正确转义的动态内容，请改用 `#fallback` 或 `#placeholder` **插槽**。

所以选择规则是：**内容里带用户输入 → 只能用插槽；只有静态文案 → 才能用 prop。**

```vue
<template>
  <!-- render <span>Hello world</span> server-side if the default slot fails to render -->
  <NuxtClientFallback
    fallback-tag="span"
    fallback="Hello world"
  >
    <BrokeInSSR />
  </NuxtClientFallback>
</template>
```

`#fallback` 插槽的官方说明：指定在插槽于服务端渲染失败时要在服务端显示的内容。

```vue
<template>
  <NuxtClientFallback>
    <!-- ... -->
    <template #fallback>
      <!-- this will be rendered on server side if the default slot fails to render in ssr -->
      <p>Hello world</p>
    </template>
  </NuxtClientFallback>
</template>
```

注意这条注释：**`#fallback` 里的内容也是服务端渲染的**——它是那块内容在 SSR 阶段的替身，不是客户端加载后的样子。

### `@ssr-error` 只在服务端触发

官方的事件说明："`@ssr-error`：当某个子节点在 SSR 中触发错误时发出的事件。**注意，这只会在服务端触发。**"

```vue
<template>
  <NuxtClientFallback @ssr-error="logSomeError">
    <!-- ... -->
  </NuxtClientFallback>
</template>
```

这条限制决定了它的用途是**服务端记日志**，不是给用户弹提示：浏览器里这个事件永远不会有。

## 二、`useHydration`：把服务端算出来的值带到客户端

官方定义："允许完全控制水合周期，以设置和接收来自服务端的数据。"再具体一层：`useHydration` 是一个内置组合式函数，提供了一种**在每次新的 HTTP 请求时于服务端设置数据、并在客户端接收该数据**的方式。

数据流向官方写得很清楚：**服务端 `get` 函数返回的数据，会以作为第一个参数提供的唯一 key，存进 `nuxtApp.payload`。在 hydration 期间，这份数据随后在客户端被取回，从而防止冗余的计算或 API 调用。**

### 签名与参数：两侧各只跑一边

```ts
export function useHydration<T> (key: string, get: () => T, set: (value: T) => void): void
```

| 参数 | 类型 | 官方说明 |
| --- | --- | --- |
| `key` | `string` | 在你的 Nuxt 应用中标识这份数据的**唯一** key |
| `get` | `() => T` | **只在服务端执行**的函数（在 SSR 渲染完成时调用），用于设置初始值 |
| `set` | `(value: T) => void` | **只在客户端执行**的函数（在创建初始 Vue 实例时调用），用于接收数据 |

官方在"Return Values"一节只写了一句：**这个组合式函数不返回任何值。** 所以任何指望 `useHydration(...)` 返回结果的写法都是错的，数据是通过 `set` 回调交出来的。

### 官方给的前后对比

这一组对比最能说明它省掉了什么：

```ts [With useHydration]
export default defineNuxtPlugin((nuxtApp) => {
  const myStore = new MyStore()

  useHydration(
    'myStoreState',
    () => myStore.getState(),
    data => myStore.setState(data),
  )
})
```

```ts [Without useHydration]
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

不用它，就得自己判 `import.meta.server` / `import.meta.client`、自己在 `app:rendered` 写入 payload、自己在 `app:created` 读回来；用它，这三件事变成"一个 key + 两个函数"。注意 `get` 对应的正是 `app:rendered`（官方说"在 SSR 渲染完成时调用"），`set` 对应的正是 `app:created`（"在创建初始 Vue 实例时调用"）。

### 两条官方定位，别把它当状态管理

- "**这是一个高级组合式函数，主要设计用于在插件内使用，大多由 Nuxt 模块使用。**"
- "`useHydration` 被设计用来**在 SSR 期间保证状态同步与恢复**。如果你需要一个 SSR 友好的全局响应式状态，官方推荐的选择是 `useState`。"

第二条是选型判据：你要的是"让同一个外部状态实例（比如某个 store 对象）在服务端和客户端拿到同一份值"，用 `useHydration`；你要的是"Nuxt 的一份跨导航的响应式状态"，用 `useState`。

## 三、两者什么时候会碰到一起

官方没有把这两个写成一套 API，但它们管的是同一根线上的两头：

| 情况 | 用到的机制 |
| --- | --- |
| 服务端渲染某个子组件时抛错，想让这块在浏览器里补渲出来 | `<NuxtClientFallback>`（需先开 `experimental.clientFallback`） |
| 服务端已经算好的值，不想让浏览器再算一遍或再请求一次 | `useHydration` |
| 一块内容在 SSR 阶段就交给客户端了，那它就没有服务端那次取数 | 这块内容要自己拿数据——它享受不到 `get` 在服务端那一次的结果 |
| 想把错误只记在服务端日志里 | `@ssr-error`（客户端不会触发） |
| 需要在 SSR 期间同步一个非响应式的外部 store | `useHydration`（响应式全局状态改用 `useState`） |

第三行就是把两者连起来的那条推理：**客户端兜底的那一块，等于放弃了服务端那次渲染的产物**，所以那一块里的数据只能靠浏览器自己取——这正是它们该一起想的原因。

## 自查

1. `<NuxtClientFallback>` 兜的是哪一侧的错误？用它之前必须做什么？
2. `fallback` prop 和 `#fallback` 插槽，哪个能放用户输入？依据是哪一条警告？
3. `@ssr-error` 在哪一侧触发？这对它的用途意味着什么？
4. `useHydration` 的三个参数分别在哪一侧、什么时机执行？它有返回值吗？
5. 需要 SSR 友好的全局响应式状态时，官方推荐哪个？为什么不是 `useHydration`？

## 下一篇

101 讲 `<NuxtRouteAnnouncer>`：它放在哪里、`politeness` 与 `atomic` 的默认值，以及为什么页内变化的播报要改用 `<NuxtAnnouncer>` 加 `useAnnouncer`。

## 本文依据

- `https://nuxt.com/docs/4.x/api/components/nuxt-client-fallback`
  - 原文："Nuxt provides the `<NuxtClientFallback>` component to render its content on the client if any of its children trigger an error in SSR."
  - 原文（note）："This component is experimental and in order to use it you must enable the `experimental.clientFallback` option in your `nuxt.config`."
  - 原文示例（`app/pages/example.vue`）：`<Sidebar />`、`<!-- this component will be rendered on client-side -->`、`<NuxtClientFallback fallback-tag="span"><Comments /><BrokeInSSR /></NuxtClientFallback>`
  - 原文（Events）："`@ssr-error`: Event emitted when a child triggers an error in SSR. Note that this will only be triggered on the server."；示例 `<NuxtClientFallback @ssr-error="logSomeError">`
  - 原文（Props）："`placeholderTag` | `fallbackTag`: Specify a fallback tag to be rendered if the slot fails to render on the server."（type `string`，default `div`）；"`placeholder` | `fallback`: Specify fallback content to be rendered if the slot fails to render."（type `string`）；"`keepFallback`: Keep the fallback content if it failed to render server-side."（type `boolean`，default `false`）
  - 原文（warning）："The `placeholder` and `fallback` props render content as raw HTML. Do not pass untrusted user input to these props as it may lead to XSS vulnerabilities. Use the `#fallback` or `#placeholder` slots instead for dynamic content that needs proper escaping."
  - 原文示例：`<!-- render <span>Hello world</span> server-side if the default slot fails to render -->` 配 `fallback-tag="span"` 与 `fallback="Hello world"`
  - 原文（Slots）："`#fallback`: specify content to be displayed server-side if the slot fails to render."；示例注释 `<!-- this will be rendered on server side if the default slot fails to render in ssr -->` 与 `<p>Hello world</p>`
  - 原文 Source：`client-fallback.client.ts` 与 `client-fallback.server.ts`（客户端与服务端两份实现）
- `https://nuxt.com/docs/4.x/guide/going-further/experimental-features`
  - 原文（clientFallback 节）："Enables the experimental `<NuxtClientFallback>` component for rendering content on the client if there's an error in SSR."；示例 `experimental: { clientFallback: true }`
- `https://nuxt.com/docs/4.x/api/composables/use-hydration`
  - 原文（描述）："Allows full control of the hydration cycle to set and receive data from the server."
  - 原文："`useHydration` is a built-in composable that provides a way to set data on the server side every time a new HTTP request is made and receive that data on the client side. This way `useHydration` allows you to take full control of the hydration cycle."
  - 原文（note）："This is an advanced composable, primarily designed for use within plugins, mostly used by Nuxt modules."
  - 原文（note）："`useHydration` is designed to **ensure state synchronization and restoration during SSR**. If you need to create a globally reactive state that is SSR-friendly in Nuxt, `useState` is the recommended choice."
  - 原文（Usage）："The data returned from the `get` function on the server is stored in `nuxtApp.payload` under the unique key provided as the first parameter to `useHydration`. During hydration, this data is then retrieved on the client, preventing redundant computations or API calls."
  - 原文（Type）：`export function useHydration<T> (key: string, get: () => T, set: (value: T) => void): void`
  - 原文（Parameters 表）：`key` — "A unique key that identifies the data in your Nuxt application."；`get` — "A function executed **only on the server** (called when SSR rendering is done) to set the initial value."；`set` — "A function executed **only on the client** (called when initial Vue instance is created) to receive the data."
  - 原文（Return Values）："This composable does not return any value."
  - 原文（code-group）：使用 `useHydration('myStoreState', () => myStore.getState(), data => myStore.setState(data))` 与不使用时的 `if (import.meta.server) { nuxt.hooks.hook('app:rendered', ...) }` / `if (import.meta.client) { nuxt.hooks.hook('app:created', ...) }` 对照
- `https://nuxt.com/docs/4.x/api/components/nuxt-error-boundary`
  - 对照用（见 095）："The \<NuxtErrorBoundary> component handles client-side errors happening in its default slot."；"`@error`: Event emitted when the default slot of the component throws an error."
