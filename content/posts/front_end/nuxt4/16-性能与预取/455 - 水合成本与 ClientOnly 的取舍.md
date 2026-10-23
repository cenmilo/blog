---
url: /posts/e77b3f9ed62e01dcb3fa363d12a7e0dc/
title: 455 - 水合成本与 ClientOnly 的取舍
date: 2027-03-12T11:00:00+08:00
updated: 2027-03-12T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：水合不匹配不是可以忽略的警告。官方给的后果是两条硬的：Vue 会被迫**重渲染整个组件树**，从而抬高应用变为可交互所需的时间；用户看到的是内容闪烁与意外的布局位移。`ClientOnly` 是官方在小结里点名的一种解法（"Wrap client-only code: Use ClientOnly component for browser-specific content"），但它有自己的代价——默认插槽的内容会被**从服务端构建里 tree-shaken 出去**，官方在同一句括号里补了"这确实意味着其中组件用到的 CSS 可能不会内联进首屏那份 HTML"。占位怎么写、什么时候该改用 `NuxtTime` 或延迟水合，这一篇按原文分开列。

categories:
  - 前端开发

tags:
  - nuxt4
  - 水合
  - ClientOnly
  - 性能
  - 首屏
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

控制台一条水合不匹配警告，来源是页面上"当前时间"那一小块。你按搜到的说法把它包进 `<ClientOnly>`——警告干净了，但首屏那块位置先空一下、再被内容顶开。

这就是水合这件事上的取舍：**省掉一次不匹配，可能换来一次位移**。这一篇把官方写明的成本与写法列清。

## 先说成本：不匹配会贵在哪

水合这一页开头一句话就是态度：

> When developing, you may face hydration issues. Don't ignore those warnings.

接着"为什么必须修"分成性能与功能两块。性能那块只有两条，但都量化到机制：

> - **Increased time to interactive**: Hydration errors force Vue to re-render the entire component tree, which will increase the time for your Nuxt app to become interactive
> - **Poor user experience**: Users may see content flashing or unexpected layout shifts

第一条是这一篇的依据所在：**不匹配会迫使 Vue 重渲染整个组件树**，可交互时间因此被抬高。第二条把"闪烁"和"布局位移"写成用户可见后果——注意这里出现的是 **layout shifts**，与 453 讲的 CLS 是同一件事，只是官方是在水合语境下写的。

功能那块的三条：事件监听器可能挂不上导致按钮与表单失灵、应用状态与用户所见不一致、以及"Search engines may index different content than what users actually see"。

## ClientOnly 的原文范围

组件那一页给的定义：

> The `<ClientOnly>` component is used for purposely rendering a component only on client side.

**purposely**（有意）这个词是官方的用词，它把用法限定成"你主动决定这块只在浏览器渲染"，不是用来消警告的通用手段。

紧跟着一条 `::note`，就是开头那个空一下的原因：

> The content of the default slot will be tree-shaken out of the server build. (This does mean that any CSS used by components within it may not be inlined when rendering the initial HTML.)

两个后果在同一句里：**默认插槽的内容被从服务端构建里 tree-shake 出去**；括号里补一句"这确实意味着里面组件用到的 CSS 可能不会内联进首屏 HTML"。

属性与插槽是这样给的：

| 写法 | 官方说明 |
| --- | --- |
| `placeholderTag` 或 `fallbackTag` | 指定服务端渲染出来的标签 |
| `placeholder` 或 `fallback` | 指定服务端渲染出来的内容 |
| `#fallback` 插槽 | 指定服务端渲染、并**在 `<ClientOnly>` 于浏览器挂载之前一直显示**的内容 |

官方给的两段示例分别是属性式与插槽式：

```vue [app/pages/example.vue]
<template>
  <div>
    <Sidebar />
    <!-- The <Comment> component will only be rendered on client-side -->
    <ClientOnly
      fallback-tag="span"
      fallback="Loading comments..."
    >
      <Comment />
    </ClientOnly>
  </div>
</template>
```

```vue [app/pages/example.vue]
<template>
  <div>
    <Sidebar />
    <!-- This renders the "span" element on the server side -->
    <ClientOnly fallback-tag="span">
      <!-- this component will only be rendered on client side -->
      <Comments />
      <template #fallback>
        <!-- this will be rendered on server side -->
        <p>Loading comments...</p>
      </template>
    </ClientOnly>
  </div>
</template>
```

