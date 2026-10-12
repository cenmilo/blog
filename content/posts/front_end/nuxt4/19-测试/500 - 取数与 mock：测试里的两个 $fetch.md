---
url: /posts/098e1fb2f405f92ce80689f0db63649e/
title: 取数与 mock：测试里的两个 $fetch
date: 2027-03-27T11:00:00+08:00
updated: 2027-03-27T11:00:00+08:00
author: cmdragon

summary:
  摘要：测试文件里写 `$fetch`，先看清你是从哪儿引的。从 `@nuxt/test-utils/e2e` 引，它是"取一个服务端渲染页面的 HTML"的工具；不引而直接用，那是 Nuxt 通过 ofetch 全局暴露的 `$fetch`，在你的 Vue 应用或 API 路由里发 HTTP 请求。两者行为差别很大，而单元测试环境里还有第三种情况：你注册的假接口。这一篇把三条通路对齐讲清楚，包括 SSR 期间 `$fetch` 内部路由会被直接调用（省掉一次真实 API 请求）这条提示对造 mock 意味着什么，以及 cookies 在服务端不会被带上这条安全限制。

categories:
  - 前端开发

tags:
  - nuxt4
  - 取数
  - fetch
  - mock
  - 测试
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

同一个测试文件里，`$fetch('/api/item')` 有三种可能的结果：拿到整页 HTML、拿到接口 JSON，或者请求根本没被打到。差别只在**你是从哪拿到这个 `$fetch` 的**。

## 三个身份先分清

| 你写的 | 来源 | 官方定义 | 返回 |
| --- | --- | --- | --- |
| `import { $fetch } from '@nuxt/test-utils/e2e'` | 测试工具 | Get the HTML of a server-rendered page | 页面 HTML 字符串 |
| 直接用 `$fetch`（不 import） | Nuxt 全局自动导入 | Nuxt uses ofetch to expose globally the `$fetch` helper for making HTTP requests | 响应数据 |
| `registerEndpoint('/test/', ...)` 造出来的接口 | 测试工具（runtime 侧） | create Nitro endpoint that returns mocked data | 你写的假数据 |

第一个只在端到端测试里用，路径指向的是**页面**，示例是 `$fetch('/')`。第二个是你业务代码里的取数方式，官方定义那句限定得很清楚：用来在 "within your Vue app or API routes" 里发 HTTP 请求。第三个不是 `$fetch`，但它决定了第二个在测试里能不能拿到东西。

## 内部路由在 SSR 期间不发网络请求

`api/utils/$fetch` 那页有一条 `::tip`，是理解测试行为的关键：

> During server-side rendering, calling `$fetch` to fetch your internal API routes will directly call the relevant function (emulating the request), **saving an additional API call**.

也就是：**服务端渲染期间请求自己的 API 路由，并不会真的走一次网络**，而是直接调用对应的处理函数，模拟这次请求。

同页另一条 `::note` 是配套的另一半：在组件里直接用 `$fetch` 而不包 `useAsyncData`，会导致取两次——服务端一次、客户端水合时再一次，因为 `$fetch` 不把状态从服务端传给客户端。官方建议是 `useFetch`，或 `useAsyncData` + `$fetch`：

```vue [app/app.vue]
<script setup lang="ts">
// During SSR data is fetched twice, once on the server and once on the client.
const dataTwice = await $fetch('/api/item')

// During SSR data is fetched only on the server side and transferred to the client.
const { data } = await useAsyncData('item', () => $fetch('/api/item'))

// You can also useFetch as shortcut of useAsyncData + $fetch
const { data } = await useFetch('/api/item')
</script>
```

把这两条搬到测试里，你会遇到一个文档没有直接回答的情况：既然服务端渲染时内部路由是**直接调用处理函数**，那用 `registerEndpoint` 注册的假接口能不能拦下它？

**文档未说明这一点。** 能确证的只有一句：`registerEndpoint` 的用途说明是"当组件为了展示数据而请求 API 时"用它，配合的 Note 讲的是外部 API 走 `baseURL` 那条路。要判断你项目里的实际拦截效果，最省事的做法是跑一次对照：同一个组件，先用 `registerEndpoint` 注册 `/api/item` 返回 `{ name: 'mocked' }`，再在 `server/api/item.ts` 里返回不同数据，看渲染结果里出现的是哪一份。出现的若是真实处理函数的数据，说明这条通路没被拦，就得改用别的手段（比如 `mockNuxtImport` 换掉取数用的组合式函数）。

## 只在客户端执行的方法里可以放心用

那页给的第二个用法示例是表单提交：

