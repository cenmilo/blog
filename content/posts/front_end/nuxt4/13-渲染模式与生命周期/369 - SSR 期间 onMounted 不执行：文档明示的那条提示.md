---
url: /posts/bbef452595b34d69b22faf69bbfb150d/
title: 369 - SSR 期间 onMounted 不执行：文档明示的那条提示
date: 2027-02-12T09:00:00+08:00
updated: 2027-02-12T09:00:00+08:00
author: cmdragon

summary:
  摘要：`onMounted` 在服务端渲染时一次都不会调用——这不是 bug，是官方在服务端七步的"渲染取数"那一步直接写明的规则。原因也很朴素：服务端没有动态更新、没有 DOM 操作，Vue 的 `onBeforeMount`、`onMounted` 及其后的钩子在 SSR 期间"NOT executed"。配套的还有两条：SSR 期间 Vue 默认暂停依赖追踪，且服务端根本没有响应式（自上而下渲染成静态 HTML，已渲染的内容没法回头改）。这一篇讲清这条限制、它为什么会咬人，以及首屏数据该改放哪。

categories:
  - 前端开发

tags:
  - nuxt4
  - onMounted
  - SSR
  - 生命周期
  - 响应式
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


你在 Vue 里习惯这么取首屏数据：

```vue
<script setup>
import { onMounted } from 'vue'
const list = ref([])
onMounted(async () => {
  list.value = await $fetch('/api/list')
})
</script>
```

纯客户端没问题。可 Nuxt 默认是 universal 渲染，这段 `onMounted` 在服务端那一趟**根本不会执行**，于是首屏 HTML 里 `list` 是空的，数据要等浏览器水合后才填进去——SEO 拿不到、还闪一下。这不是你写错了，是框架的既定规则。

## 文档明示的那句

`guide/concepts/nuxt-lifecycle` 服务端七步的"Page and components"那步，原话：

> Since there are no dynamic updates and no DOM operations occur on the server, Vue lifecycle hooks such as `onBeforeMount`, `onMounted`, and subsequent hooks are **NOT** executed during SSR.

三个点：

- 不执行的不止 `onMounted`，`onBeforeMount` 以及**后续**的钩子（`onBeforeUpdate`、`onMounted` 之后那整串）在服务端都不跑。
- 原因写得直接：**服务端没有动态更新、没有 DOM 操作**。挂载、渲染到 DOM 这类事在服务器上无从谈起。
- 这跟 368 那句对得上："Unlike on the server, the browser executes the full Vue lifecycle"——完整 Vue 生命周期只属于客户端那一半。

## 顺带两条同源限制

同一步里官方还给了两句，帮理解"服务端那一趟到底不是完整运行时"：

> By default, Vue pauses dependency tracking during SSR for better performance.

服务端默认**暂停依赖追踪**。

::::callout
There is no reactivity on the server side because Vue SSR renders the app top-down as static HTML, making it impossible to go back and modify content that has already been rendered.
::::

服务端**没有响应式**：Vue SSR 是自上而下渲染成静态 HTML，已经渲染出去的内容没法回头改。所以别指望在服务端 setup 里"改个 ref 触发重新渲染"——那一趟只负责拼出 HTML。

## setup 顶层 vs 钩子：哪个会跑

要分清楚，服务端那一趟并非全不执行：

- `<script setup>` 的**顶层代码会跑**（比如上面的 `const list = ref([])`）。365 那张表说的"页面在两端跑"就是这个——setup 函数体两端各执行一次。
- 但 setup 里注册的 **Vue 生命周期钩子**（onMounted 等）在服务端不触发。

所以问题不在"代码跑没跑"，而在"onMounted 这个回调在服务端压根没被调用"。

## 首屏数据改放哪

既然要靠服务端把数据拼进 HTML，就别用 `onMounted` + `$fetch`，改用 SSR 友好的取数组合式——它们就是为这个设计的：`useFetch` / `useAsyncData`（在 setup 里 `await`，服务端那一步会等它们 resolve 再序列化页面）。这是取数分区的主线，本系列 210 起详讲。这一节要记住的只有一句：**首屏要出现在 HTML 里的数据，不能靠 `onMounted` 取。**

## 自查

- `onBeforeMount`、`onMounted` 及其后续 Vue 钩子，SSR 期间一律不执行（官方 "NOT executed"）。
- 根因：服务端没有动态更新、没有 DOM 操作；Vue 默认暂停依赖追踪、服务端无响应式（自上而下渲染成静态 HTML）。
- setup 顶层会跑（页面两端执行），但钩子回调在服务端不触发。
- 首屏数据别放 `onMounted`，用 `useFetch`/`useAsyncData` 让服务端等待取数后再出 HTML。

## 下一篇

下一篇 370 讲这条限制的连锁后果：为什么把 `setInterval`、事件监听这类副作用写在 `<script setup>` 根作用域会泄漏——服务端卸载钩子永远不叫，定时器就一直在。
