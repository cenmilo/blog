---
url: /posts/406c3b9694823dbb1e9aa145e8468152/
title: 318 - useRequestFetch：服务端取数时把请求头带上
date: 2027-01-26T09:00:00+08:00
updated: 2027-01-26T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：浏览器发请求会自动带 cookie、authorization 这些头；但 SSR 阶段在服务端用 $fetch 打自家接口时，出于安全考虑不会自动转发这些头——于是内部接口读不到登录态。useRequestFetch 返回一个"会把这次请求的上下文和头一起转发"的 fetch 实例。这一篇讲它的定位、那个 forwarded/notForwarded 的对照示例（{ cookies: { foo: 'bar' } } vs { cookies: {} }）、哪些头不转发、以及 useFetch 底层用的就是它这条 tip。

categories:
  - 前端开发

tags:
  - nuxt4
  - useRequestFetch
  - 转发头
  - SSR
  - useFetch
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


315 里我们手动用 `useRequestHeaders(['authorization'])` 把某个头塞进取数请求。这一篇的 `useRequestFetch` 是它的"整体版"——一次性把这次请求的上下文和头都转发过去。

## 要解决的问题

官方把两端的差别讲得很清楚：

> When making a client-side fetch request, the browser automatically sends the necessary headers. However, when making a request during server-side rendering, due to security considerations, we need to forward the headers manually.

也就是说：

- **浏览器里发请求**：cookie、authorization 这些头浏览器自动带上。
- **SSR 阶段服务端发请求**：出于安全考虑（"due to security considerations"），**不会自动转发**这些头，得手动转。

这就是 210-249 那几篇（211）那条"SSR 里 `$fetch('/api/...')` 不带 cookie"的根因。`useRequestFetch` 就是把"手动转发"这件事包成一个现成的 fetch 实例。

## 它是什么

官方定位：

> Forward the request context and headers for server-side fetch requests with the useRequestFetch composable.

调它拿到一个 `requestFetch`，用它打内部请求就会自动转发上下文和头。

## 转发 vs 不转发：官方那个对照示例

官方把两条路并排放在同一个页面里，最能说明差别：

```vue [app/pages/index.vue]
<script setup lang="ts">
// 这行会把用户请求头转发给 /api/cookies
// 结果：{ cookies: { foo: 'bar' } }
const requestFetch = useRequestFetch()
const { data: forwarded } = await useAsyncData(() => requestFetch('/api/cookies'))

// 这行什么都不转发
// 结果：{ cookies: {} }
const { data: notForwarded } = await useAsyncData((_nuxtApp, { signal }) => $fetch('/api/cookies', { signal }))
</script>
```

对应那个被访问的服务端接口：

```ts [server/api/cookies.ts]
export default defineEventHandler((event) => {
  const cookies = parseCookies(event)
  return { cookies }
})
```

读数很直白：用 `useRequestFetch()` 的结果是 `{ cookies: { foo: 'bar' } }`（cookie 转发到了），用裸 `$fetch` 的结果是 `{ cookies: {} }`（什么都没转过去）。同一个内部接口，靠的是"这次请求的头有没有带过来"。

## 哪些头不转发

不是所有头都照转。官方 note 列了不转发的那批（举例性质）：

> Headers that are not meant to be forwarded will not be included in the request. These headers include, for example: `transfer-encoding`, `connection`, `keep-alive`, `upgrade`, `expect`, `host`, `accept`

这些是 hop-by-hop 或本就不该原样转发的头，`useRequestFetch` 会把它们剔掉。

## 一条 tip：useFetch 底层就是它

官方 tip 点破了 210-249 那条"为什么 useFetch 能自动带上下文"：

> The `useFetch` composable uses `useRequestFetch` under the hood to automatically forward the request context and headers.

也就是说 219/220 说的"useFetch 相对路径会转发头、`$fetch` 不会"，底层原因就是 useFetch 内部用的正是 `useRequestFetch`。你要自己拼 `useAsyncData` + 内部取数又想要头转发，就直接 `useRequestFetch()`；懒得管就用 `useFetch`，它已经替你转了。

还有一条 tip 说明它在浏览器侧不特殊：

> In the browser during client-side navigation, `useRequestFetch` will behave just like regular `$fetch`.

即客户端导航时它和 `$fetch` 一样（因为浏览器本来就自动带那些头，无需转发）。

## 三种"给内部请求带头"的做法对齐

| 做法 | 转发范围 | 出处 |
| --- | --- | --- |
| 手动 `headers: useRequestHeaders(['authorization'])` | 你挑的某几个头 | 315 |
| `useRequestFetch()` 实例 | 上下文 + 头（除去不转发那批） | 本篇 |
| `useFetch('/api/...', ...)` | 内部已用 useRequestFetch，自动 | 219/220 |

## 自查

- SSR 阶段 `$fetch('/api/...')` 不自动转发 cookie/authorization（安全考虑）→ 内部接口读不到登录态。
- 要转发：用 `useRequestFetch()` 拿实例来打内部请求；或直接 `useFetch`（底层已替你转）。
- 转发会剔除 `transfer-encoding`/`connection`/`keep-alive`/`upgrade`/`expect`/`host`/`accept` 这些不该转的头。
- 客户端导航时 useRequestFetch 行为等同 `$fetch`。

## 下一篇

下一篇 319 把这一层的时间线串起来：一次初始请求在服务端经历的七步（Nitro 插件 → 服务端中间件 → 应用插件 → 路由校验 → 应用中间件 → 页面与组件 → HTML 输出）。
