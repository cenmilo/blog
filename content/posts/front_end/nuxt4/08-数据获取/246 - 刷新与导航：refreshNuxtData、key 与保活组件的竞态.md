---
url: /posts/abd199edff755d7c1d0561e3a92cd831/
title: 246 - 刷新与导航：refreshNuxtData、key 与保活组件的竞态
date: 2027-01-02T09:00:00+08:00
updated: 2027-01-02T09:00:00+08:00
author: cmdragon

summary:
  摘要：这一篇讲"刷新"和"导航"撞在一起时的三件已记载事实与一处空白。**第一件：全局刷新不认识"当前是否显示"**——官方 note 原话："If your component is cached by `<KeepAlive>` and enters a deactivated state, the `asyncData` inside the component will still be refetched until the component is unmounted."（**生效边界是卸载，不是激活**）。**第二件：不传 key 就是全部重取**——"All `useAsyncData` and `useFetch` keys are re-fetched when no `keys` are explicitly specified."，而它返回的 Promise 有明确含义："returns a promise, resolving when all or specific `asyncData` instances have been refreshed."。**第三件：导航本身会清缓存**——236 引过 `purgeCachedData` 的定位句 "Whether to clean up Nuxt static and asyncData caches on route navigation." 与解释句 "This helps prevent memory leaks and ensures fresh data is loaded when needed."；响应式 key 换掉时官方也写了清理条件："the old data will be cleaned up **if no other components use it**"。**并发的两条规则**：`refresh` 默认"上一次没跑完会被等住"（"By default, Nuxt waits until a `refresh` is finished before it can be executed again."），`dedupe` 默认 `'cancel'` 且 `'cancel'` 时新请求会 abort 旧请求（signal aborted 三条时机之一）。**空白处我也写出来：导航进行中调用 `refreshNuxtData()` 会不会被路由切换打断、以及被 purge 掉的缓存与已在飞的请求如何互相影响，两节都没有句子记载**，我按未记载处理并给实测法。

categories:
  - 前端开发

tags:
  - nuxt4
  - 刷新
  - keepalive
  - 竞态
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


我遇到过的形状是这样的：列表页 `keepalive` 保活，切到详情页后我在别处调了一次 `refreshNuxtData()`，结果网络面板里出现了"那个看不见的列表页"的请求。**这不是幻觉，是官方写明的行为。**

## 第一条：全局刷新不看你是否在显示

`refreshNuxtData` 页那条 note 我逐字抄：

> If your component is cached by `<KeepAlive>` and enters a deactivated state, the `asyncData` inside the component will still be refetched until the component is unmounted.

拆成三个概念更清楚（这三个词在 137 讲 `key`/保活时区分过）：

| 状态 | 组件还在 | 会不会被重取 |
| --- | --- | --- |
| 激活（正在显示） | 是 | 会 |
| deactivated（保活但隐藏） | 是 | **会**（官方明写） |
| 已卸载 | 否 | 不会 |

所以"以为看不见的页面不会发请求"这个直觉在保活场景下不成立。**副作用同理**：如果那个隐藏页面的 handler 里有上报、写状态等动作，它也会再执行一次——尽管 212 的 warning 已经要求 handler 无副作用，这条 note 说明为什么必须遵守。

要控制作用范围，办法只有一个：**显式给 key**。

```ts
// you could also pass an array of keys to refresh multiple data
await refreshNuxtData(['count', 'user'])
```

## 第二条：不给参数是全站重取

两条原文放一起看：

| 项目 | 官方原话 |
| --- | --- |
| 参数 | "A single string or an array of strings as `keys` that are used to fetch the data. This parameter is **optional**. All `useAsyncData` and `useFetch` keys are re-fetched when no `keys` are explicitly specified." |
| 返回 | "`refreshNuxtData` returns a promise, resolving when all or specific `asyncData` instances have been refreshed." |

第二条实用在**它能 await 完**：官方两段示例都是这个形状（`refreshing.value = true` → `try { await refreshNuxtData(...) } finally { refreshing.value = false }`，按钮再配 `:disabled="refreshing"`）。**"刷新中"这件事由这个 Promise 决定，不是由请求发出决定**，所以 loading 态挂在它上面是准确的。

顺带一条边界：它覆盖的是四个 composable（`useAsyncData`、`useLazyAsyncData`、`useFetch`、`useLazyFetch`），而 229 记过 `clearNuxtData` 的说明只点了两个——**这两份覆盖范围文字不一致**，别当成同一个作用域。

## 第三条：导航自己会动缓存

这一条来自 236，但对"刷新时机"影响最直接：

> Whether to clean up Nuxt static and asyncData caches on route navigation.
>
> Nuxt will automatically purge cached data from `useAsyncData` and `nuxtApp.static.data`. This helps prevent memory leaks and ensures fresh data is loaded when needed, but it is possible to disable it.

再叠上响应式 key 的清理条件：

> // When userId changes, the data will be automatically refetched
> // and the old data will be cleaned up if no other components use it