**属性只能给纯文本或标签名，插槽能给出真实结构的占位**——从两段示例的差别看，要避免那块位置塌陷，得用 `#fallback` 而不是 `fallback="Loading comments..."`。

还有一条容易漏的实现细节，在示例之后的小节里：`<ClientOnly>` 里的组件是**挂载之后才渲染**的，所以要拿到它内部的 DOM 元素，官方给的是 `useTemplateRef` 加 `watch(..., { once: true })`，而不是直接访问 ref。

## 官方给的另外三条替代路

时间类内容这一处，水合页的"解法"句其实给了两个选项：

> **Solution**: Use `NuxtTime` component or handle it client-side

一边是 `<NuxtTime :datetime="new Date()" format="HH:mm" />`，一边才是包 `ClientOnly` 加 `#fallback` 的那段。**`NuxtTime` 是排在前面那个**——它是服务端渲染安全的做法，不会引入占位与位移。这一条对应 099 那篇。

浏览器专用库那一节给的是另一种写法：在 `onMounted` 里动态导入——

```ts
onMounted(async () => {
  const { default: SomeBrowserLibrary } = await import('browser-only-lib')
  SomeBrowserLibrary.init()
})
```

这条不需要 `ClientOnly`，因为它本来就只在浏览器里跑，而且**代码只在挂载后才取**（这同时是体积上的好处，回到 439）。

想留下的那块仍然走服务端渲染、只是推迟变可交互，那是延迟水合的路子（378 讲过 `hydrate-on-visible` 这些策略）。它和水合不匹配是两件不同的事：延迟水合改的是**什么时候挂事件**，`ClientOnly` 改的是**服务端渲不渲这段**。

## 占位样式还有一处开关

`experimental.clientNodePlaceholder` 这一节把 `.client.vue` 组件、以及 `createClientOnly()` 包出来的客户端组件与服务端占位的实现说了清楚：开启后两者都用注释节点（`<!--placeholder-->`）代替 `<div>` 作为客户端组件的服务端占位，修的是"占位 `<div>` 与组件根元素同名时 scoped 样式可能应用不上"这个 Vue 水合问题。

但它自己带一条 `::warning`，正好是性能与外观的取舍：

> Enabling this means attributes (`class`, `style`, etc.) passed to `.client.vue` components will not appear in the SSR HTML. If you need styled placeholders to prevent layout shift, use `<ClientOnly>` with a `#fallback` slot instead.

**要防位移，官方直接建议改用 `<ClientOnly>` 加 `#fallback`**——这一句是这一篇最明确的推荐。

## 一张取舍表

| 你的情况 | 官方给出的做法 | 代价 |
| --- | --- | --- |
| 内容是时间驱动（问候语、时刻） | 用 `NuxtTime` | 无占位问题 |
| 逻辑只能在浏览器跑（本地库） | `onMounted` 里动态导入 | 首次进入才有内容 |
| 整块只在浏览器渲染，且要保证位置不塌 | `ClientOnly` + `#fallback` 插槽 | 默认插槽被 tree-shake 出服务端构建，里面 CSS 可能不内联 |
| 想消掉 `.client.vue` 的 scoped 样式问题 | `clientNodePlaceholder` | class/style 不进 SSR HTML，可能反而造成位移 |
| 内容服务端渲好，只是暂时不用可交互 | 延迟水合策略（`hydrate-*`） | 仍需 JS 到位才能交互 |

## 自查

- 水合不匹配的官方成本：**Vue 被迫重渲染整个组件树**、可交互时间上升，用户看到闪烁与布局位移。
- `ClientOnly` 是给"有意只在客户端渲染"的，不是消警告的工具；默认插槽会被 tree-shake 出服务端构建，其中 CSS 可能进不了首屏。
- 要占位不塌，用 `#fallback` 插槽而不是 `fallback` 文本属性；官方的防位移建议也是这一条。
- 时间内容首选 `NuxtTime`；浏览器专用库用 `onMounted` 动态导入；推迟交互用 `hydrate-*`。
- 里面要拿 DOM 元素时，用 `useTemplateRef` + `watch(..., { once: true })`。

## 下一篇

下一篇 456 讲这一批里最容易互相打架的地方：性能动作与可访问性各自的官方要求在哪里冲突——`hydrate-never` 官方自己写明别用在需要交互的组件上，`<div @click>` 省掉了预取也一并省掉了焦点与中键打开，`custom` 插槽要你把预取自己接回去。
