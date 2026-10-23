---
url: /posts/f47f0635264b5c50f90d371a21ce8eb5/
title: 255 - useCookie：一个能在水合两侧写的 ref
date: 2027-01-05T09:00:00+08:00
updated: 2027-01-05T09:00:00+08:00
author: cmdragon

summary:
  摘要：250 解释过 `useState` 只在一次加载里活着，要跨请求持久就得用 cookie。官方定位句是 "useCookie is an SSR-friendly composable to read and write cookies."，写法 `const cookie = useCookie(name, options)`，可用位置是 "Within your pages, components, and plugins"，一条硬约束是 note 里的 "`useCookie` only works in the Nuxt context."。**它的返回值语义在 Return Values 里写得很完整**："Returns a Vue `Ref<T>` representing the cookie value. **Updating the ref will update the cookie (unless `readonly` is set).** The ref is SSR-friendly and will work on both client and server."——"两侧都能用"正是它相对 `localStorage` 的价值（199 那张成因表把 `localStorage` 列为第一类错配原因，解法官方给的就是 `useCookie`，理由句 "This works on both server and client"）。序列化不用你管：tip 是 "The returned ref will automatically serialize and deserialize cookie values to JSON."，参数说明里补一句 "Most of the options will be directly passed to the cookie package."。官方 Basic Usage 那段还有一个我原来没注意的用法：**把 ref 赋成 `null` 是重置**（`<button @click="counter = null">reset</button>`），而整段说明是 "Whenever we update the `counter` variable, the cookie will be updated accordingly."。最后一条跨层：服务端接口里读写要换 h3 的两个 helper——"You can use `getCookie` and `setCookie` from `h3` package to set cookies in server API routes."

categories:
  - 前端开发

tags:
  - nuxt4
  - useCookie
  - cookie
  - SSR
  - 状态

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


cookie 是一个能被服务端与浏览器共同看见的字符串，所以"这个用户上次选了什么"这类状态放它里面，才不会出现服务端读不到的情况。先声明：这一篇只讲概念与官方那四段，**14 个选项逐条在下一篇**。

## 它是什么，能在哪用

> useCookie is an SSR-friendly composable to read and write cookies.

> Within your pages, components, and plugins, you can use `useCookie` to read and write cookies in an SSR-friendly way.

```ts [Usage]
const cookie = useCookie(name, options)
```

两条限制各一句：

- note："`useCookie` only works in the [Nuxt context]."（也就是 233 那张表：`setup`、插件、路由中间件这些位置，异步回调里会丢）
- 签名 `useCookie<T = string | null | undefined> (name: string, options?: CookieOptions<T>): CookieRef<T>`——**泛型默认是 `string | null | undefined`**，这点很实在：cookie 不存在时它就是 `null`（`CookieRef<T> extends Ref<T> {}`，所以它就是个 ref）。

## 读写语义：赋值即写 cookie

Return Values 整段我逐字抄，因为它把三个问题一起答了：

> Returns a Vue `Ref<T>` representing the cookie value. Updating the ref will update the cookie (unless `readonly` is set). The ref is SSR-friendly and will work on both client and server.

| 问题 | 答案 |
| --- | --- |
| 读 | 直接读 `.value`（模板里自动解包） |
| 写 | 给 `.value` 赋值就会更新 cookie，**除非设了 `readonly`** |
| 哪一侧能用 | 两侧都能（这就是它替代 `localStorage` 的理由） |

序列化也不用手写："The returned ref will automatically serialize and deserialize cookie values to JSON." ——注意**这一句是 JSON**，和 `useState` 的口径一致（252）：复杂类型不要指望它，具体编解码可以自定义（256 讲 `encode`/`decode`）。

## 官方 Basic Usage：那个 reset 按钮

> The example below creates a cookie called `counter`. If the cookie doesn't exist, it is initially set to a random value. **Whenever we update the `counter` variable, the cookie will be updated accordingly.**

```vue [app/app.vue]
<script setup lang="ts">
const counter = useCookie('counter')

counter.value ||= Math.round(Math.random() * 1000)
</script>

<template>
  <div>
    <h1>Counter: {{ counter || '-' }}</h1>
    <button @click="counter = null">
      reset
    </button>
    <button @click="counter--">
      -
    </button>
    <button @click="counter++">
      +
    </button>
  </div>
</template>
```

三处我读出来的写法细节：

1. `counter.value ||= ...` —— **先填充再渲染**，避免首帧空值（这是"cookie 不存在"时的常见处理）。
2. **`counter = null` 是重置**：模板里直接对 ref 赋值，`null` 表示"没有值"。官方没有另外提供 `remove()` 之类的 API，这一页只有这一个删值方式（**是否真的把浏览器里的 cookie 删掉、还是留一个值为空的 cookie，文档没写**，257 会连清理一起讲）。
3. 模板里 `{{ counter || '-' }}` 说明作者预期它会读到空。

## 与服务端接口里的 cookie 是两套 API

官方在 Cookies in API Routes 一节给的是 h3 的两个 helper：

> You can use `getCookie` and `setCookie` from `h3` package to set cookies in server API routes.

```ts [server/api/counter.ts]
export default defineEventHandler((event) => {
  // Read counter cookie
  let counter = getCookie(event, 'counter') || 0

  // Increase counter cookie by 1
  setCookie(event, 'counter', ++counter)

  // Send JSON response
  return { counter }
})
```

**这条边界我记成一句**：`useCookie` 属于 Vue 侧（`setup`、插件、中间件），服务端事件处理里用 `getCookie`/`setCookie`——两边操作的是同一个浏览器 cookie，但 API 不通用。151 早记过"服务端那侧同名工具其实是 h3 helper"这同一类事实。

## 与 `useState` 的分工

| | `useState` | `useCookie` |
| --- | --- | --- |
| 活多久 | 一次加载（SSR 与它水合出来的客户端之间传递，250/252） | 直到过期或被清（`maxAge`/`expires`，256） |
| 服务端能读吗 | 能（同一次请求内） | 能（它本来就是随请求进来的头，223） |
| 刷新页面后还在吗 | **不在** | **在** |
| 能放什么 | 可序列化值；类/函数/symbol 不要放（251） | 会被 JSON 序列化的值（tip 那句） |
| 清理方式 | `clearNuxtState`（253） | 赋 `null`，或让 `expires` 到期 |

`useState` 那页的 important 和 199 那张表说的其实是同一件事的两面：**要跨请求，就别把状态只放在内存里**。官方对 `localStorage` 的态度我也在 260 一起处理——它没有"推荐"也没有"禁用"，只出现在错配反例里，而那一格给的解法正是 `useCookie`。

## 自查

1. `useCookie` 返回值更新 cookie 的条件是什么？（答：给 `.value` 赋值即更新，除非设了 `readonly`。）
2. 泛型默认值暗示了什么？（答：`T = string | null | undefined`——cookie 可能不存在，读回来是 `null`。）
3. 服务端接口里读写 cookie 用什么？为什么不是 `useCookie`？（答：`getCookie`/`setCookie`（h3），因为 `useCookie` 只在 Nuxt 上下文的 Vue 侧可用。）

## 下一篇

256 - useCookie 选项表逐条：14 个键与两处要小心
