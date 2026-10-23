---
url: /posts/11b946b51306fb86fc9fb005f9e870a9/
title: 279 - payload：nuxtApp 读写服务端到客户端的数据
date: 2027-01-13T09:00:00+08:00
updated: 2027-01-13T09:00:00+08:00
author: cmdragon

summary:
  摘要：payload 是 Nuxt 从服务端带到客户端的那份数据——水合能接上、刷新不重复取数，靠的都是它。nuxtApp.payload 上有三个键：serverRendered（这次是不是服务端渲染的）、data（useFetch/useAsyncData 的取数结果缓存）、state（useState 存的状态）。这一篇按这三个键讲读写，重点补一段自定义序列化：默认 payload 用 devalue，支持不了的类型（比如 Luxon 的 DateTime）可以用 definePayloadPlugin + definePayloadReducer/Reviver 自己教 Nuxt 怎么存怎么还原。

categories:
  - 前端开发

tags:
  - nuxt4
  - payload
  - nuxtApp
  - 序列化
  - 水合
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


前面几篇反复提到"payload"这个词，这一篇正式讲它。`nuxtApp.payload` 就是那份"服务端算出来、随 HTML 发给浏览器、供水合时接手"的数据。你能通过它读、也能往里写自定义内容。

## payload 的定位与三个键

use-nuxt-app 页对它的定位：

> `payload` exposes data and state variables from server side to client side. The following keys will be available on the client after they have been passed from the server side:

三个键，官方逐个列了：

- `serverRendered` (boolean) - Indicates if response is server-side-rendered. —— 这次首屏是不是服务端渲染出来的。纯 `ssr: false` 的应用这里就是 `false`。你写"跨水合不重复执行"的判断（233、278）时，`payload.serverRendered` 就是常见的一半条件。
- `data` (object) - useFetch/useAsyncData 的取数结果。
- `state` (object) - useState 存的状态。

## data：取数结果的缓存

`data` 那个键装的是通过 `useFetch` / `useAsyncData` 拿到的数据，官方说明它带缓存作用：

> When you fetch the data from an API endpoint using either `useFetch` or `useAsyncData`, resulting payload can be accessed from the `payload.data`. This data is cached and helps you prevent fetching the same data in case an identical request is made more than once.

官方例子：`useAsyncData('count', ...)` 取回 `{ count: 1 }` 之后——

> After fetching the value of `count` using `useAsyncData` in the example above, if you access `payload.data`, you will see `{ count: 1 }` recorded there.

而且 `payload.data` 在服务端也能读——官方补了一句："When accessing the same `payload.data` from `ssrcontext`, you can access the same value on the server side as well."（把 279 和 280 的 `ssrContext.payload` 接上了）。这也解释了 210-249 那一路的机制：服务端取过的数，浏览器不再重复请求，就是因为结果在这儿、水合时直接读走。key（例子里的 `'count'`）就是它在 `data` 里的索引。

## state：useState 的落点

`state` 那个键装 `useState` 的数据，官方给了读取路径和一个插件示例：

> When you use `useState` composable in Nuxt to set shared state, this state data is accessed through `payload.state.[name-of-your-state]`.

```ts [app/plugins/my-plugin.ts]
export const useColor = () => useState<string>('color', () => 'pink')

export default defineNuxtPlugin((nuxtApp) => {
  if (import.meta.server) {
    const color = useColor()
  }
})
```

对新手这行的价值是：**你能亲眼看到 `useState('color')` 到底存在哪。** 259 那篇记过一句"文档未说明 useState 的值存在哪个对象"——这里给出了答案：`payload.state.color`（`[name-of-your-state]` 就是 key）。要排查"某个 useState 有没有进 payload、进的是不是你要的 key"，直接看 `nuxtApp.payload.state` 就行。

官方还紧跟一句：`payload` 支持比 JSON 更高级的类型——`ref`、`reactive`、`shallowRef`、`shallowReactive` 和 `NuxtError`。这和 235 讲的两道序列化对得上：接口→浏览器走 JSON，服务端→水合这份 payload 走 devalue，所以 ref/Date 这些能活着过去。

## 自定义序列化：reducer / reviver

payload 默认支持不了的类型（典型是某个类的实例，比如 Luxon 的 `DateTime`），官方从 v3.4 起给了自定义口子，叫 reducer（存的时候怎么压平）/ reviver（读的时候怎么还原）：

> Since Nuxt v3.4, it is possible to define your own reducer/reviver for types that are not supported by Nuxt.

完整例子（注释是官方的，我译了要点）：

```ts [app/plugins/date-time-payload.ts]
/**
 * 这类插件在 Nuxt 生命周期里非常早运行，在 payload 被还原之前。
 * 此时你拿不到 router 或其它 Nuxt 注入的属性。
 * 注意 "DateTime" 这个字符串是类型标识，reducer 和 reviver 两边必须一致。
 */
export default definePayloadPlugin((nuxtApp) => {
  definePayloadReducer('DateTime', (value) => {
    return value instanceof DateTime && value.toJSON()
  })
  definePayloadReviver('DateTime', (value) => {
    return DateTime.fromISO(value)
  })
})
```

三个要点都在官方注释里，别漏：

1. 用的是 `definePayloadPlugin`（不是普通 `defineNuxtPlugin`），它 "runs very early ... before we revive the payload"。
2. 因此在这个插件里 "You will not have access to the router or other Nuxt-injected properties."——别指望 `useRouter()` 之类。
3. 那个类型标识字符串（例子里的 `'DateTime'`）**必须 reducer 和 reviver 两边一模一样**，否则还原时对不上。

这条正是 252 篇 useState 排错里 `Cannot stringify arbitrary non-POJOs` 的解法——存进不了默认序列化的类实例，就在这里给这个类配一对 reducer/reviver。

## 读写心智

把 payload 放回整章主线：

- **读**：`nuxtApp.payload.data[key]`（取数结果）、`nuxtApp.payload.state[key]`（useState 值）、`nuxtApp.payload.serverRendered`（是否 SSR）。
- **写自定义内容**：默认类型够用时不用碰；要支持自定义类，走 `definePayloadPlugin` + reducer/reviver。
- **敏感信息别进它**（264）：payload 是发给浏览器的明文，私钥别 `useState`、别渲染。

## 自查

- 想知道某个 `useState('x')` 落没落进 payload → 看 `nuxtApp.payload.state.x`；取数结果 → `payload.data.<key>`（前提是你给了显式 key）。
- 要往 payload 存一个自定义类的实例 → 配 `definePayloadPlugin` + `definePayloadReducer`/`definePayloadReviver`，两边类型标识字符串一致。
- payload 插件里不要用 router/注入属性——它跑得太早。
- payload 是浏览器可见明文，别塞敏感值。

## 下一篇

下一篇 280 讲 `nuxtApp.ssrContext`：那份只在服务端存在、带 `url`/`event`/`payload` 三个属性的上下文，以及它在插件/服务端里能干什么。
