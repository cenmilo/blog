---
url: /posts/2954f46ced949e2ebebabad8c600049b/
title: 106 - 服务端组件与岛屿：为什么有些组件不该把 JavaScript 带进浏览器
date: 2026-11-16T10:00:00+08:00
updated: 2026-11-16T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇解释**为什么需要岛屿**，答案在官方的开头两段里。默认行为的浪费点是这句："Nuxt 默认在服务器上渲染你的应用，但随后它会把**每一个组件**的 JavaScript 发给浏览器并水合整页。"对于内容密集、在客户端永不变化的组件（官方举的是 markdown 渲染、语法高亮、CMS 输出），官方判定这是**白费工作**："用户下载、解析并执行代码，而这些代码唯一的工作就是复现页面上已经存在的 HTML。"服务端组件（**官方同时写道 "also called island components"，即也叫岛屿组件**）把这件事反过来："它在服务器上渲染，它的 HTML 被嵌入页面，而它的 JavaScript 一点都不会发给客户端。它的依赖（一个 markdown 解析器、一个高亮库）同样留在服务器上。"标记方式有三种：给组件加 `.server` 后缀（例如 `app/components/HighlightedMarkdown.server.vue`，之后像普通组件一样用）；放在 `~/components/islands/` 里的组件也会注册为岛屿，可直接用 `<NuxtIsland name="MyIsland" />` 渲染（对应 `~/components/islands/MyIsland.vue`）；而**如果给一个 `.server.vue` 配一个同名的 `.client.vue`，此时它不是岛屿——客户端那一半照常水合**。有一条硬约束：官方 warning 写着"服务端组件（以及岛屿）必须有单个根元素。（HTML 注释也被视为元素。）"开关是 `experimental.componentIslands`，默认值 `'auto'`——"一旦你的应用包含一个服务端组件或岛屿就自动启用该特性，所以大多数情况下你不需要任何配置"，只有要远程岛屿或选择性客户端水合时才显式设 `selectiveClient` 与 `remoteIsland`；官方同时声明"服务端组件仍被标记为实验特性"。渲染机制是理解一切限制的钥匙：**渲染一个岛屿会向一个专用的 island 端点发请求，该端点会在服务器上创建一个全新的、隔离的 Vue 应用来只渲染那个组件、创建一个可通过 `nuxtApp.ssrContext.islandContext` 访问的 island context、并再跑一遍你的插件（除非对象式插件设了 `env: { islands: false }`）**。因为隔离，三条后果直接成立：页面与岛屿之间不能共享状态（provide/inject、Pinia、`useState` 都过不去，要传数据请用 props）；岛屿里的 `useRoute()` 反映的是**岛屿自己的请求**而不是用户所在页面；**渲染岛屿时路由中间件不会运行**。最后一个代价必须算进决策：首屏服务端渲染时岛屿是内联渲染的、没有额外请求，但在**客户端导航**时目标页上的每个岛屿都要向服务器取回，官方因此给出适用判据——岛屿最适合靠整页加载到达的页面（内容页与营销页）或每页岛屿数量很少的情形，**如果一个组件需要在客户端频繁更新，岛屿大概是错误的工具**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 岛屿
  - 服务端组件
  - 水合
  - 包体积

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


一段 markdown、一次语法高亮、一块 CMS 输出——它们在浏览器里永远不会变，但默认情况下它们的代码照样被打包发过去、还要跑一遍来"水合"。这一篇讲 Nuxt 怎么把这类组件从包里剔出去。

## 默认行为的浪费在哪

官方的诊断段值得整段读：

> Nuxt 默认在服务器上渲染你的应用，但随后它会把**每一个组件**的 JavaScript 发给浏览器并水合整页。对于内容密集的组件（markdown 渲染、语法高亮、CMS 输出）这种在客户端从不改变的内容，这是**白费工作**：用户下载、解析并执行代码，而这些代码唯一的工作就是复现页面上已经存在的 HTML。

服务端组件把这件事倒过来：

> 它在服务器上渲染，它的 HTML 被嵌入页面，而它的 JavaScript 一点都不会发给客户端。它的依赖（一个 markdown 解析器、一个高亮库）同样留在服务器上。

**关键在"依赖也留在服务器上"**：省掉的不是组件自己那几行，而是它拖进来的整棵依赖树。

