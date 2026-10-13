---
url: /posts/22c9f9daf0e7894f39db8651b86a8581/
title: 262 - useHydration：自己接管那一次数据传递
date: 2027-01-07T10:00:00+08:00
updated: 2027-01-07T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：100 与 261 都提过它，这一篇只讲这一节的全部原文并把它用透。定位句是 "Allows full control of the hydration cycle to set and receive data from the server."，正文那句更具体："useHydration is a built-in composable that provides a way to **set data on the server side every time a new HTTP request is made and receive that data on the client side**."，随后一句 "This way `useHydration` allows you to take full control of the hydration cycle."。两条 note 划边界：它是 "an advanced composable, primarily designed for use within plugins, mostly used by Nuxt modules."，而"**If you need to create a globally reactive state that is SSR-friendly in Nuxt, `useState` is the recommended choice.**"。机制在 Usage 段：「The data returned from the `get` function on the server is stored in **`nuxtApp.payload` under the unique key provided as the first parameter** to `useHydration`. During hydration, this data is then retrieved on the client, **preventing redundant computations or API calls**.」签名一行没有重载：`useHydration<T> (key: string, get: () => T, set: (value: T) => void): void`，**返回 `void`**。参数表两行是这一节最有信息量的地方：`get` 是 "A function executed **only on the server** (**called when SSR rendering is done**) to set the initial value."，`set` 是 "A function executed **only on the client** (**called when initial Vue instance is created**) to receive the data."。这两句时机也解释了为什么"Without useHydration"那段对照示例要挂 `app:rendered` 与 `app:created` 两个钩子——而那段示例自身有一处笔误（回调参数写成 `nuxtApp`，钩子却调 `nuxt.hooks.hook(...)`），我照抄并标明不可运行。

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


什么时候才轮到自己接管水合？答案是**当你手上那个东西不是 ref、框架看不见它**的时候——比如一个类实例、一个第三方 store。这一页就是为这种情况写的。

## 整页四段，逐段抄

第一段（它做什么）：

> `useHydration` is a built-in composable that provides a way to set data on the server side every time a new HTTP request is made and receive that data on the client side. This way `useHydration` allows you to take full control of the hydration cycle.

注意"every time a new HTTP request is made"——**服务端每次请求都会跑 `get`**，不是进程级一次。

两条 note（边界与退路）：

> This is an advanced composable, primarily designed for use within plugins, mostly used by Nuxt modules.

> `useHydration` is designed to **ensure state synchronization and restoration during SSR**. If you need to create a globally reactive state that is SSR-friendly in Nuxt, [`useState`] is the recommended choice.

第二段（机制与落点）：

> The data returned from the `get` function on the server is stored in `nuxtApp.payload` under the unique key provided as the first parameter to `useHydration`. During hydration, this data is then retrieved on the client, preventing redundant computations or API calls.

这一句给了三件事：**存在 `nuxtApp.payload`**、**键就是你传的第一个参数**、**目的是省掉重复计算或重复请求**。

签名与返回：

```ts [Signature]
export function useHydration<T> (key: string, get: () => T, set: (value: T) => void): void
```

| 参数 | 官方说明 |
| --- | --- |
| `key` | "A unique key that identifies the data in your Nuxt application." |
| `get` | "A function executed **only on the server** (called when SSR rendering is done) to set the initial value." |
| `set` | "A function executed **only on the client** (called when initial Vue instance is created) to receive the data." |

Return Values 一句："This composable does not return any value."（与签名末尾的 `: void` 一致，不像 `useHead` 那样自相矛盾——187/195 记过那条。）

## 参数表那两句时机，值回整页票价

`get` 与 `set` 的括号里各写了一个时机名：**SSR rendering is done** 与 **initial Vue instance is created**。这两个名字对应的是钩子事件（232 的生命周期清单里都有：服务端渲染完调 `app:rendered`、客户端实例建好调 `app:created`）。

官方那对示例正是把这件事摊开给你看：

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

