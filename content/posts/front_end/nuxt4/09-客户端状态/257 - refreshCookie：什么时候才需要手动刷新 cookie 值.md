---
url: /posts/795bdd5c17e622b4f6bacba3883b212d/
title: 257 - refreshCookie：什么时候才需要手动刷新 cookie 值
date: 2027-01-05T11:00:00+08:00
updated: 2027-01-05T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`refreshCookie` 解决一个很窄但很难自己想到的问题：**cookie 已经在浏览器里变了，但 `useCookie` 那个 ref 还是旧值**。官方定位句 "Refresh useCookie values manually when a cookie has changed"（v3.10 起可用），Purpose 两句是 "The `refreshCookie` function is designed to refresh cookie value returned by `useCookie`. **This is useful for updating the `useCookie` ref when we know the new cookie value has been set in the browser.**"；签名只有一行 `refreshCookie (name: string): void`——**传的是 cookie 名字，不是 ref**。官方示例给的是登录后刷新：`const tokenCookie = useCookie('token')`，在 `login` 里 `await $fetch('/api/token', {...}) // Sets token cookie on response` 之后 `refreshCookie('token')`，再配 `const loggedIn = computed(() => !!tokenCookie.value)`。**而那一节最重要的一句其实是那条 note**："Since Nuxt v3.12.0, the experimental `cookieStore` option is **enabled by default**. It automatically refreshes the `useCookie` value when cookies change in the browser."——也就是说默认配置下这件事是自动的，`refreshCookie` 属于兜底；而 `useCookie` 的选项表里 `watch` 那行还挂着同一句提醒："Note: Refresh `useCookie` values manually when a cookie has changed with `refreshCookie`."

categories:
  - 前端开发

tags:
  - nuxt4
  - refreshCookie
  - cookie
  - 响应式
  - 登录

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


场景很具体：接口在响应里用 `Set-Cookie` 种了一个 token，浏览器 DevTools 里能看到它，但页面上 `useCookie('token').value` 还是空——因为那个 ref 是在 setup 时读的值，它不知道外部发生了什么。

## 它是什么

> Refresh useCookie values manually when a cookie has changed

版本要求是一条 important：**"This utility is available since Nuxt v3.10."**

Purpose 两句：

> The `refreshCookie` function is designed to refresh cookie value returned by `useCookie`.
> This is useful for updating the `useCookie` ref when we know the new cookie value has been set in the browser.

签名一行：

```ts
export function refreshCookie (name: string): void
```

三点从签名就能读出来：**参数是 cookie 的名字（字符串）**、**只作用在一个名字上**、**返回 `void`（不能 await，也不告诉你刷没刷到）**。

## 官方示例：登录后刷新会话

```vue [app/app.vue]
<script setup lang="ts">
const tokenCookie = useCookie('token')

const login = async (username, password) => {
  const token = await $fetch('/api/token', { /** ... */ }) // Sets `token` cookie on response
  refreshCookie('token')
}

const loggedIn = computed(() => !!tokenCookie.value)
</script>
```

这段的形状值得照抄，因为它把三件事接上了：

1. **写 cookie 的动作在服务端**（注释 "Sets `token` cookie on response"），所以前端这边没有赋值机会。
2. `refreshCookie('token')` 是**"我知道它变了，请重读"** 的信号。
3. UI 判断走 `computed`，所以刷新之后 `loggedIn` 会跟着变——**这一环依赖的是 ref 被更新，而不是组件重新挂载**。

## 但默认配置下，它多半不需要你调

这一页最重要的一条不是用法，是那条 note：

> Since [Nuxt v3.12.0], the experimental `cookieStore` option is **enabled by default**. It automatically refreshes the `useCookie` value when cookies change in the browser.

也就是说：**现代浏览器里这件事已经自动化了**。`useCookie` 那页 `watch` 行的提醒则是同一件事的另一种口径：

> **Note:** Refresh `useCookie` values manually when a cookie has changed with `refreshCookie`.

**两条并排读会打架**：一条说自动，一条说手动。差异点很可能在"是否支持 CookieStore"（`cookieStore` 那节的措辞是 "to listen for cookie updates (**if supported by the browser**) and refresh `useCookie` ref values"）——不支持的浏览器就退回到需要 `refreshCookie`。但官方没有把这两句写成一个判断表，所以我给的只是排查顺序，不是结论：

1. 先确认你的表现是不是"ref 不更新"（在 `login` 之后打一次 `tokenCookie.value`）。
2. 如果确实不更新，先查 `experimental.cookieStore` 有没有被关掉（258）。
3. 环境或浏览器不支持 CookieStore 时，就按官方示例显式调 `refreshCookie(name)`。

## 三个边界，文档没写的我不猜

| 我想知道 | 状态 |
| --- | --- |
| `refreshCookie` 在服务端调用会怎样 | 未写。它的定位是"浏览器里已经种好了新 cookie"，服务端那次并没有浏览器可同步 |
| 刷新与 `default` 的先后 | 未写。我只确认参数表里 `default` 是 "returning the default value **if the cookie does not exist**"（256） |
| 它会不会触发 `watch` | 未写。但官方示例正是靠 `computed` 读它，说明 ref 变化能被响应式系统看到 |

还有一个方向性错误值得写出来：**`refreshCookie` 不写 cookie，也不删 cookie**——它只让 `useCookie` 重新读一次。删除要按 255 那样赋值 `null`（并且"是否真的清掉浏览器里的 cookie"文档未写，需要自己验），过期要按 256 的 `maxAge`/`expires`。**这三件事在中文里都叫"清 cookie"，在文档里是三个不同的动作。**

## 自查

1. `refreshCookie` 的参数与返回是什么？（答：`name: string`、返回 `void`；作用对象是同名 `useCookie` 的 ref。）
2. 默认情况下为什么多数时候不用手动调它？（答：`experimental.cookieStore` 自 v3.12.0 起默认开启，浏览器里 cookie 变化会自动刷新 `useCookie` 的值。）
3. "刷新、赋 null、过期"三件事分别由什么负责？（答：`refreshCookie` 只重读；赋 `null` 是 255 的重置写法；到期由 `maxAge`/`expires` 决定，见 256。）

## 下一篇

258 - cookieStore 实验开关：它让手动刷新变成自动
