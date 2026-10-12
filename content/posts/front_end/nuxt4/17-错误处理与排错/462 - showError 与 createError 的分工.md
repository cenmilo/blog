---
url: /posts/c3c68ac41c7978d3a3d115410d7d1ef0/
title: showError 与 createError 的分工
date: 2027-03-15T09:00:00+08:00
updated: 2027-03-15T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：这两个函数常被当成一回事，官方给的定义把它们分得很开：`createError()` 造一个带元数据的错误对象，"**is meant to be thrown**"，在 Vue 与 Nitro 两侧都能用；`showError()` 是在 Nuxt context 内**直接把错误显示成全屏错误页**，它把错误写进 `useError()` 那份响应式共享状态，并且 `::tip` 明写它会调用 `app:error` 钩子。更关键的取向句在错误处理页：对 `showError`，官方的建议是"**其实更推荐用 `throw createError()`**"。另外还有两条最容易踩的规则：API 路由里 `message` 不会传到客户端、能传的是短 `statusText` 或 `data`；客户端抛出的 `createError` 默认是**非致命**的，要全屏错误页必须写 `fatal: true`。

categories:
  - 前端开发

tags:
  - nuxt4
  - createError
  - showError
  - 错误处理
  - API
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

服务端路由里写 `throw createError({ status: 404 })`，客户端点按钮时写 `showError({ status: 404 })`——看起来是两种写法干同一件事。

读完两个参考页会发现不是：**一个负责造错误对象给你抛，一个负责直接把它显示出来**，而且官方明确表了态。

## createError：造出来是为了抛

参考页的定义句最后五个词是重点：

> You can use this function to create an error object with additional metadata. It is usable in both the Vue and Nitro portions of your app, and **is meant to be thrown**.

三个信息：**带元数据的错误对象**、**Vue 侧和 Nitro 侧都能用**、**用途是被 throw**。

参数只有一个：

- `err`: `string | { cause, data, message, name, stack, status, statusText, fatal }`

两种形态的差别官方写清了：传字符串时它成为错误的 `message`，**`status` 默认 `500`**；传对象时可以设 `status`、`message` 和其他错误属性。

### 抛出之后在哪一侧会怎样

这一组不对称是最容易判断错的：

> - on server-side, it will trigger a full-screen error page which you can clear with `clearError`.
> - on client-side, it will throw a non-fatal error for you to handle. If you need to trigger a full-screen error page, then you can do this by setting `fatal: true`.

**服务端抛出＝直接全屏错误页；客户端抛出＝非致命错误，等你自己处理**。客户端想要全屏页，必须显式 `fatal: true`。这也和 460 那条 fatal 定义对得上（fatal＝服务端任何未处理错误，或客户端 `fatal: true` 创建的错误）。

官方给的页面内示例就是这条路径：

```vue [app/pages/movies/[slug\\].vue]
<script setup lang="ts">
const route = useRoute()
const { data } = await useFetch(`/api/movies/${route.params.slug}`)
if (!data.value) {
  throw createError({ status: 404, statusText: 'Page Not Found' })
}
</script>
```

### cause：只在开发模式可见

想保留原始错误，官方给的是 `cause`：

```ts
try {
  await fetchMovie(route.params.slug)
} catch (cause) {
  throw createError({
    status: 500,
    message: 'Could not load movie',
    cause,
  })
}
```

规则段说得很细：

> In development, the cause chain is exposed to your error page via the `cause` property of the error, serialized as `{ name, message, stack, cause }` (primitive causes are passed through as-is; other values are omitted). In production, causes are never included in error responses or in the error page payload.

三点：**开发模式下**因果链会经 `cause` 暴露给错误页、序列化形状是 `{ name, message, stack, cause }`、**原始值的 cause 原样传、其他值被省略**；**生产环境里 cause 永远不进错误响应，也不进错误页 payload**。

## showError：调用它、它去显示

参考页的定性是"quick and simple way to show a full screen error page if needed"，用法条件是这一句：

