---
url: /posts/6af724096529faaf242837b7e1dc0445/
title: 212 - useAsyncData 两个重载：key 在前还是 handler 在前
date: 2026-12-21T11:00:00+08:00
updated: 2026-12-21T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：我第一次用 `useAsyncData` 是被它的参数顺序绕住的：一会儿看的是 `useAsyncData('users', () => ...)`，一会儿又看到 `useAsyncData(() => ...)`，到底有没有 key？官方 Type 一节给的答案是**两个重载**：一个是 `useAsyncData(handler, options?)`，一个是 `useAsyncData(key, handler, options?)`，返回类型都是 `AsyncData<DataT, DataE> & Promise<AsyncData<DataT, DataE>>`。第二个参数 `handler` 的类型不是普通函数，而是 `AsyncDataHandler<ResT> = (nuxtApp: NuxtApp, options: { signal: AbortSignal }) => Promise<ResT>`——**它收两个入参：Nuxt 实例与一个带 `signal` 的对象**，官方 Usage 示例因此写成 `(_nuxtApp, { signal }) => $fetch('/api/users', { signal })`，那个 `signal` 就是"这次请求可以被取消"的把手。`key` 的类型是 `MaybeRefOrGetter<string>`，官方专门有一节 "Reactive Keys" 演示用 computed 当 key（路由一变自动重取）。两个必须记住的约束都在 Parameters 里：`handler` "must return a truthy value (for example, it should not be `undefined` or `null`) **or the request may be duplicated on the client side**"，以及那条 warning——"The `handler` function should be **side-effect free** to ensure predictable behavior during SSR and CSR hydration. If you need to trigger side effects, use the `callOnce` utility to do so."选项表那 14 行本篇不展开，留给下一篇。

categories:
  - 前端开发

tags:
  - nuxt4
  - useAsyncData
  - handler
  - AbortSignal
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


211 末尾那条等价关系（`useFetch` = `useAsyncData` + `$fetch`）意味着这个函数是使用频率最高的一个。这一篇只解决"参数到底怎么排、每个参数是什么"。

## 两个重载，差别只在有没有 key

官方 Type 一节原文（照抄，两条函数签名 + 返回类型）：

```ts [Signature]
export type AsyncDataHandler<ResT> = (nuxtApp: NuxtApp, options: { signal: AbortSignal }) => Promise<ResT>

export function useAsyncData<ResT, DataE = unknown, DataT = ResT> (
  handler: AsyncDataHandler<ResT>,
  options?: AsyncDataOptions<ResT, DataT>,
): AsyncData<DataT, DataE> & Promise<AsyncData<DataT, DataE>>

export function useAsyncData<ResT, DataE = unknown, DataT = ResT> (
  key: MaybeRefOrGetter<string>,
  handler: AsyncDataHandler<ResT>,
  options?: AsyncDataOptions<ResT, DataT>,
): AsyncData<DataT, DataE> & Promise<AsyncData<DataT, DataE>>
```

三点读法：

- **`key` 在最前面**，所以"两个参数还是三个参数"取决于你有没有显式给 key，不是取决于选项。
- 返回值写成 `AsyncData<...> & Promise<AsyncData<...>>`——**它既是一个可 await 的 Promise，又是那个装着 `data`/`error`/`status` 的对象**。这就是 210 那句"不 await 也能用"在类型层面的依据：不 await 你拿到的还是同一个对象，只是里面的值还没填。
- 泛型有三个：`ResT`（handler 返回的原始类型）、`DataE`（错误，默认 `unknown`）、`DataT`（最终 `data` 的类型，默认等于 `ResT`）。`DataT` 与 `ResT` 能分开，正是给 `transform`/`pick` 用的（213 会看到那两行）。

官方 Usage 示例给的是**带 key 的三参数形态**：

```vue [app/pages/index.vue]
<script setup lang="ts">
const { data, status, pending, error, refresh, clear } = await useAsyncData(
  'mountains',
  (_nuxtApp, { signal }) => $fetch('https://api.nuxtjs.dev/mountains', { signal }),
)
</script>
```

## handler 的两个入参分别是什么

`AsyncDataHandler` 的类型定义说明了一切：第一个参数是 `nuxtApp`（Nuxt 实例），第二个是 `{ signal: AbortSignal }`。文档在 "Make Your `handler` Abortable" 一节给了人话版本：

> You can make your `handler` function abortable by using the `signal` provided in the second argument. This is useful for cancelling requests when they are no longer needed, such as when a user navigates away from a page. `$fetch` natively supports abort signals.

先解释一句 `AbortSignal`：它是一个"可以喊停"的对象，传给支持它的异步操作（`$fetch`、`AbortSignal` 事件），一旦 abort，那次操作就被取消。为什么取数需要这个？**因为用户可能在数据回来之前就走了**，不取消的话旧请求还在跑、还可能把结果写进已经不该有它的页面。

官方在这里给了一段很特别的示例——注释里写着连续调用 `refresh()` 的实际效果：

