---
url: /posts/1befab184398b8f0c78369847014956c/
title: 222 - 服务端那次请求要不要带头：useRequestFetch 与 baseURL 选项
date: 2026-12-25T09:00:00+08:00
updated: 2026-12-25T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：211 留了一个尾巴（服务端那一趟不带 cookie），这一篇给出官方指定的解法：`useRequestFetch`。它的定位句是 "You can use `useRequestFetch` to forward the request context and headers when making server-side fetch requests."，前半段解释了为什么浏览器侧不用管："When making a client-side fetch request, the browser automatically sends the necessary headers. However, when making a request during server-side rendering, **due to security considerations, we need to forward the headers manually.**"。有两点必须连在一起读：一是这条链默认是 `useFetch` 在做——"The `useFetch` composable uses `useRequestFetch` under the hood to automatically forward the request context and headers."；二是不是所有头都转，官方列了七项"**not meant to be forwarded**"的：`transfer-encoding`、`connection`、`keep-alive`、`upgrade`、`expect`、`host`、`accept`。官方给的正反例连返回值都写出来了：`useRequestFetch()` 那次得到 `{ cookies: { foo: 'bar' } }`，包着裸 `$fetch` 的那次得到 `{ cookies: {} }`；另一条 tip 说明**客户端导航时它就等于普通 `$fetch`**。至于 `baseURL`，我要提醒它有两个不同层面的东西撞名：`useFetch` 选项表里那行是"Base URL for the request."（单次请求用），而配置里的 `app.baseURL` 是 "The base path of your Nuxt application."（默认 `"/"`，还能用 `NUXT_APP_BASE_URL` 在运行时设）——**两者关系文档没有写**。

categories:
  - 前端开发

tags:
  - nuxt4
  - useRequestFetch
  - baseURL
  - 请求头
  - SSR

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先解释这一篇里两个词，因为它们在讲同一件事的两端：

- **请求头（headers）**：浏览器发请求时附带的那些键值，`cookie` 是其中一个。
- **转发（forward）**：服务端替你去打自己的接口时，把这些头一起带过去。

问题就出在"服务端替你打"这一步：那次请求的发起者是服务器，不是你浏览器。

## 为什么浏览器侧不用管、服务端侧要管

原文两句把差异讲完了：

> When making a client-side fetch request, the browser automatically sends the necessary headers. However, when making a request during server-side rendering, due to security considerations, we need to forward the headers manually.

"security considerations" 是官方的说法，`$fetch` 那一节给了更具体的两个名词：**Server-Side Request Forgery (SSRF)** 与 **Authentication Misuse**（211 抄过）。所以默认不转是刻意的安全选择，不是遗漏。

`useRequestFetch` 就是那个"手转"的官方工具，它返回一个可用的取数函数：

```vue [app/pages/index.vue]
<script setup lang="ts">
// This will forward the user's headers to the `/api/cookies` event handler
// Result: { cookies: { foo: 'bar' } }
const requestFetch = useRequestFetch()
const { data: forwarded } = await useAsyncData(() => requestFetch('/api/cookies'))

// This will NOT forward anything
// Result: { cookies: {} }
const { data: notForwarded } = await useAsyncData((_nuxtApp, { signal }) => $fetch('/api/cookies', { signal }))
</script>
```

配套的服务端接口只有三行：

```ts [server/api/cookies.ts]
export default defineEventHandler((event) => {
  const cookies = parseCookies(event)

  return { cookies }
})
```

**注意官方把结果值都写出来了**：转发的那次拿到 `{ cookies: { foo: 'bar' } }`，没转发的拿到 `{ cookies: {} }`。这个对照是最省事的判据——照着搭一个 `/api/cookies`，两端读数立刻能确认你的调用走的是哪条路。

## 两条 tip：谁默认在用，以及浏览器侧它是谁

第一条：

> The `useFetch` composable uses `useRequestFetch` under the hood to automatically forward the request context and headers.

