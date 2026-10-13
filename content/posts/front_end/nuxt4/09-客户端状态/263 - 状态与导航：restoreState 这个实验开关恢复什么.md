---
url: /posts/3ad86b19db9ddce43e05fb30b8e40665/
title: 263 - 状态与导航：restoreState 这个实验开关恢复什么
date: 2027-01-07T11:00:00+08:00
updated: 2027-01-07T11:00:00+08:00
author: cmdragon

summary:
  摘要：前面几篇的 useState 都在讲"一次加载中怎么把状态留住"。restoreState 处理的是另一种场景：页面因为 chunk 加载失败而重载、或者你手动调用 reloadNuxtApp 之后，怎么把之前的状态找回来——办法是把它存进 sessionStorage。这一篇把这个实验开关的定位句、它的副作用（首屏可能闪一下）和那个关于 key 的重要提醒都摊开，并且说清一个容易混的点：它恢复的是"重载之后"，跟 SSR 水合不是一回事。

categories:
  - 前端开发

tags:
  - nuxt4
  - restoreState
  - useState
  - 实验特性
  - 水合
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

用户点了个链接，浏览器去拉那个页面对应的 JavaScript 文件（chunk），结果没拉下来——网络抖了一下、或者你刚部署了新包、旧文件名的 chunk 已经没了。这时 Nuxt 会重载页面。问题是：重载之前用户填了一半的表单、选了半天的筛选条件，全没了。`restoreState` 就是拿来救这个的。

## 它是什么：把状态塞进 sessionStorage，重载后找回

官方在实验特性那一节给 `restoreState` 的定位句写得很具体：

> Allows Nuxt app state to be restored from `sessionStorage` when reloading the page after a chunk error or manual `reloadNuxtApp()` call.

一句一句拆给第一次听到的人：

- **chunk error**：打包后代码被拆成一个个小文件（chunk），点某个页面才去下载对应那个。下载失败就是 chunk error。
- **manual `reloadNuxtApp()` call**：你自己调用官方那个 `reloadNuxtApp` 工具主动整页重载。
- **从 sessionStorage 恢复**：它是把状态临时存进浏览器当前标签页的 `sessionStorage`（关掉标签就清空），重载后再读回来。

打开它就是在 `nuxt.config.ts` 里：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    restoreState: true,
  },
})
```

注意这是 `experimental` 节里的开关。官方对整节实验特性有一句统一的话——"these features are experimental and could be removed or modified in the future"，也就是随时可能改名、改行为甚至删掉。别把它当稳定 API 依赖。

## 副作用：首屏可能闪一下

它为了不把水合搞崩，做了一个顺序上的让步。官方原文：

> To avoid hydration errors, it will be applied only after the Vue app has been mounted, meaning there may be a flicker on initial load.

这里要解释两个术语。**水合（hydration）**= 浏览器拿到服务器发来的 HTML 后，Vue 接手这段 HTML、挂上事件和数据，让它变成真正可交互的应用。**mounted**= Vue 应用挂载完成这个时点。

这句话的意思是：`restoreState` 存回来的状态，是在 Vue 已经挂载**之后**才应用的，而不是挂载前。为什么要这样？因为如果在挂载前塞进去，服务端渲染出来的 HTML 和客户端一开始拿到的状态对不上，就会报水合错误。代价就是——首次加载时你能看到内容"闪"一下：先是默认状态渲染出来，紧接着恢复的状态覆盖上去。

## 关键提醒：给 useState 显式写 key

这一节有一个 important，是这一篇最该记住的东西：

> Consider carefully before enabling this as it can cause unexpected behavior, and consider providing explicit keys to `useState` as auto-generated keys may not match across builds.

回到 `useState`（251 篇）：它可以不写 key，让编译器按"文件名 + 行号"自动生成。问题就出在"跨构建"。你这次部署生成的自动 key，和上次部署时存进 sessionStorage 的 key，可能对不上——因为文件一改、行号一变，自动 key 就变了。`restoreState` 拿着新 key 去找旧状态，找不到，恢复就失效。

所以如果你要开 `restoreState`，官方建议的配套做法是：给每个要恢复的 `useState` 都显式写一个稳定的字符串 key。

```ts
// 开 restoreState 时，别依赖自动 key
const filter = useState('search-filter', () => '')
const draft = useState('compose-draft', () => '')
```

## 别和水合搞混

这一节容易混的是"恢复时机"。把三个概念摆一起：

| 机制 | 状态跨什么存活 | 存哪儿 |
| --- | --- | --- |
| SSR 水合 | 服务端渲染 → 客户端接手（同一次加载） | HTML 里的 payload |
| `useCookie`（255） | 跨请求、跨刷新 | 浏览器 cookie |
| `restoreState` | 跨"整页重载"（chunk 错误或主动 reload） | sessionStorage |

`useState` 本身只保证"这一次页面加载内、服务端和客户端之间"共享（它的值进的是 HTML payload）。它**不**保证你刷新页面还在、也**不**保证关标签再开还在——那些要靠 `useCookie`。`restoreState` 补的恰好是中间那一档：不是普通刷新，而是"页面崩了/你主动重载"之后，从 sessionStorage 把状态捞回来。

## 自查

- 要不要开 `restoreState`，取决于你有没有"chunk 加载失败重载后希望用户不丢状态"的真实需求。没有就别开——官方明说它"can cause unexpected behavior"。
- 开了就必须给要恢复的 `useState` 都写显式字符串 key，否则跨构建对不上、恢复直接失效。
- 接受首屏可能"闪"一下这个既定副作用，这是它规避水合错误的代价，不是 bug。
- 记住它是实验开关，升级 Nuxt 版本时回看一眼有没有改名或改默认。

## 下一篇

下一篇 264 反过来讲：既然 `useState` 的值要进 HTML payload，那敏感信息就该谨慎——文档点名的三处"别把什么放进 payload / 客户端包"的注意。

## 本文依据

- Experimental Features（`guide/going-further/experimental-features`）中 `restoreState` 一节：
  - "Allows Nuxt app state to be restored from `sessionStorage` when reloading the page after a chunk error or manual `reloadNuxtApp()` call."
  - "To avoid hydration errors, it will be applied only after the Vue app has been mounted, meaning there may be a flicker on initial load."
  - important："Consider carefully before enabling this as it can cause unexpected behavior, and consider providing explicit keys to `useState` as auto-generated keys may not match across builds."
  - 页首统一 note："Note that these features are experimental and could be removed or modified in the future."
- useState（`api/composables/use-state`）：不传 key 时"then a key that is unique to the file and line number ... will be generated for you"，即自动 key 随构建变化的来源。
- State Management（`getting-started/state-management`）：`useState` 的值 "preserved after server-side rendering (during client-side hydration) and shared across all components"——即它作用在一次加载内。