**这段"手写版"里有一处官方笔误**：插件回调参数命名是 `nuxtApp`，可两处钩子写的是 `nuxt.hooks.hook(...)`，而 `nuxt` 在那个作用域里并不存在。我照抄是提醒你**别把这段当可运行代码**（199 也记过一次这个笔误）；真要手写等价版，钩子应该挂在插件拿到的那个实例上。

从这个对照我能确认的唯一结论是：**`useHydration` 内部就是这对时机 + 同一个 payload 键**，所以它的行为可以按"服务端渲染完写、客户端实例创建时读"来推。至于"如果 payload 里没有这个键，`set` 会被调用吗、参数是什么"，**这一页一个字都没写**（下面第三节给测法）。

## 我什么时候真的会用它

| 情况 | 用什么 | 为什么 |
| --- | --- | --- |
| 普通响应式共享状态 | `useState` | 官方明说推荐它 |
| 首屏要渲染的数据 | `useAsyncData`/`useFetch` | 它们自动进 payload（210） |
| 一次性副作用（第三方初始化） | `callOnce` | 它管执行次数（230） |
| **自己的类实例/store 要跨水合** | `useHydration` | 框架不会替你序列化它 |

第四行有个硬前提我得先划出来：**你的 store 状态必须能被 payload 承载**。235 记过 payload 用 `devalue` 序列化，支持的是正则/`Date`/`Map`/`Set`/`ref` 等，**自定义类实例需要 reducer/reviver**（`definePayloadPlugin`，252 那条）。所以 `myStore.getState()` 返回的必须是普通数据；如果它返回一个类实例，光加 `useHydration` 不够。

## 三处未说明，给三个测法

| 我要确认的 | 测法 |
| --- | --- |
| 键冲突会怎样 | 在两个插件里都用 `useHydration('same', ...)` 写不同形状，看服务端日志与客户端读到的值（对照 215 讲的共享语义） |
| 首次进入无 payload 时 `set` 是否被调用 | 在 `set` 里打印参数，用一个纯客户端导航进入的页面测 |
| 与 `app:created` 的先后是否影响读 DOM | 231 讲的水合前脚本、199 讲的"库要在水合完成后初始化"都跟这个时机有关，一次 `onMounted` 对照即可判断自己要不要再往后挪 |

这三条都不是文档能答的，所以我没有把它们写成结论。

## 自查

1. `get` 与 `set` 各自在什么时候跑？（答：`get` 只在服务端、SSR 渲染完成时；`set` 只在客户端、初始 Vue 实例创建时。）
2. 数据落在哪里、键由谁定？（答：`nuxtApp.payload`，键就是 `useHydration` 的第一个参数。）
3. 为什么它比 `useState` 更窄？（答：官方定位它是 advanced、主要在插件/模块里用，而"SSR 友好的全局响应式状态"明确推荐 `useState`。）

## 下一篇

263 - 状态与导航：restoreState 实验开关

## 本文依据

- `api/composables/use-hydration`：定位句；正文两句；两条 note（advanced/插件/模块；与 `useState` 的分工句）；Usage 段三句（payload、unique key、preventing redundant computations or API calls）；code-group 两段示例（With / Without，含 `nuxt.hooks.hook(...)` 与回调参数 `nuxtApp` 不一致这处笔误）；签名 `useHydration<T> (key, get, set): void`；Parameters 三行（含两处时机括号）；Return Values 句。
- `guide/concepts/nuxt-lifecycle`：`app:rendered`（服务端渲染完）与 `app:created`（插件之后）两个钩子的位置（232 已逐句引）。
- `api/composables/use-nuxt-app`：payload 一节与自定义 reducer/reviver、`definePayloadPlugin`（100/252 已逐句引）。
- `api/composables/use-state`：官方推荐位与序列化边界（251/252 已逐句引）。
- `api/utils/call-once`：一次性副作用的定位与"不返回值"（230 已逐句引）。
- `getting-started/data-fetching`：取数函数自动进 payload 的机制句（210 已逐句引）。