这解释了为什么很多人从 `useFetch` 改成"包一层的 `useAsyncData`"之后，登录态就丢了——**丢的是这个自动代理**。211 里 `useFetch` 那段相对路径的说明（"Nuxt will use `useRequestFetch` to proxy client headers and cookies"）是同一条事实的另一处写法。

第二条：

> In the browser during client-side navigation, `useRequestFetch` will behave just like regular `$fetch`.

所以我不用担心"用了它就会在客户端多做什么"——**浏览器侧它和普通 `$fetch` 一样**，差别只在服务端那一趟。

## 那七个头不转

note 原文与清单：

> Headers that are **not meant to be forwarded** will **not be included** in the request. These headers include, for example: `transfer-encoding`, `connection`, `keep-alive`, `upgrade`, `expect`, `host`, `accept`

我把这七项分成两类看：**连接类**（`transfer-encoding`、`connection`、`keep-alive`、`upgrade`、`expect`）描述的是"这一次浏览器到服务器之间的物理会话"，转发给内部接口没有意义甚至有害；**内容协商类**（`host`、`accept`）则关系到"你是哪个域名的哪个客户端"，官方的措辞是 "for example"，也就是**这份清单不是封闭列表**。

`host` 这一项特别值得注意：它和 204 那个坑是同一族（缓存命中时 host 读成 `localhost`）。**如果你依赖请求里的 `host` 拼绝对 URL，请回去看 `cache.varies` 那条。**

另外 `api/kit` 那类"不转发某些头"的说法也在 211 的 caution 清单里出现过（`x-forwarded-*`、`cf-*` 等），两处口径一致。

## baseURL：两个层面撞了同一个名字

| 出现处 | 官方原话 | 类型与默认 |
| --- | --- | --- |
| `useFetch` 选项表 | "Base URL for the request." | `MaybeRefOrGetter<string>`，默认 `-` |
| `app.baseURL`（nuxt.config） | "The base path of your Nuxt application." | `string`，默认 `"/"` |

配置层那条还给了运行时覆盖方式：

> This can also be set at runtime by setting the NUXT_APP_BASE_URL environment variable.

```bash
NUXT_APP_BASE_URL=/prefix/ node .output/server/index.mjs
```

**两者的关系文档没有写**——也就是说，从这两节也推不出"设了 `app.baseURL` 之后，`useFetch` 里相对路径会自动带上这个前缀"。这个空白很实际：把应用挂在子路径下（示例用的 `/prefix/`）时，内部接口到底拼成 `/prefix/api/...` 还是 `/api/...`，必须实测。我的测法：设上 `NUXT_APP_BASE_URL=/prefix/`，服务端渲染时打印 `useRequestURL().pathname` 与实际请求出去的 URL，两边对一次。

要固定给某类请求挂 baseURL 或鉴权头，官方给的出口不是每次传选项，而是做自己的变体：

> Need a custom `useFetch` with pre-defined defaults (like `baseURL` or auth headers)? Use `createUseFetch` to create a fully typed custom composable.

223 会接着讲手动转头的另一种粒度（`useRequestHeaders`），以及它和这里"整体代理"的取舍。

## 自查

1. 为什么服务端那一趟默认不带 cookie？（答：官方说是安全考量，`$fetch` 那一节点名 SSRF 与 Authentication Misuse。）
2. `useFetch` 与"包 `useAsyncData` 的裸 `$fetch`"在请求头上有什么不同？（答：前者内部用 `useRequestFetch` 自动转发上下文与头，后者不转——官方示例的读数分别是 `{ cookies: { foo: 'bar' } }` 与 `{ cookies: {} }`。）
3. `useFetch` 的 `baseURL` 和 `app.baseURL` 是什么关系？（答：文档没写；一个是单次请求的 base，一个是应用的基础路径，需实测。）

## 下一篇

223 - 请求头转发：只挑需要的用 useRequestHeaders
