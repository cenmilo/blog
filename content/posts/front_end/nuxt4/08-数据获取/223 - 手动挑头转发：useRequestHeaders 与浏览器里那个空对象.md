---
url: /posts/1efabe7832c8f08e572dd92a76115137/
title: 223 - 手动挑头转发：useRequestHeaders 与浏览器里那个空对象
date: 2026-12-25T10:00:00+08:00
updated: 2026-12-25T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：222 讲了自动代理，这一篇讲手动那一路：只挑自己需要的头带过去。`useRequestHeaders` 的定位是 "Use useRequestHeaders to access the incoming request headers."，可在页面、组件与插件里用；**不传参数就是取全部头，传数组就是只取那几项**：`const headers = useRequestHeaders()` 与 `const { cookie } = useRequestHeaders(['cookie'])`。最容易误判的一条是官方那条 tip：**"In the browser, `useRequestHeaders` will return an empty object."**——它在浏览器里不报错、只是给你空对象，所以"登录态在客户端导航时丢了"往往就是拿它当万能读头工具用。官方给的用途是把首个请求的 `authorization` 头代理给后续内部请求，示例只有一行：`useFetch('/api/confidential', { headers: useRequestHeaders(['authorization']) })`。这一篇把三种做法摆在一起比较（`useFetch` 相对路径自动代理、`useRequestFetch()`、`useRequestHeaders` 手挑），并说明各自什么时候用；另外要提醒：手挑的头会随请求发出去，211 那份"不该转发的头"清单在这里同样适用。

categories:
  - 前端开发

tags:
  - nuxt4
  - 请求头
  - cookie
  - useRequestHeaders
  - SSR

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


前情提要一下：浏览器发请求时自己会带上 `cookie` 这些头，但服务端替你发的那一次不会自动带（211/222）。官方给的解法有两层——自动代理与手动挑，这一篇是后者。

## 两种取法，官方示例四行

```ts
// Get all request headers
const headers = useRequestHeaders()

// Get only cookie request header
const { cookie } = useRequestHeaders(['cookie'])
```

第二行这个解构值得停一下：**参数数组控制"取哪几项"，返回值是一个对象**，所以只取一项就解构一项。它适合放在 `useFetch` 的 `headers` 选项里，也适合自己拼给 `$fetch`。

## 那条 tip 是这一页最重要的内容

> In the browser, `useRequestHeaders` will return an empty object.

我把它翻译成症状：**同一段代码，服务端渲染时能读到 cookie，客户端导航时读到空对象**。因为浏览器里压根没有"进来的那个请求"可读——那一次是本地导航，不是 HTTP 请求。

这解释了两类诡异现象：

| 现象 | 原因 |
| --- | --- |
| 首屏正常，点进去的页面变成未登录 | 首屏是服务端渲染（能读到头），后续是客户端导航（空对象） |
| 调试时 `console.log(headers)` 有值，上线偶发失效 | 只有首屏那次有值，其余时刻是空 |

所以我的判断是：**如果你需要"两侧都有"的凭据，别用请求头当唯一来源**——官方在别处给过 SSR 友好的替代品，比如 `useCookie`（199 那张表里，`localStorage` 的官方解法就是它，理由句是 "This works on both server and client"）。

## 官方给的用途：代理 authorization

Example 一节整段：

> We can use `useRequestHeaders` to access and proxy the initial request's `authorization` header to any future internal requests during SSR.
>
> The example below adds the `authorization` request header to an isomorphic `$fetch` call.

```vue [app/pages/some-page.vue]
<script setup lang="ts">
const { data } = await useFetch('/api/confidential', {
  headers: useRequestHeaders(['authorization']),
})
</script>
```

注意它写的是 "to any future internal requests **during SSR**"——**适用范围限定在服务端那一趟**。这句话和那条 tip 是同一件事的两面：只在 SSR 有意义，因为浏览器自己会带。

