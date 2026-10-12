---
url: /posts/54c2ce7ae21b984ff1ed8a93f806c7ef/
title: 取数失败与全局错误：error ref 和 useError 是两个地方
date: 2026-12-29T09:00:00+08:00
updated: 2026-12-29T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇讲一件很容易做错的事：**取数失败默认不会跳到错误页**。`useFetch`/`useAsyncData` 返回的那个 `error` 是**局部的 ref**——官方说明是 "Error object if the asynchronous function threw an error."，你不读它，它就静静躺在那儿；而全站错误页要的是**抛出去**的错误，官方给的两个入口各有说法：`throw createError()`（"If you throw an error created with `createError`: on server-side, it will trigger a full-screen error page which you can clear with `clearError`. on client-side, it will throw a non-fatal error for you to handle. **If you need to trigger a full-screen error page, then you can do this by setting `fatal: true`.**"），以及 `showError`——它确实能触发全屏错误页（"You can call this function at any point on client-side, or (on server side) directly within middleware, plugins or `setup()` functions."），**但官方紧接着的建议是 "It is recommended instead to use `throw createError()`."**。读全局错误用 `useError()`，它返回 `Ref<NuxtError | undefined>`、"available on both client and server"、"reactive and will update automatically when the error state changes"。官方在文档里给的取数→404 组合就是一段：拿到 `data` 后判空再 `throw createError({ status: 404, statusText: 'Page Not Found' })`。最后要提醒一处文档内部不一致：**`useError` 的返回类型在两页写法不同**，`api/composables/use-error` 给的是 `{status, statusText?, message, data?, cause?, fatal}`，而错误处理页的签名是 `Ref<Error | { url, status, statusText, message, description, data }>`（多了 `url`/`description`、少了 `fatal`/`cause`）。

categories:
  - 前端开发

tags:
  - nuxt4
  - 错误处理
  - useError
  - createError
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先给一个具体症状：接口 500 了，页面既不报错也不显示错误页，只是那块内容空白。**这不是 bug，是默认行为**——取数的错误和全站错误页是两套通道。

## 两个通道分别是什么

| 通道 | 是什么 | 谁会看到 |
| --- | --- | --- |
| 返回值里的 `error` | 每次取数自带的一个 ref："Error object if the asynchronous function threw an error." | 只有你读了它的组件 |
| 全局错误 | 用 `throw createError()` 或 `showError()` 触发，`useError()` 能读到 | 整站，通常是 `error.vue` 那一屏 |

**关键点：接口失败不会自动升级成全局错误。** 214 那张表里 `error` 只是一个普通返回值，`status` 会变成 `'error'`（214 的四值），仅此而已。

## 官方文档里给的组合写法

错误处理页有一段完整例子，正好是"取数 → 判空 → 交给全局错误页"：

```vue twoslash [pages/movies/[slug].vue]
<script setup lang="ts">
const route = useRoute()
const { data } = await useFetch(`/api/movies/${route.params.slug}`)

if (!data.value) {
  throw createError({
    status: 404,
    statusText: 'Page Not Found',
  })
}
</script>
```

它判的是 `!data.value` 而不是 `error.value`——**这一处我觉得值得琢磨**：接口返回空值时走这个分支，但接口 500 时 `data` 也是空，于是错误会被报成 404 而不是 500。**这是官方示例的取舍，不是唯一正确写法。**我自己的做法是两支都判：先看 `error.value` 有没有，再判 `data.value` 空不空。

配套那条 tip 也要记：`statusText` 只允许水平制表符、空格和可见 ASCII（原话 "It should only contain horizontal tabs, spaces, and visible ASCII characters (`[\t\u0020-\u007E]`)."），**中文说明要放 `message`**（111 也记过这条）。

## 抛错之后会发生什么，两侧不一样

`createError` 那段原文必须整段读：

> If you throw an error created with `createError`:
> - on server-side, it will trigger a full-screen error page which you can clear with `clearError`.
> - on client-side, it will throw a non-fatal error for you to handle. If you need to trigger a full-screen error page, then you can do this by setting `fatal: true`.

也就是说**同一个 `throw`，在客户端默认是"非致命"**——不弹全屏错误页。要弹就显式 `fatal: true`。

还有一条对排错影响很大：

> In development, an error `cause` is preserved and exposed to your error page so you can trace the original error; in production, causes are never included in error responses or the error page payload.

**开发时能看到 `cause`、上线就没了**——所以别把"看 cause 定位"当成线上手段，线上要留信息得写进 `data`（056 记过"自定义字段必须放 `data`"）。

## showError：能用，但官方建议你别用

原文两句放在一起看才有意思：

> You can call this function at any point on client-side, or (on server side) directly within middleware, plugins or `setup()` functions. It will trigger a full-screen error page which you can clear with `clearError`.
>
> It is recommended instead to use `throw createError()`.

