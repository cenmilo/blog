---
url: /posts/a7efd584b6f57554e18f271b78a3160c/
title: 为什么要 useAsyncData：同一份请求跑两遍的后果
date: 2026-12-21T09:00:00+08:00
updated: 2026-12-21T09:00:00+08:00
author: cmdragon

summary:
  摘要：从 P08 开始讲取数。第一篇先把"为什么框架要再造一个取数函数"讲清楚，答案只有一句：**同一段代码会被跑两遍**——服务端渲染 HTML 时一次，浏览器接手这段 HTML 时又一次。官方在 "The Need for `useFetch` and `useAsyncData`" 一节把这条讲完了：「If the `$fetch` function is used to perform data fetching in the setup function of a Vue component, this may cause data to be fetched twice, once on the server (to render the HTML) and once again on the client (when the HTML is hydrated). **This can cause hydration issues, increase the time to interactivity and cause unpredictable behavior.**」解法是 `useFetch` 与 `useAsyncData`：「if an API call is made on the server, the data is forwarded to the client in the payload」，而 payload 是「a JavaScript object accessible through `useNuxtApp().payload`」，专门用来在水合时避免重复取数；`useAsyncData` 那一页的 note 也说得很直白——它把响应加进 Nuxt payload，**"without re-fetching the data on client side when the page hydrates"**。三个工具的定位是官方那三行：`$fetch` 最简单、`useFetch` 是包了一层 `$fetch` 并保证"fetches data only once in universal rendering"、`useAsyncData` 与 `useFetch` 类似但给更细的控制。最后一条容易被误解的是 `await`：**`await` 不改变服务端渲染出来的 HTML**（"the fully populated result is always sent to the browser"），它改的是你后面代码的执行时机与客户端导航要不要被阻塞。

categories:
  - 前端开发

tags:
  - nuxt4
  - 取数
  - useAsyncData
  - payload
  - SSR

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


在进入正题之前把两个词说人话，后面全靠它们：

- **服务端渲染（SSR）**：服务器先把 HTML 拼好，再发给浏览器。用户第一眼看到的就是有内容的页面。
- **水合**：浏览器拿到这段 HTML 之后，Vue 在同一棵 DOM 上挂上事件与状态，页面从此可交互。

关键点在于：**这两步跑的是同一份组件代码**。

## 一遍就够的事为什么会跑两遍

官方那一节的原话（这一篇的核心就这一段）：

> Nuxt is a framework which can run isomorphic (or universal) code in both server and client environments. If the `$fetch` function is used to perform data fetching in the setup function of a Vue component, this may cause data to be fetched twice, once on the server (to render the HTML) and once again on the client (when the HTML is hydrated). This can cause hydration issues, increase the time to interactivity and cause unpredictable behavior.

翻译成一个具体症状：你在网络面板里看到同一个接口**出现两次**，一次来自文档请求的服务端渲染，一次来自浏览器启动。而后三种后果分别是：

| 原话里的后果 | 你会看到什么 |
| --- | --- |
| "hydration issues" | 水合不匹配的警告；199 那张表里"两侧数据不一致"这一类的加强版 |
| "increase the time to interactivity" | 页面已经显示出来了，但按钮要点两次才有反应 |
| "unpredictable behavior" | 最难受的一种，表现为偶发，复现靠运气 |

## 解法：把结果放进 payload 带过去

紧接着那句是解法的定义：

> The `useFetch` and `useAsyncData` composables solve this problem by ensuring that if an API call is made on the server, the data is forwarded to the client in the payload.

> The payload is a JavaScript object accessible through `useNuxtApp().payload`. It is used on the client to avoid refetching the same data when the code is executed in the browser during hydration.

官方还给了看它的地方：**"Use the Nuxt DevTools to inspect this data in the Payload tab."** 我建议你今天就把这一条用起来——**它是唯一一个能让你亲眼确认"数据真的从服务端带过来了"的位置**，比读代码猜快得多。

`useAsyncData` 那一页的 note 是同一定义的另一种写法，措辞更硬：

> `useAsyncData` is a composable meant to be called directly in the Nuxt context. It returns reactive composables and handles adding responses to the Nuxt payload so they can be passed from server to client **without re-fetching the data on client side** when the page hydrates.

注意里面两个限定：一是**要直接在 Nuxt 上下文里调用**（这条限制在 151 那张表里对应"composable 只能在插件/路由中间件/setup 同步调用"那一类），二是它返回的是**响应式对象**，所以数据填进来时模板会自己更新。

## 官方对三个工具的分工（三行原话）

| 工具 | 官方定位（逐字） |
| --- | --- |
| `$fetch` | "is the simplest way to make a network request." |
| `useFetch` | "is a wrapper around `$fetch` that fetches data only once in universal rendering." |
| `useAsyncData` | "is similar to `useFetch` but offers more fine-grained control." |

同节还有一句常被忽略：**"Both `useFetch` and `useAsyncData` share a common set of options and patterns that we will detail in the last sections."** ——这两个组合式的选项是同一套，所以学一次就够（213 那张选项表两边通用）。

官方那个对照示例把三者关系摆在一起，值得逐行看：

```vue [app/app.vue]
<script setup lang="ts">
const { data } = await useFetch('/api/data')

async function handleFormSubmit () {
  const res = await $fetch('/api/submit', {
    method: 'POST',
    body: {
      // My form data
    },
  })
}
</script>
```