```vue [app/pages/contact.vue]
<script setup lang="ts">
async function contactForm () {
  await $fetch('/api/contact', {
    method: 'POST',
    body: { hello: 'world' },
  })
}
</script>

<template>
  <button @click="contactForm">
    Contact
  </button>
</template>
```

配套那句是 "You can use `$fetch` in any methods that are executed only on client-side."。对测试的含义很直接：**这种要点击才发生的请求，在只挂组件的单元测试里不会自动执行**，你得触发那个 `click`（`@vue/test-utils` 的触发方式，或者 `renderSuspended` 那边用 Testing Library 的 `fireEvent`），或者走端到端在真浏览器里点。

另外两条属于背景知识，也一并留着：`$fetch` 是 Nuxt 里发 HTTP 调用的首选方式，取代 Nuxt 2 时代的 `@nuxt/http` 与 `@nuxtjs/axios`；用 `$fetch` 请求自签名证书的 HTTPS 地址时，开发环境要在环境里设 `NODE_TLS_REJECT_UNAUTHORIZED=0`。后一条只适用于本机开发——它关掉的是证书校验，别把它写进任何会跑真实流量的环境里，自签场景更稳妥的做法是让运行环境信任那张证书的签发 CA。

## cookies 不会被带上，这是有意的

那页最后一个小节 Pass Headers and Cookies 讲了一条安全限制，测试里踩到会很费解：

> When we call `$fetch` in the browser, user headers like `cookie` will be directly sent to the API.

但：

> However, during Server-Side Rendering, due to security risks such as Server-Side Request Forgery (SSRF) or Authentication Misuse, the `$fetch` wouldn't include the user's browser cookies, nor pass on cookies from the fetch response.

官方给的示例组合是 `useAsyncData(() => $fetch('/api/cookies'))` 配 `server/api/cookies.ts` 里 `getCookie(event, 'foo')`，注释写着 "This will NOT forward headers or cookies during SSR"。要转发必须手动传：

```vue [app/pages/index.vue]
<script setup lang="ts">
// This will forward the user's headers and cookies to `/api/cookies`
const requestFetch = useRequestFetch()
const { data } = await useAsyncData(() => requestFetch('/api/cookies'))
</script>
```

还有一句补充：在服务端用**相对 URL** 调 `useFetch` 时，Nuxt 会用 `useRequestFetch` 来代理请求头和 cookies，`host` 这类不该转发的头除外。

对测试的影响是这样的：端到端测试里如果登录态靠 cookie，而你的页面是在服务端用 `$fetch` 去取数据，那这条链路在服务端拿不到浏览器 cookie——**这属于设计如此，不是测试环境坏了**。要造带登录态的测试，路径是 `useRequestFetch`，或者在浏览器侧完成请求。

## 自查

- 三个 `$fetch`：`/e2e` 那个取页面 HTML，全局那个在应用与 API 路由里发请求，`registerEndpoint` 造的是假接口。
- SSR 期间请求内部 API 路由是直接调用处理函数、省一次真实调用；不包 `useAsyncData` 会取两次。
- 「`registerEndpoint` 能否拦下 SSR 期间的内部调用」文档未说明，用两份不同返回值做一次对照测试来判断。
- 只在客户端执行的方法里的请求，单元测试要主动触发事件才会发生。
- 服务端渲染时 `$fetch` 不带浏览器 cookie、也不回传响应的 cookie，这是防 SSRF 与认证滥用的设计；要转发用 `useRequestFetch`。

## 下一篇

下一篇 501 讲模块作者怎么测自己的模块：fixture 应用的组织、`setup` 指向 fixture 的写法，以及那半张还开着的单元测试讨论。

## 本文依据

- https://nuxt.com/docs/4.x/api/utils/$fetch（ofetch 全局暴露的定义句、SSR 期间直接调用内部路由处理函数的 `::tip`、不包 `useAsyncData` 会双取的 `::note`、三段对比示例、只在客户端执行的方法示例与 POST 用法、取代 `@nuxt/http`/`@nuxtjs/axios` 的 `::tip`、自签名证书需 `NODE_TLS_REJECT_UNAUTHORIZED=0` 的 `::note`、Pass Headers and Cookies 小节三条与 `useRequestFetch` 示例）
- https://nuxt.com/docs/4.x/getting-started/testing（e2e 的 `$fetch(url)` 定义、`registerEndpoint` 一节两个参数与默认 GET 及外部 API 走 `baseURL` + 环境覆盖 `$test` 的 Note、`mountSuspended`/`renderSuspended` 与 `fireEvent` 的对应关系）