**所以我做"刷新后回到旧页面希望还有数据"这类设计时，得先确认三件事**：那次刷新是不是跨了导航（跨了就可能已被 purge）、旧 key 有没有其它使用者、以及这次取数用的是 `payload` 还是 `static`（213 那条默认实现按 `nuxtApp.isHydrating` 分）。

## 并发：两条已写的规则

| 规则 | 官方原话 | 含义 |
| --- | --- | --- |
| 同一个 refresh 会等 | "By default, Nuxt waits until a `refresh` is finished before it can be executed again." | 连点不会堆请求 |
| `dedupe` 默认 `'cancel'` | "Policy when triggering an execution more than once at a time."（默认 `'cancel'`） | 同一时刻并发时的策略 |
| `'cancel'` 会打断旧请求 | "A new request is made with `dedupe: 'cancel'`"（signal aborted 三条时机之一） | 旧请求被 abort |

另外两个单次可调的旋钮在 `AsyncDataExecuteOptions` 里（`dedupe`、`timeout`、`signal`），也就是说**这一次刷新的策略可以临时改**：

```ts
refresh({ signal: abortController.signal })
```

`watch` 触发的那一类刷新还在 `enabled` 的管辖内（"While `false`, every execution is blocked (initial fetch, `execute`/`refresh`, and watch triggers)"），而 `getCachedData` 在这三种刷新路径上是否仍被采用由 `granularCachedData` 决定（236/240）。

## 一处空白：刷新撞上导航，文档没写

我把两页都翻过后确认：**没有任何一句描述"导航进行中调用 `refreshNuxtData()` 会怎样"**——会不会被路由切换取消、`await` 什么时候 resolve、以及 purge 与在飞请求谁先动，都没有记载。所以我给的只是测法，不是结论：

1. 造一个慢接口（服务端 `await new Promise(r => setTimeout(r, 2000))` 后返回）。
2. 页面 A 里 `refreshNuxtData('slow')` 不 await，立刻 `navigateTo('/b')`。
3. 观察三件事：网络里那次请求是否完成、`/b` 上再读 `useNuxtData('slow')` 拿到什么、以及控制台有没有出现"组件已卸载仍写状态"类警告。

这组实验同时能验证上面的"deactivated 仍被重取"是否在你的保活配置下成立。**测完把结论写进项目文档**，因为这块短期内不会被官方补上。

## 我的取舍

- 需要"跨页数据同步"⇒ 显式 key + 按 key 刷新（`refreshNuxtData(['a','b'])`），**不用无参版**。
- 需要"用户改完就重取"⇒ 优先手上实例的 `refresh()`（官方推荐"能拿实例就用实例方法"，229 引过）。
- 需要"离开页面就丢数据"⇒ 用 `clear()`（240 给过在路由 watch 里调的官方示例），别指望 purge 帮你清 `payload`。
- 有保活组件且它会取数 ⇒ 把它的取数当作"隐藏时也会跑"来设计副作用边界。

## 自查

1. 保活但隐藏的组件会不会被 `refreshNuxtData()` 重取？依据句？（答：会，官方说 "will still be refetched until the component is unmounted"。）
2. `refreshNuxtData` 返回的 Promise 什么时候 resolve？（答：全部或指定的 asyncData 实例刷新完成时。）
3. 文档没写清的是哪一处？我给出的测法关键在哪一步？（答：刷新与导航并发时的行为；关键是让接口足够慢，使导航必然发生在请求完成之前。）

## 下一篇

247 - 取消与超时：signal、timeout 与 dedupe 三件事

## 本文依据

- `api/utils/refresh-nuxt-data`：定位句（覆盖四个 composable）；`<KeepAlive>` deactivated note；`keys` 参数句与"不给键时全部重取"；返回值句；两段示例（`refreshing`、`try/finally`、`:disabled`、`['count', 'user']` 与数组注释）；"能拿实例就用 refresh/execute"那条 note（229 已逐句引）。
- `api/composables/use-async-data`：`refresh` 行"等上次跑完"、`AsyncDataExecuteOptions` 三键、signal aborted 三条时机、`enabled` 长说明、Shared State 两张清单（212/214/215/218 已逐句引）。
- `getting-started/data-fetching`：Reactive Keys 注释（old data cleaned up if no other components use it）、Refresh and execute 段、Clear 段（236/239/240 已逐句引）。
- `guide/going-further/experimental-features`：`purgeCachedData` 与 `granularCachedData` 两节（236 已逐句引）。
- `api/utils/clear-nuxt-data`：覆盖范围句与参数、返回 void（229 已逐句引）。
- `api/components/nuxt-page`、`api/utils/define-page-meta`：`keepalive` 的两处入口与 `KeepAliveProps` 类型（"By default, Nuxt does not enable `<Transition>` and `<KeepAlive>`..."、`keepalive` 的 type 行），用来界定"保活"在这套文档里指什么（092/119/135 已逐句引）。