> In the example above, `useFetch` would make sure that the request would occur on the server and is properly forwarded to the browser. `$fetch` has no such mechanism and is a better option to use when the request is solely made from the browser.

这个例子的排布本身就是一条规则：**首屏要显示的数据走 `useFetch`，点提交按钮才发的请求走 `$fetch`**。后者不会被带进 payload，也不该被带进去。

## `await` 的真正作用范围

这一节我要单独拿出来，因为它把很多人（包括曾经的我）的错误直觉纠正了。官方原话：

> Examples in this documentation usually `await` calls to `useFetch` and `useAsyncData`, but this is not always required.
>
> The `await` does **not** change the server-rendered HTML. During server rendering, Nuxt waits for the request to resolve before serializing the page either way (`<Suspense>`, and `onServerPrefetch` under the hood), so the fully populated result is always sent to the browser.

那 `await` 改了什么？文档给的两条：

- **With `await`**：执行暂停直到数据就绪，后面的代码可以放心假设 `data` 已有值；客户端导航时**阻塞导航**，用户留在原页（可配 `<NuxtLoadingIndicator>`），落到的页一定是填满数据的——官方标 "This is the default behaviour."
- **Without `await`**：立刻继续往下走，`data` 先是默认值；客户端导航**立即发生**，加载与错误状态由你自己处理，通常靠返回的 `status` 与 `error`。

还有一句我读了两遍才接受："Neither approach is universally better; the right choice depends on the experience you want for that route." ——官方明确不站队。

**`await` 与 `lazy` 不是一回事**，这一点文档特意写了区别："`lazy` is an explicit flag that defers the request until the component mounts, whereas simply not awaiting starts the request during setup. Prefer `lazy` (or `useLazyFetch` / `useLazyAsyncData`) when you want non-blocking behaviour, since it makes the intent explicit."

然后是那条 warning，讲的是最容易写错的一种组合：

> The `await` and `lazy` are independent, and on the client awaiting a `lazy` function won't have the effect you expect. If you `await` a `lazy` call (for example `await useLazyFetch(...)` or `await useFetch(..., { lazy: true })`), it still blocks server rendering as usual, but on **client-side navigation** the `await` resolves immediately without waiting for the request. `data` will still be at its default value right after the `await`, and you must handle the loading state via `status`. If you actually want the navigation to wait for the data, drop the `lazy` option rather than relying on `await`.

症状描述得很精确：**写了 `await` 但 `data` 还是默认值**，因为 `lazy` 让那个 `await` 在客户端导航时立刻返回。解法官方直接给了：想要等待就别用 `lazy`，别指望 `await` 拦住它。

## Suspense：这层机制挂在哪

官方给的解释只有一句，但它说明了"为什么阻塞是可能的"：

> Nuxt uses Vue's `<Suspense>` component under the hood to prevent navigation before every async data is available to the view. The data fetching composables can help you leverage this feature and use what suits best on a per-call basis.

配套还有一条 note：**"You can add the `<NuxtLoadingIndicator>` to add a progress bar between page navigations."** ——导航被阻塞时给用户反馈，这是官方给的那块牌子（这个组件的六个 prop 与进度曲线在 102/146 讲过）。

## 顺手记下与本篇相邻的两条

- **服务端调用内部 API 时 `$fetch` 不真的发请求**：`api/utils/$fetch` 那页 tip 写 "During server-side rendering, calling `$fetch` to fetch your internal API routes will directly call the relevant function (emulating the request), **saving an additional API call**." 这是 211 的主线，这里只提一句免得你以为服务端那次一定是一次网络往返。
- **取数函数是保留名**：`useAsyncData` 那页 warning 写 "`useAsyncData` is a reserved function name transformed by the compiler, so you should not name your own function `useAsyncData`." ——自己封的包装函数别叫这个名字。

## 自查

1. 为什么同一份请求会发两次？（答：服务端渲染与浏览器水合跑的是同一份组件代码，`$fetch` 不把结果带到客户端。）
2. 怎么确认数据确实从服务端带过去了？（答：Nuxt DevTools 的 Payload 标签页看 payload，或读 `useNuxtApp().payload`。）
3. 写了 `await useFetch(..., { lazy: true })` 为什么数据还是空的？（答：官方那条 warning——客户端导航时 `await` 会立即返回，`data` 仍是默认值，加载状态得用 `status` 处理；要等待就别加 `lazy`。）

## 下一篇

211 - 裸用取数助手会跑两遍：去重与导航阻止都没有

## 本文依据

- `getting-started/data-fetching`：三工具清单与 "In a nutshell" 三行；"Both `useFetch` and `useAsyncData` share a common set of options and patterns..."；The Need for 一节两段原文（isomorphic/universal、跑两次、三条后果）；payload 两段与 DevTools **Payload tab** 那条 tip；`useFetch` + `$fetch` 的示例与其后两句解释；Suspense 一节与 `<NuxtLoadingIndicator>` note；A note on `await` 全节（不改变服务端 HTML、with/without 两条、"Neither approach is universally better"、`lazy` 与不加 `await` 的区别句、await 一个 lazy 调用的 warning 全文）。
- `api/composables/use-async-data`：note "meant to be called directly in the Nuxt context... without re-fetching the data on client side when the page hydrates"；"You do not need to `await` `useAsyncData`..." 那条 note；reserved function name 的 warning。
- `api/utils/dollarfetch`：SSR 调用内部 API 路由会直接调函数、省一次请求的那条 tip（留给 211 展开）。
