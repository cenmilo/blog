---
url: /posts/b3a8fd83de8fc6b2132587440b434649/
title: Teleport：Nuxt 里唯一有服务端支持的挂载点是哪一个
date: 2026-11-13T11:00:00+08:00
updated: 2026-11-13T11:00:00+08:00
author: cmdragon

summary:
  摘要：`<Teleport>` 的作用官方写成一句——"把一个组件传送到 DOM 中的另一个位置"，它是 Vue 的内置组件，Nuxt 这一页要处理的是**服务端渲染带来的限制**。官方用一条 warning 把两件事放在一起说清：`<Teleport>` 的 `to` 目标**期望一个 CSS 选择器字符串或一个真实的 DOM 节点**；而 **"Nuxt 目前只对传送到 `#teleports` 有 SSR 支持，其他目标则需要用 `<ClientOnly>` 包裹以获得客户端支持。"** 官方因此只给了两个写法：名为 Body Teleport 的例子把弹层写进 `<Teleport to="#teleports">`（内部是 `<div v-if="open" class="modal">`，由一个按钮把 `open` 置真、另一个置假）；名为 Client-side Teleport 的例子则是 `<ClientOnly>` 里套 `<Teleport to="#some-selector">`。这两个例子背后的差别值得记住：**同一个组件在模板树里的位置可以不变，但它在真实 DOM 里落到哪、什么时候落进去，是被这一条约束决定的**；而包 `<ClientOnly>` 不是无害的包装——按 096 引的官方 note，被包住的默认插槽内容会被 tree-shake 出服务端构建，所以那一块不会出现在初始 HTML 里。

categories:
  - 前端开发

tags:
  - nuxt4
  - teleport
  - 挂载点
  - 弹层
  - 水合

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


弹层要放到 DOM 顶层去，不然会被父容器的 `overflow`、`transform` 或层叠上下文困住。Vue 用 `<Teleport>` 解决这件事。Nuxt 只在这一页上加了一条硬约束，而这条约束决定了两种写法该在什么时候各用哪一个。

## 组件树的位置，不等于 DOM 里的位置

| 官方原话 | 要点 |
| --- | --- |
| "`<Teleport>` 组件把一个组件传送到 DOM 中的另一个位置。" | 传送的是**落点**，不是渲染时机 |
| （warning）"`<Teleport>` 的 `to` 目标期望一个 **CSS 选择器字符串或一个真实的 DOM 节点**。" | `to` 能写什么 |
| （warning）"**Nuxt 目前只对传送到 `#teleports` 有 SSR 支持，其他目标则通过使用 `<ClientOnly>` 包裹来获得客户端支持。**" | 服务端只有一个落点 |

最后那条就是这一篇的全部内容：**在 Nuxt 里，能被服务端渲染接住的传送目标只有 `#teleports` 这一个；想传到别的地方，就得放弃服务端那一次渲染。**

## 写法一：官方给的 Body Teleport

```vue
<template>
  <button @click="open = true">
    Open Modal
  </button>
  <Teleport to="#teleports">
    <div
      v-if="open"
      class="modal"
    >
      <p>Hello from the modal!</p>
      <button @click="open = false">
        Close
      </button>
    </div>
  </Teleport>
</template>
```

读这段要注意三件事：

- `<Teleport>` 写在组件模板里，位置由你决定；但里面的 `<div class="modal">` 在真实 DOM 里落在 `#teleports` 那个目标处，而不是紧跟在触发按钮后面。
- 是否显示用的是 `v-if="open"`——**传送的内容仍然受组件自己的状态控制**，`Teleport` 只改落点。
- `to="#teleports"` 是官方在这一页给的**唯一**带 SSR 支持的写法。这一节官方就把它命名为 "Body Teleport"。

## 写法二：官方给的 Client-side Teleport

```vue
<template>
  <ClientOnly>
    <Teleport to="#some-selector">
      <!-- content -->
    </Teleport>
  </ClientOnly>
</template>
```

结构上就是外面多套一层 `<ClientOnly>`，`to` 换成你自己的选择器（`#some-selector`）。

这一层的代价是明确的，来自 `<ClientOnly>` 那条 note：**被包的默认插槽内容会被 tree-shake 出服务端构建，其中组件使用的 CSS 也可能不会在渲染初始 HTML 时被内联。** 也就是说：

| | `to="#teleports"` | `<ClientOnly>` + 任意 `to` |
| --- | --- | --- |
| 服务端 HTML 里有没有这块 | 有（官方：只有它具备 SSR 支持） | **没有** |
| 首屏就能看到内容 | 可以 | 不行，要等客户端挂载 |
| 落点选择 | 只能 `#teleports` | 任意 CSS 选择器或 DOM 节点 |
| 额外副作用 | 无 | 该块 CSS 可能不进初始 HTML |

**所以顺序是反的：先试 `#teleports`，只有在必须传到别处时才付 `<ClientOnly>` 的代价**，而不是默认就包一层。

