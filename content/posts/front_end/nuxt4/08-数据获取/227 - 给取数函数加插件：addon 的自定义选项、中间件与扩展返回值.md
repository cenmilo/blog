---
url: /posts/1b6c0524f35a56d19ae9243b2e941df5/
title: 227 - 给取数函数加插件：addon 的自定义选项、中间件与扩展返回值
date: 2026-12-26T11:00:00+08:00
updated: 2026-12-26T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：addon 是 226 那两个工厂函数的扩展机制，官方定位句是 "define a reusable extension for `useFetch` composables created with `createUseFetch`"，能力清单四样："An addon can declare custom call-site options, wrap the handler with middleware, attach custom reactive logic or extend the returned object."。执行时机与顺序两条要记：「The addon's `setup` function runs on every call of the composable, and **receives the merged options (factory defaults plus caller options)**」，以及「Addons run in the order of the `addons` array, with **middleware from the first addon as the outermost wrapper**. Listing the same addon object more than once **runs it only once**」。自定义选项靠给 `setup` 的参数标注 `UseFetchAddonOptions<{ refreshOnFocus?: boolean }>` 来声明，"The options become part of the created composable's signature, fully typed for callers"——但有一句必须看到：**"Custom options stay in the merged options object, so they are also passed to `$fetch` along with the request options, which ignores keys it does not recognise."**（自定义键会一路带进 `$fetch`，靠它忽略未知键才不会出事）。扩展返回值的方式是 `setup` **返回一个函数**，该函数收到 asyncData 实例、它返回的任意对象被合进组合式的返回值。**最容易踩的是缓存那一格**：「Custom options are **not** part of the auto-generated key by default. If a custom option affects the response, provide a `key` resolver...」，还要配一条 TypeScript 顺序警告"Declare `setup` before `key`"。

categories:
  - 前端开发

tags:
  - nuxt4
  - addon
  - 中间件
  - 自定义选项
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先给一句人话定位：addon 就是**把"每个页面都要重复写的那点取数逻辑"做成一个可复用的小插件**，插进 226 造出来的组合式上。两节规则相同，`useAsyncData` 那侧的说明甚至直接写"Addons for useAsyncData work the same way as useFetch addons."。

## 最小骨架：一个开关式的 addon

官方第一个例子把四种能力里的三种串在了一起：

```ts [app/composables/useCustomFetch.ts]
const refreshOnFocus = defineUseFetchAddon({
  // augment the call-site options for the custom useFetch instance
  setup: (options: UseFetchAddonOptions<{ refreshOnFocus?: boolean }>) => {
    // run code *before* calling `useAsyncData` in `useFetch`
    if (import.meta.server || !options.refreshOnFocus) { return }

    return (asyncData) => {
      // run code *after* calling `useAsyncData` in `useFetch`
      const focused = useWindowFocus()
      watch(focused, (focused) => {
        if (focused) { asyncData.refresh() }
      })

      return { focused } // extend the returned object with a new property
    }
  },
})

export const useCustomFetch = createUseFetch({ addons: [refreshOnFocus] })
```

调用侧那个新选项是**有类型的**：

```vue [app/pages/index.vue]
<script setup lang="ts">
// `refreshOnFocus` is typed on the created composable
const { data } = await useCustomFetch(
  'https://api.nuxtjs.dev/mountains',
  { refreshOnFocus: true },
)
</script>
```

读这段有四个层次：

1. `setup` 的参数标注 `UseFetchAddonOptions<{ 自定义选项 }>` ——**这就是声明自定义选项的方式**，官方解释是 "Declare custom options by annotating the `setup` `options` parameter with `UseFetchAddonOptions<{ ... }>`."，并给出后果："The options become part of the created composable's signature, fully typed for callers."
2. **早期返回 `return`（无值）= 这个 addon 这次不参与**，官方注释写的是"run code *before* calling `useFetch` 里的 useAsyncData"。
3. `setup` **返回一个函数**，那个函数在实例创建之后跑（注释 "run code *after* calling `useAsyncData` in `useFetch`"）。
4. 函数返回的对象被合进返回值（`focused` 因此能用 `useCustomFetch` 的解构拿到）。

时机与合并规则，官方一句话讲完：

> The addon's `setup` function runs on every call of the composable, and receives the merged options (factory defaults plus caller options).

顺序规则有两条，第二条我起初没看懂它的用意：

> Addons run in the order of the `addons` array, with middleware from the first addon as the outermost wrapper. Listing the same addon object more than once runs it only once.

**同一个 addon 对象重复列只跑一次**——所以数组里塞两遍同一个不是"跑两次"，是无效重复。

## 自定义选项会一路带进 $fetch

这一句必须抄，因为它是我唯一能解释"为什么自定义键不会把请求搞坏"的依据：

> Custom options stay in the merged options object, so they are also passed to `$fetch` along with the request options, which ignores keys it does not recognise.

也就是：**你的 `refreshOnFocus` 会跟着请求选项进 `$fetch`，只是 ofetch 不认的键会被忽略**，所以没出事。这不算设计保证，官方描述的是现状——**如果哪天目标实现严格校验选项，这些自定义键就有风险**。真要安全，就把自定义选项收在 addon 内部消化（下面 `auth` 那个例子就是这么做的）。

## 改请求：往数组里 push

官方第二个例子（鉴权）演示了**在 setup 早期直接改选项**：

```ts
const auth = defineUseFetchAddon({
  setup: (options: UseFetchAddonOptions<{ auth?: MaybeRefOrGetter<boolean> }>) => {
    const { token } = useTokenStore()
    options.auth ??= true
    options.onRequest.push(({ options: fetchOptions }) => {
      if (!toValue(options.auth)) { return }

      fetchOptions.headers.set('Authorization', `Bearer ${token.value}`)
    })
  },
})
```