## 三种写法，第三种不是岛屿

### ① `.server` 后缀

官方说明："给一个组件加上 `.server` 后缀，就把它变成一个独立的服务端组件。"

```bash [Directory Structure]
-| app/
---| components/
-----| HighlightedMarkdown.server.vue
```

用的时候和普通组件没有区别：

```vue [app/pages/example.vue]
<template>
  <div>
    <!--
      rendered on the server; the markdown parsing and highlighting
      libraries are not included in your client bundle
     -->
    <HighlightedMarkdown markdown="# Headline" />
  </div>
</template>
```

那段官方注释就是这个特性的全部卖点：**markdown 解析与高亮库不在你的客户端包里。**

### ② `islands/` 目录 + `<NuxtIsland>`

官方说明：`~/components/islands/` 里的组件**同样被注册为岛屿**，可以直接用 `<NuxtIsland>` 渲染，例如 `~/components/islands/MyIsland.vue` 对应 `<NuxtIsland name="MyIsland" />`。

这一种的入口是显式的——你写的是组件名而不是标签名，因此也就拿到了 `<NuxtIsland>` 上那些控制项（`lazy`、`context`、`#fallback`，下一篇展开）。

### ③ 同名 `.client.vue` 配对：这时它不是岛屿

官方原文：**"你也可以把一个 `.server.vue` 组件与一个同名的 `.client.vue` 组件配对，以获得独立的服务端与客户端实现。这种情况下该组件并不是一个岛屿：客户端那一半会正常水合。"**

这条特别容易误会成"岛屿的完整版"。它其实是另一件事——**两套实现分别用于两侧**，JS 照发、水合照跑。详见组件目录那一篇。

## 一条硬约束：单根元素

官方 warning 一句话："**服务端组件（以及岛屿）必须有单个根元素。（HTML 注释也被视为元素。）**"

注意括号里那半句——这和页面"必须有单个根元素以便路由过渡"是同一条口径（注释也算一个元素）。多一个注释都会让它变成 fragment。

## 开关：多数情况下你不用配

官方说明：组件岛屿由 `experimental.componentIslands` 控制。**默认值是 `'auto'`，一旦你的应用里出现一个服务端组件或岛屿就自动启用该特性，所以大多数情况下你不需要任何配置。**

只有需要以下两样时才显式设：

```ts
export default defineNuxtConfig({
  experimental: {
    componentIslands: {
      selectiveClient: true, // or 'deep', to enable `nuxt-client`
      remoteIsland: false, // allow rendering islands from a remote source
    },
  },
})
```

- `selectiveClient: true`（或 `'deep'`）才启用 `nuxt-client`——即岛屿里某个组件可以单独在客户端水合。
- `remoteIsland: false` 是默认值，即默认**不允许**渲染来自远程源的岛屿。

而官方在同一节挂了一条状态声明："**服务端组件仍被标记为实验特性。你可以在 GitHub 上跟进路线图。**"

## 为什么限制这么多：渲染机制

官方的小节标题就是"How Islands Are Rendered"，机制是三件事：

> 服务端组件在底层使用 `<NuxtIsland>`。渲染一个岛屿会向一个**专用的 island 端点**发出请求，该端点会：
> - 在服务器上创建一个**全新的、隔离的 Vue 应用**，只用来渲染那一个组件
> - 创建一个 island context，你在岛屿内部可以通过 `nuxtApp.ssrContext.islandContext` 访问它
> - **再跑一遍你的插件**，除非它们（用对象式插件）设置了 `env: { islands: false }`

"再跑一遍插件"这条解释了为什么插件里不能放"每个请求只能执行一次"的副作用；而"隔离的 Vue 应用"解释了三条后果：

| 官方结论 | 影响 |
| --- | --- |
| "你无法在页面与岛屿之间共享状态（provide/inject、Pinia、`useState`）；请通过 props 传数据。" | 状态进不去，只能走 props |
| "岛屿里的 `useRoute()` 反映的是岛屿自己的请求，而不是用户当前所在页面。如果岛屿需要路由信息，请显式传入——作为 props，或通过 `<NuxtIsland>` 的 `context` prop（在岛屿内从 `nuxtApp.ssrContext.islandContext` 读取）。" | 它不知道你在哪一页 |
| "渲染岛屿时，路由中间件不会运行。" | 想保护岛屿内容，得靠服务端那道（路由级中间件不覆盖它） |