签名是 `function showError (err: string | Error | { status, statusText }): Error`——**注意它的参数形状里没有 `message`/`data`/`fatal`**，而 `createError` 收的是 `{ cause, data, message, name, stack, status, statusText, fatal }`。这也是那句"建议改用 `throw createError()`"的合理来源：信息量能带得多得多。

`clearError` 则给错误页一个出口：`function clearError (options?: { redirect?: string }): Promise<void>`，"It also takes an optional path to redirect to (for example, if you want to navigate to a 'safe' page)."

## 读全局错误：useError

`api/composables/use-error` 那页给的是：

```ts
export const useError: () => Ref<NuxtError | undefined>

interface NuxtError<DataT = unknown> {
  status: number
  statusText?: string
  message: string
  data?: DataT
  cause?: unknown
  fatal: boolean
}
```

配套三句我关心的是最后那句反应式行为：

> The `useError` composable returns the global Nuxt error that is being handled and is available on both client and server. It provides a reactive, SSR-friendly error state across your app.

> Returns a `Ref` containing the current Nuxt error (or `undefined` if there is no error). **The error object is reactive and will update automatically when the error state changes.**

官方示例只有一小块（读一次打日志）：

```vue
<script setup lang="ts">
const error = useError()

if (error.value) {
  console.error('Nuxt error:', error.value)
}
</script>
```

**这一处示例有个我该提醒你的性质**：它在 setup 里读一次 `error.value`。因为返回值是 ref，如果错误是在这之后才出现的，那次 `if` 判断不会重跑——要响应变化得放进模板或 `watch`。官方那句 "will update automatically" 说的是 ref 本身会更新，不是"这段代码会重跑"。

## 两处文档不一致，我不替你选

| 位置 | `useError` 的类型写法 |
| --- | --- |
| `api/composables/use-error` | `() => Ref<NuxtError \| undefined>`，字段 `status`/`statusText?`/`message`/`data?`/`cause?`/`fatal` |
| `getting-started/error-handling` | `function useError (): Ref<Error \| { url, status, statusText, message, description, data }>` |

差异是实打实的：一边有 `url` 和 `description`，另一边有 `fatal` 与 `cause`。第三处（`app/error` 那页的 `NuxtError`，056 抄过）又是 8 个字段、带 `unhandled` 和两个 legacy 等价键、**没有 `message`**。

**所以"错误对象上到底有什么"只能实测**：把 `useError().value` 打到控制台、以及从 `error.vue` 的 prop 里打一次，比较两边字段。别按任何一页的签名写死代码。

## 不想整站跳转：局部兜住

如果你要的是"这块内容坏了，别把整站换掉"，官方给的组件是 `<NuxtErrorBoundary>`："This component is responsible for handling errors that occur within its default slot. On client-side, it will prevent the error from bubbling up to the top level, and will render the `#error` slot instead."，槽里能拿到 `error` 与 `clearError`，还有一句换路由自动清除的 tip（095 已细讲）。

**但它是 Vue 层面的错误边界，处理的是渲染/生命周期里抛出的错误**——取数失败默认并不抛（它在 `error` ref 里），所以要在边界内触发它，得在取数之后主动 `throw`（官方那段 404 例子正是这个形状）。

## 自查

1. 接口 500 之后默认会看到什么？（答：页面不跳错误页；只有那次取数返回的 `error` ref 有值、`status` 变成 `'error'`。）
2. 客户端 `throw createError()` 会全屏吗？（答：默认 non-fatal；要全屏得设 `fatal: true`。）
3. 三页给的错误对象字段一样吗？（答：不一样，`use-error` 页有 `message`/`fatal`/`cause`，错误处理页有 `url`/`description`，`app/error` 页有 `unhandled` 而无 `message`，必须实测。）

## 下一篇

235 - payload 有多重：载荷提取、pick 与序列化两道关

## 本文依据

- `api/composables/use-error`：定位句与 "available on both client and server"、"reactive, SSR-friendly"；`useError: () => Ref<NuxtError | undefined>` 与 `NuxtError` 六字段；Return Values 段（含 "will update automatically"）；示例小块。
- `getting-started/error-handling`：`useError` 签名 `Ref<Error | { url, status, statusText, message, description, data }>`；`createError` 签名与三句（服务端全屏、客户端非致命、`fatal: true`）；`cause` 开发/生产那段；`movies/[slug]` 的 404 示例；`statusText` 字符集 tip；`showError` 两句与签名；`clearError` 签名与可选 redirect；`<NuxtErrorBoundary>` 三句与 tip；Node 16 的 cookie note。
- `directory-structure/app/error`：`NuxtError` 八字段清单与"自定义字段放 `data`"（056/207 已逐句引）。
- `api/composables/use-async-data`、`getting-started/data-fetching`：`error` 与 `status` 四值的说明（214 已逐句引）。
- `api/components/nuxt-error-boundary`：错误边界与 `#error` 槽（095 已逐句引）。