```ts [app/pages/index.vue]
const { data, error } = await useAsyncData(
  'users',
  (_nuxtApp, { signal }) => $fetch('/api/users', { signal }),
)

refresh() // will actually cancel the $fetch request (if dedupe: cancel)
refresh() // will actually cancel the $fetch request (if dedupe: cancel)
refresh()

clear() // will cancel the latest pending handler
```

注意第一行 `await` 之后，`refresh()`/`clear()` 是直接当函数调用的，**没有加 `error`/`data` 前缀**——`refresh` 是这一节示例里从解构中"消失"的那个成员（示例解构了 `data, error`，却直接调 `refresh()`）。这个形状在 214 讲返回值时我要专门对一次，先记下：示例代码本身不严谨，别照抄。

文档还列了 signal 会被 abort 的三种时机（原文三条）：

> The handler signal will be aborted when:
> - A new request is made with `dedupe: 'cancel'`
> - The `clear` function is called
> - The `options.timeout` duration is exceeded

另外两种取消写法也给了：给 `refresh` 传自己的 signal（`refresh({ signal: abortController.signal })` 配 `abortController?.abort()`），以及 handler 不支持 abort 时自己接住事件（`signal?.addEventListener('abort', () => reject(new Error('Request aborted')))`）。第二种能成立的前提是 handler 自己实现取消逻辑——这是文档给的手动方案，不是框架自动做的。

## key 可以是 computed：Reactive Keys 那一节

`key` 的类型是 `MaybeRefOrGetter<string>`（ref、getter、普通字符串都行），官方专门有一节：

> You can use a computed ref, plain ref or a getter function as the key, allowing for dynamic data fetching that automatically updates when the key changes:

```vue [app/pages/[id].vue]
<script setup lang="ts">
const route = useRoute()
const userId = computed(() => `user-${route.params.id}`)

// When the route changes and userId updates, the data will be automatically refetched
const { data: user } = useAsyncData(
  userId,
  () => fetchUserById(route.params.id),
)
</script>
```

这段注释"路由变化 ⇒ 自动重取"是我在 122 里引过的那条机制的另一处写法。这里还有个细节：**这个示例没有 `await`**（上一节的示例都有），说明响应式 key 的用法本来就不依赖首屏阻塞。

顺带把 watch 那条也摆这儿，因为两者常被混用：官方 "Watch Parameters" 一节的例子是 `'posts'` 固定 key + `{ watch: [page] }`，靠监听 ref 触发重取；而 Reactive Keys 靠的是 **key 本身变了**。差别在 215 讲 key 归属时会用到（key 决定数据存在 payload 的哪个格子，watch 决定什么时候再跑一次 handler）。

## 两条硬约束

第一条关于 handler 的返回值（Parameters 原文）：

- `handler`: an asynchronous function that must return a truthy value (for example, it should not be `undefined` or `null`) or the request may be duplicated on the client side.

我把这条当作踩坑预告：**接口真的返回 `null` 时，症状不是"数据为空"，而是客户端又发了一次请求**。`data` 的默认值与 `default` 选项的关系留到 213。

第二条是关于副作用的 warning：

> The `handler` function should be **side-effect free** to ensure predictable behavior during SSR and CSR hydration. If you need to trigger side effects, use the `callOnce` utility to do so.

"side-effect free" 的意思是 handler 里只做取数本身，不要顺手改全局状态、不要发第二个请求、不要写 cookie。原因和 210 讲的"同一份代码跑两遍"完全一致：跑两遍就意味着副作用发生两次。真要执行一次的动作，官方给的口子是 `callOnce`（那是 213 之后的另一篇）。

`options` 这一头本篇只给两句，剩下的 14 行给下一篇：

- `options` (object): Configuration for the asynchronous function call. **All options can be a static value, a `ref`, or a computed value.**

> All options can be given a `computed` or `ref` value. These will be watched and new requests made automatically with any new values if they are updated.

也就是说**选项变化会自动重发请求**，这条和 `watch` 是两套触发机制，213 逐项拆开讲。

## 名字本身是保留的

最后一条与"三参数"无关但必须抄：

> `useAsyncData` is a reserved function name transformed by the compiler, so you should not name your own function `useAsyncData`.

它和 117 讲的宏是同一类情况：`useAsyncData` 会被编译器处理，你再定义一个同名函数会撞上。

## 自查

1. `useAsyncData` 的两条签名差别在哪？（答：是否显式给 `key`，key 一定在最前面；返回类型两条相同。）
2. handler 的第二个参数是什么，三种会被 abort 的时机？（答：`{ signal }`，一个 `AbortSignal`；时机是 `dedupe: 'cancel'` 触发的新请求、调用 `clear`、超过 `options.timeout`。）
3. handler 返回 `null` 会怎样，为什么不能放副作用？（答："the request may be duplicated on the client side"；副作用会因两侧各跑一遍而执行两次，需要用 `callOnce`。）

## 下一篇

213 - useAsyncData 选项表逐条：14 个键与它们的默认值