## 三种做法放一起选

| 做法 | 谁转发 | 粒度 | 什么时候用 |
| --- | --- | --- | --- |
| `useFetch('/api/...', {...})` 相对路径 | Nuxt 自动（底层 `useRequestFetch`） | 自动，排除不该转发的头 | 默认选择，首屏数据 |
| `const f = useRequestFetch(); f('/api/...')` | 你显式拿到那个代理函数 | 自动，但作用域是你包的那次 | 包在 `useAsyncData` 里、或换掉裸 `$fetch` 时 |
| `useRequestHeaders(['cookie'])` | 你手动挑具体头 | **精确到某几个头** | 只想带一两个头、或要把头放进自定义客户端 |

三条都只解决"服务端那一趟"的问题；222 那句 "In the browser during client-side navigation, `useRequestFetch` will behave just like regular `$fetch`." 说明第一条与第二条在浏览器侧就是普通请求。

## 手挑不等于随便挑

官方在 `useFetch`/`$fetch` 那两处都给了不该转发的清单（211 抄过：`host`、`accept`、`content-length`、`content-md5`、`content-type`、`x-forwarded-*`、`cf-connecting-ip`、`cf-ray`；222 又给了 `transfer-encoding`、`connection`、`keep-alive`、`upgrade`、`expect`）。

`useRequestHeaders(['cookie'])` 这种写法很轻，但风险不轻：**你写进数组的每一个头，都会被原样发给目标接口**。所以我的习惯是只写当前接口真的需要的那一个，别顺手 `useRequestHeaders()` 全量塞进 `headers`——那行示例确实展示了"取全部"，但官方同一节也在提醒 "Be very careful before proxying headers to an external API and just include headers that you need."（211 引的 caution）。

## 怎么确认头到底带过去了

最省事的是官方在 222 那页给的那个探针接口：写一个把 cookie 回显出来的内部路由（`parseCookies(event)` 后返回 `{ cookies }`），然后三种写法各跑一次：

- 服务端渲染时返回 `{ cookies: {} }` ⇒ 那次没转发（多半用了裸 `$fetch`）。
- 返回 `{ cookies: { foo: 'bar' } }` ⇒ 转发生效。

这个测试的价值在于**它把"哪一侧跑的"和"带头了没有"两件事一次分清**，比读代码猜快。

## 自查

1. 在浏览器里调 `useRequestHeaders(['cookie'])` 得到什么？（答：空对象——官方 tip 明写 "In the browser, `useRequestHeaders` will return an empty object."。）
2. 官方给的适用时机是哪一句？（答："to any future internal requests during SSR"。）
3. 想要首屏和客户端导航都能读到同一份凭据，官方推荐的替代是什么？（答：走 SSR 友好的状态/存储，例如 `useCookie`，它两侧都能用。）

## 下一篇

224 - useLazyAsyncData：它只是把 lazy 设成 true 的那层壳

## 本文依据

- `api/composables/use-request-headers`：定位句与适用范围（"within your pages, components, and plugins"）；两种取法示例（全量与 `['cookie']` 解构）；tip "In the browser, `useRequestHeaders` will return an empty object."；Example 段两句与 `headers: useRequestHeaders(['authorization'])` 示例。
- `api/composables/use-request-fetch`：自动代理定位句、七个不转发的头、两条 tip（`useFetch` 底层使用它、客户端导航等同 `$fetch`）、`{ cookies: { foo: 'bar' } }` 与 `{ cookies: {} }` 的正反读数（222 已逐句引）。
- `api/utils/dollarfetch`：SSR 不带 cookie 的理由（SSRF / Authentication Misuse）、不该代理的头清单、`useFetch` 相对路径自动代理那句（211 已逐句引）。
- `guide/best-practices/hydration`：`localStorage` 案例与 `useCookie` 解法及 "This works on both server and client"（199 已逐句引）。
