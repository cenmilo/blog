---
url: /posts/16348b4943365cf8c7931baab1872bc3/
title: 308 - 服务端错误 createError：status、statusText、data 与 fatal
date: 2027-01-22T11:00:00+08:00
updated: 2027-01-22T11:00:00+08:00
author: cmdragon

summary:
  摘要：接口要回一个带状态码的错误，官方工具是 createError 抛出去。这一篇把它的参数摊开：既能传字符串（当 message、status 默认 500），也能传对象（status/statusText/message/data/cause/fatal 等）；在 Vue 侧抛服务端触发全屏错误页、客户端默认是 non-fatal 要 fatal:true 才全屏；在 API 路由里抛则变成对应状态的 JSON 错误。还讲 cause（开发环境暴露、生产不进响应）和一条官方安全提醒——别把动态用户输入放进 message。

categories:
  - 前端开发

tags:
  - nuxt4
  - createError
  - 服务端错误
  - status
  - fatal
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上一篇（307）是"正常内容 + 标个状态码"。这一篇是它的对面：**出错时怎么抛一个带状态码的错误**。工具是 `createError`，官方定位：

> Create an error object with additional metadata. ... It is usable in both the Vue and Nitro portions of your app, and is meant to be thrown.

关键词 **meant to be thrown**——它是设计来 `throw` 的，不是 `return` 的。

## 参数：字符串或对象

官方给的参数形状：

> `err`: `string | { cause, data, message, name, stack, status, statusText, fatal }`

两种传法：

> You can pass either a string or an object to the `createError` function. If you pass a string, it will be used as the error `message`, and the `status` will default to `500`. If you pass an object, you can set multiple properties of the error, such as `status`, `message`, and other error properties.

- 传**字符串** → 它当 `message`，`status` 默认 **500**。
- 传**对象** → 你能设 `status`、`statusText`、`message`、`data`、`cause`、`fatal` 等。

## 在 Vue 应用里抛：服务端全屏、客户端默认非致命

官方 "In Vue App" 三条，讲"throw createError 之后会怎样"，两端不同：

> - on server-side, it will trigger a full-screen error page which you can clear with `clearError`.
> - on client-side, it will throw a non-fatal error for you to handle. If you need to trigger a full-screen error page, then you can do this by setting `fatal: true`.

这条两端差别要记牢（234 也讲过）：

- **服务端** `throw createError(...)` → 直接触发**全屏错误页**。
- **客户端** `throw createError(...)` → 是一个 **non-fatal** 错误，交给你自己处理；要它也弹全屏错误页，得显式 `fatal: true`。

官方示例（页面里取不到数据就抛 404）：

```vue [app/pages/movies/[slug].vue]
<script setup lang="ts">
const route = useRoute()
const { data } = await useFetch(`/api/movies/${route.params.slug}`)
if (!data.value) {
  throw createError({ status: 404, statusText: 'Page Not Found' })
}
</script>
```

（这里判的是 `!data.value`，会把任何取不到都报成 404——是示例的简化取舍，真实项目里应区分 404 和 500。）

## 在 API 路由里抛

官方 "In API Routes" 给的就是接口里出错的写法：

```ts [server/api/error.ts]
export default eventHandler(() => {
  throw createError({
    status: 404,
    statusText: 'Page Not Found',
  })
})
```

在 API 路由里 throw 一个 createError，响应就带上这个 `status`（和 `statusText`）以 JSON 错误形式返回。这也是 292 里服务端鉴权 `requireUserSession` 抛 401 的底层机制。

## cause：包一层保留原始错误，但生产不外泄

想保留你包住的那个原始错误，用 `cause`：

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

官方对它的暴露范围有明确边界：

> In development, the cause chain is exposed to your error page via the `cause` property of the error, serialized as `{ name, message, stack, cause }` (primitive causes are passed through as-is; other values are omitted). In production, causes are never included in error responses or in the error page payload.

两条：

- **开发环境**：错误链通过 `cause`（序列化成 `{ name, message, stack, cause }`）显示到错误页，方便你排查。
- **生产环境**：cause **永远不进错误响应、也不进错误页 payload**——不会把内部堆栈泄露给前端。这跟 234 那句"`cause` 只在开发环境暴露"一致。

## 一条安全提醒：别把用户输入放进 message

官方在 API Routes 那段末尾给了安全口径（211、259 一路都在强调别把不该暴露的东西下发）：

> always consider avoiding to put dynamic user input to the message to avoid potential security issues.

也就是 `message`（和会下发给客户端的 `statusText`/`data`）里，别直接拼用户的动态输入——可能引入安全问题。要给前端带结构化数据，用 `data`（下一篇 309 讲 `message` 不下发、`statusText`/`data` 才下发的区别，以及 `error.value.data.data` 的取用位置）。

## 自查

- 出错要 `throw createError(...)`，不是 return；传字符串=只给 message、status 落 500，要精确状态用对象。
- 客户端默认 non-fatal，要全屏错误页补 `fatal: true`；服务端直接全屏。
- API 路由里 throw → 带对应 status 的 JSON 错误；要传结构化回前端数据用 `data`（309）。
- `cause` 只在开发暴露、生产不进响应；别把用户动态输入塞进 message。

## 下一篇

下一篇 309 专门讲那条"下发规则"：为什么 API 路由里 createError 的 `message` 不会传到客户端，要前端拿到信息该用 `statusText` 或 `data`，以及 `error.value.data.data` 的位置。

## 本文依据

- createError（`api/utils/create-error`）：定位句 "usable in both the Vue and Nitro portions of your app, and is meant to be thrown."；参数 `string | { cause, data, message, name, stack, status, statusText, fatal }` 与字符串→message+status 500、对象→多属性；In Vue App 两端差别（服务端全屏可 clearError、客户端 non-fatal 需 fatal:true）与 `movies/[slug].vue` 示例；In API Routes（`server/api/error.ts` throw 404/`statusText`）；Error Causes（`cause` 示例、开发暴露 `{ name, message, stack, cause }`、生产永不进响应/payload）；安全提醒 "always consider avoiding to put dynamic user input to the message to avoid potential security issues."