> Within the [Nuxt context](https://nuxt.com/docs/4.x/guide/going-further/nuxt-app#the-nuxt-context) you can use `showError` to show an error.

**它要求在 Nuxt context 内**（也就是运行时上下文里；脱离上下文拿不到 `useError` 那份状态）。

参数比 createError 少一项：

- `error`: `string | Error | Partial<{ cause, data, message, name, stack, status, statusText }>`

注意 `Partial`——**这份清单里没有 `fatal`**，因为它是"展示"用的，不决定致命性；`createError` 的参数里有 `fatal`。

两个动作官方各给一句：

> The error is set in the state using `useError()` to create a reactive and SSR-friendly shared error state across components.

**它做的事就是把错误写进 `useError()` 那份响应式、SSR 友好的跨组件状态**。

> `showError` calls the `app:error` hook.

**它会触发 `app:error`**——接回 460 那张表：启动错误那一类的钩子，会被 `showError` 借用。

而错误处理页给的签名更窄（`function showError (err: string | Error | { status, statusText }): Error`），并且带一句取向明确的建议：

> You can call this function at any point on client-side, or (on server side) directly within middleware, plugins or `setup()` functions. It will trigger a full-screen error page which you can clear with `clearError`.
>
> It is recommended instead to use `throw createError()`.

两句话合起来读：`showError` 能调的时机是**客户端任意时刻**、服务端则是**中间件、插件或 `setup()` 里直接调**；官方仍然建议改用 `throw createError()`。

## 三条对照

| | `createError()` | `showError()` |
| --- | --- | --- |
| 用途 | 造对象，**要被 throw** | 直接把错误显示成全屏错误页 |
| 参数里的 `fatal` | **有** | 没有（`Partial` 清单里不含） |
| 与状态的关系 | 抛出后由框架处理 | 显式写进 `useError()` 的响应式状态 |
| 与钩子 | 未写 | `::tip` 明写调用 `app:error` |
| 可用范围 | Vue 侧与 Nitro 侧 | 需在 **Nuxt context** 内；服务端限中间件/插件/`setup()` |
| 官方建议 | — | "It is recommended instead to use `throw createError()`" |

## API 路由里那两条硬规则

这一段是实际写接口时最容易踩空的，官方写得直白：

> In API routes, using `createError` by passing an object with a short `statusText` is recommended because it can be accessed on the client side. Otherwise, a `message` passed to `createError` on an API route will not propagate to the client. Alternatively, you can use the `data` property to pass data back to the client. When handling the error with `useFetch`, the custom data is available at `error.value.data.data`. In any case, always consider avoiding to put dynamic user input to the message to avoid potential security issues.

拆成四条：

1. 接口里要用 `createError` 传一个**短 `statusText`** 的对象，因为它能到客户端。
2. **接口里的 `message` 不会传到客户端**——这条经常被当成 bug，其实是规则。
3. 要带结构化的东西回来，用 `data`；用 `useFetch` 处理错误时，自定义数据在 **`error.value.data.data`** 这个位置。
4. 无论如何**别把动态的用户输入塞进 message**，官方直接说是为了避免安全问题。

第 3 条那个双 `.data` 很反直觉（外层是 h3 错误的 `data` 包装，内层才是你给的对象），写测试时容易漏一层。

官方的接口写法示例：

```ts [server/api/error.ts]
export default eventHandler(() => {
  throw createError({
    status: 404,
    statusText: 'Page Not Found',
  })
})
```

## 自查

- 造错误用 `createError` 并 throw；只是要"显示"才用 `showError`，且需在 Nuxt context 内，它会写进 `useError()` 并触发 `app:error`。
- 官方对 `showError` 的建议是让位给 `throw createError()`。
- 服务端 throw＝全屏错误页；客户端 throw＝非致命，要全屏得 `fatal: true`。
- `cause` 只在开发模式进错误页，生产不进响应也不进 payload。
- 接口里 `message` 不到客户端；短 `statusText` 或 `data` 才行，自定义数据在 `error.value.data.data`；用户输入别进 message。

## 下一篇

下一篇 463 讲怎么收掉它：`clearError(options?: { redirect?: string })` 返回 Promise，用来清掉当前正在处理的错误，并可选带一个要跳转的路径——官方举的例子是"导航到一个安全的页面"。

## 本文依据

- `createError` 参考页：定义句（含 "is meant to be thrown"、Vue 与 Nitro 两侧可用）、参数 `string | { cause, data, message, name, stack, status, statusText, fatal }`、字符串形态与 `status` 默认 `500`、In Vue App 两条（服务端全屏页可用 `clearError` 清、客户端非致命需 `fatal: true`）、页面示例、Error Causes 段与 `cause` 示例及 `{ name, message, stack, cause }` 序列化/原始值透传/生产不含的三条、In API Routes 示例与那段四规则（短 `statusText` 可达客户端、`message` 不传播、`data`、`error.value.data.data`、勿把动态用户输入放进 message）（`api/utils/create-error`）
- `showError` 参考页：定位句（quick and simple way to show a full screen error page）、"Within the Nuxt context you can use `showError`"、参数 `string | Error | Partial<{...}>`、对象形态示例、"The error is set in the state using `useError()`…"、`::tip` "`showError` calls the `app:error` hook."（`api/utils/show-error`）
- 错误处理页的 Error Utils：`createError` 签名与两条抛出后果、`cause` 的开发/生产差异句、`statusText` 只允许 HTTP 合规字符的 `::tip`（`[\t\u0020-\u007E]`）与详细说明要用 `message`；`showError` 签名、可调用时机句与"It is recommended instead to use `throw createError()`"（`getting-started/error-handling`）