调用侧就能按次关掉：`useCustomFetch('/public-endpoint', { auth: false })`（注释写 "`auth` is typed"）。

这里有个我原来不知道的能力：**`options.middleware`、`onRequest`、`onRequestError`、`onResponse`、`onResponseError` 都是可以直接 push 的数组**。官方在"包装 then/catch/finally"那节开头确认了这一点：

> `middleware`, `onRequest`, `onRequestError`, `onResponse` and `onResponseError` are always normalized to arrays...

`toValue(options.auth)` 的写法也说明**自定义选项可以是 ref 或 getter**（类型就是 `MaybeRefOrGetter<boolean>`）。

## 中间件：给请求加下限

`Middleware` 一节的定义句与例子：

> Middleware wraps the execution of the request handler. Call `next()` to continue the chain (and get the resolved data), or throw to abort. Middleware from the first addon is the outermost wrapper:

```ts
const minDuration = defineUseFetchAddon({
  setup: (options: UseFetchAddonOptions<{ minDuration?: number }>) => {
    options.middleware.push(async (next) => {
      const [result] = await Promise.all([
        next(),
        new Promise(resolve => setTimeout(resolve, toValue(options.minDuration) ?? 300)),
      ])
      return result
    })
  },
})
```

这个用途很妙：**保证请求至少花 300 毫秒**，这样加载状态不会一闪而过。它靠的就是 218 讲的 `middleware` 链（`next()` 放行、throw 中止），addon 只是把"往链上加东西"变成了可复用单元。

`useAsyncData` 那一节给的例子是另一种常见需求：`pollEvery` 选项（`UseAsyncDataAddonOptions<{ pollEvery?: number }>`）按间隔刷新。

## 扩展返回值

规则一句：**"If `setup` returns a function, that function is called with the async data instance. Any object it returns is merged into the composable's return value."**

官方例子中，这个 addon 用 `watch(asyncData.status, ...)` 记下"最近一次成功的时间"，返回时用了 `readonly`：

```ts
const timestamps = defineUseFetchAddon({
  setup: () => {
    const refreshedAt = ref<Date>()

    return (asyncData) => {
      watch(asyncData.status, (status) => {
        if (status === 'success') { refreshedAt.value = new Date() }
      }, { immediate: true })

      return { refreshedAt: readonly(refreshedAt) }
    }
  },
})
```

调用侧 `const { data, refreshedAt } = await useCustomFetch('/modules')`，注释是 "`refreshedAt` is typed"。

## 最容易踩的一处：自定义选项不进 key

这节标题就说明了问题——"Contributing to the Auto-Generated Key"：

> Custom options are **not** part of the auto-generated key by default. If a custom option affects the response, provide a `key` resolver so calls with different values do not share a cached entry. Return a serializable value, or `undefined` to contribute nothing:

```ts
const scoped = defineUseFetchAddon({
  setup: (options: UseFetchAddonOptions<{ scope?: string }>) => {
    options.scope ??= 'default'
    // ...
  },
  key: options => toValue(options.scope),
})
```

**症状可以预测**：`scope` 不同、URL 相同的两次调用会共用同一格缓存（215 讲的正是 key 决定归属）。官方给的解法就是那个 `key` 解析函数。

紧接着有一条我觉得排第二容易踩的 TypeScript 顺序警告：

> Declare `setup` before `key`. TypeScript infers the custom options from the `setup` parameter annotation, and a `key` resolver written before a `setup` that returns an extension function is checked first and typed with empty options.

意思是：**对象字面量里 `key` 写在 `setup` 前面，类型推断会先看到 `key`，于是自定义选项被推成空**。我把这条记成写法纪律：addon 对象里 `setup` 永远放第一个键。

## 高级出口：包装 then、catch、finally

官方把这称为 escape hatch，规则是"不合并、而是包一层"：

> As an advanced escape hatch, the extension object may include `then`, `catch` or `finally` functions. These are not merged into the instance. Instead they wrap the corresponding method of the awaitable promise, receiving the original method as their first argument.

多 addon 包同一个方法时，顺序与中间件一致：**"the first addon's wrapper wraps all the following ones, just as its middleware would."**

这条我只当"知道有这回事"用——它对应的是 214 那个 `AsyncData & Promise` 交叉类型（`then`/`catch`/`finally` 本来就来自 Promise 那一半）。

## 两节签名对照

```ts
function defineUseFetchAddon<Opts extends Record<string, any> = {}, Ext = {}> (addon: {
  setup: (options: UseFetchAddonOptions<Opts>) => ((asyncData: AsyncDataAddonInstance) => Ext | void) | void
  key?: (options: UseFetchAddonOptions<Opts>) => SerializableValue
}): UseFetchAddon<Opts, Ext>
```

`useAsyncData` 那侧的对应签名是：`setup: (options: UseAsyncDataAddonOptions<Opts>) => ((asyncData: AsyncDataAddonInstance) => Ext | void) | void`——**除了选项类型名，结构完全一样**，返回实例类型都用 `AsyncDataAddonInstance`。

## 自查

1. addon 的 `setup` 什么时候跑、参数是什么？（答：每次调用组合式都跑，参数是工厂默认值与调用方选项合并后的对象。）
2. 自定义选项会不会进自动生成的 key？（答：默认不会；选项影响响应时要用 `key` 解析函数返回可序列化值。）
3. 官方对象字面量里为什么 `setup` 必须写在 `key` 之前？（答：TS 从 `setup` 的参数标注推断自定义选项，先写 `key` 会被推断成空选项。）

## 下一篇

228 - useNuxtData 实战：乐观更新与失败回滚的官方形状