## `#some-selector` 这个目标什么时候存在

这是包了 `<ClientOnly>` 之后最容易出错的一点。服务端那一次渲染里 `<ClientOnly>` 内的内容根本不存在，于是这个分支的传送发生在浏览器侧——而 `to` 要的是一个**已经存在的** CSS 选择器或 DOM 节点。

官方在这一页没有替你保证目标存在，它只给了那句约束（其他目标要靠 `<ClientOnly>` 才有客户端支持）。落到自己代码里，可操作的要求就一句话：**那个挂载点必须在你自己的模板里渲染出来**（例如页面结构里确实有 `id="some-selector"` 的那个元素），否则传送的目标是空的。

判断方式很直接，两种都能定位问题：

1. 看首屏 HTML——如果这块内容不在，说明走的是 `<ClientOnly>` 分支，此时去源码里确认目标元素是否也在首屏 HTML 中存在。
2. 水合之后看元素实际落点——`<ClientOnly>` 里的组件只在挂载后才渲染，官方给的做法是 watch 一个模板 ref 来确认元素出现（`{ once: true }`）。

## 三个常见误判

| 现场 | 真实原因 |
| --- | --- |
| 弹层在 dev 下正常，`nuxt build` 后位置错或首屏没有 | 用了 `<ClientOnly>` 分支：这块不进服务端构建，初始 HTML 里没有它 |
| 传给 `to` 的选择器写得很随意，内容不见了 | `to` 需要一个可解析的目标（CSS 选择器字符串或 DOM 节点），且该目标要在传送发生时存在 |
| 想在服务端就把弹层挂到某个自定义 id | 官方目前只给了 `#teleports` 一个有 SSR 支持的目标；其他目标要走 `<ClientOnly>` |

## 决策表

| 你要放的东西 | 该用哪种 |
| --- | --- |
| 弹层、抽屉、遮罩这类需要脱离父容器的浮层 | `<Teleport to="#teleports">`（官方示例正是 modal） |
| 必须挂到某个自定义 DOM 节点（第三方容器、页面预留槽位） | `<ClientOnly>` 包 `<Teleport to="#your-id">`，并接受"首屏 HTML 里没有这块" |
| 内容需要在首屏出现、且能被索引 | 别用 `<ClientOnly>` 分支——按 096 那条 note，它不进服务端构建 |
| 只是想调整样式层叠，父级没有困住它 | 不必动用传送，先解决 CSS 层叠上下文 |

## 自查

1. `<Teleport>` 的 `to` 官方说能接受哪两类值？
2. Nuxt 目前对哪个目标有 SSR 支持？其他目标要靠什么？
3. 官方的 Body Teleport 示例里，控制显示用的是什么指令？这说明了传送与状态的什么关系？
4. 包 `<ClientOnly>` 之后，服务端 HTML 和初始 HTML 的 CSS 各会发生什么？
5. 什么情况下你会明确放弃 `<ClientOnly>` 分支？

## 下一篇

099 讲 `<NuxtTime>`：一个必须服务端渲染安全的组件是怎么设计的——`datetime` 必填、`locale` 用 BCP 47、`Intl.DateTimeFormat` 的选项直接摊平成 prop，以及 `style` 为什么是保留 prop、必须写 `relativeStyle`。

## 本文依据

- `https://nuxt.com/docs/4.x/api/components/teleports`
  - 原文（页面描述）："The \<Teleport> component teleports a component to a different location in the DOM."
  - 原文（warning）："The `to` target of `<Teleport>` expects a CSS selector string or an actual DOM node. Nuxt currently has SSR support for teleports to `#teleports` only, with client-side support for other targets using a `<ClientOnly>` wrapper."
  - 原文（Body Teleport 示例）：`<button @click="open = true">Open Modal</button>`、`<Teleport to="#teleports">`、`<div v-if="open" class="modal">`、`<p>Hello from the modal!</p>`、`<button @click="open = false">Close</button>`
  - 原文（Client-side Teleport 示例）：`<ClientOnly><Teleport to="#some-selector"><!-- content --></Teleport></ClientOnly>`
  - 原文 link-example 指向 `https://nuxt.com/docs/examples/advanced/teleport`
  - 说明：本页 `to` 的链接指向 Vue 文档 `<Teleport>` 一节，即这一组件本身来自 Vue，Nuxt 这一页处理的是 SSR 支持范围
- `https://nuxt.com/docs/4.x/api/components/client-only`
  - 对照用（见 096）："The `<ClientOnly>` component is used for purposely rendering a component only on client side."
  - 原文（note）："The content of the default slot will be tree-shaken out of the server build. (This does mean that any CSS used by components within it may not be inlined when rendering the initial HTML.)"
  - 原文（Accessing HTML Elements）："Components inside `<ClientOnly>` are rendered only after being mounted. To access the rendered elements in the DOM, you can watch a template ref"；示例含 `useTemplateRef('nuxtWelcomeRef')` 与 `{ once: true }`