还有一条数据格式的连带结论：**props 会被序列化并作为 GET 查询参数发送**。这让岛屿响应可缓存，但官方同时列出三个代价：props 必须 JSON 可序列化；受 URL 长度限制所以别传大数据；**props 可能出现在服务器访问日志、CDN 缓存和 `Referer` 头里**。也就是说——不要往岛屿 props 里放敏感内容。（这一节还有几条安全细则，放在下一篇和组件 API 一起讲。）

另外，**改动一个岛屿的 props 会触发一次网络请求，在服务器上重新渲染该组件并就地更新它的 HTML。**

## 客户端导航才是岛屿的账单

官方"The Client Navigation Round Trip"一节给了判据，这部分最影响选型：

- 在服务端渲染的首屏加载上，**岛屿是内联渲染的，没有额外请求**。
- 但在**客户端导航**时，目标页上的每个岛屿都必须从服务器取回（你能在 network 面板里看到这些请求）。这有真实成本：
  - 岛屿会在导航期间阻塞在一个网络往返上，**除非你传 `lazy` prop（配 `#fallback` 插槽）来非阻塞地渲染它们**
  - 一个每页有很多岛屿的应用，**每次导航就会发很多请求**

官方的结论写法很适合当选型标准："**岛屿最适合靠整页加载到达的页面（内容页和营销页），或者每页岛屿数量很少的情形。如果一个组件需要在客户端频繁更新，岛屿大概是错误的工具。**"

一个缓解手段官方也给了：当 `<NuxtLink>` 指向一个**服务端-only 页面**并被预取时，它的岛屿响应也会被一起预取，于是这个往返发生在用户导航**之前**而不是导航之中；预取的响应以岛屿的 name、props 与 context 为键缓存，也正是它挂载时用的同一把键；**预取失败并不致命，岛屿会在挂载时再取一次**。官方补了一条范围说明：这一条适用于渲染整个 `.server.vue` 页面的那个岛屿，页面内使用的岛屿仍然会在挂载时被取。

## 和另外三种"少发 JS"的分界

| 手段 | 切掉的粒度 | 官方定位 |
| --- | --- | --- |
| 服务端组件 / 岛屿 | **单个组件**及其依赖的客户端 JS | "它的 JavaScript 一点都不会发给客户端" |
| `ssr: false` | 反过来：整站**不在服务端渲染** | "你可以在 nuxt.config.ts 里启用仅客户端渲染" |
| `<ClientOnly>` | 单个组件块**不进服务端构建** | "组件也可以被强制只在客户端运行" |
| `noScripts` 路由规则 | 整条路由上的脚本 | 官方在岛屿一节里专门提醒了这个交互（见下） |

岛屿和 `<ClientOnly>` 方向正好相反，一个"只留服务端"、一个"只留浏览器"，但两者共同构成了官方那句组件双侧分工清单的两个极端。

`noScripts` 这一格有个官方点名的交互，值得提前知道：

> 岛屿的插槽与 `nuxt-client` 组件依赖于一段小的内联脚本，在水合之前把 teleport 的内容搬到正确位置。在用 `noScripts` 渲染的路由上，那段脚本被省略，因此**完整可交互的 `nuxt-client` 组件在这些路由上不会水合**。纯静态岛屿不受影响。

## 自查

1. 官方说默认行为浪费在哪里？它举的三个内容密集组件例子是什么？
2. `.server.vue` 与 `islands/` 目录两种写法，入口有什么不同？同名 `.client.vue` 配对时它还是岛屿吗？
3. `experimental.componentIslands` 的默认值是什么？为什么大多数项目不需要配置它？
4. 渲染一个岛屿时服务器上做了哪三件事？由此推出的三条"过不去"分别是什么？
5. 为什么官方说 props 里可能有敏感问题？首屏与客户端导航的请求差别是什么？
6. "需要在客户端频繁更新的组件"官方建议什么？

## 下一篇

107 讲 `<NuxtIsland>` 这一层的完整 API：`name`/`props`/`source`/`context`/`lazy` 各自含义、`refresh()` 与 `@error` 为什么只能直连组件才能用，以及 `source` 为什么被官方定性为"inherently dangerous"。
