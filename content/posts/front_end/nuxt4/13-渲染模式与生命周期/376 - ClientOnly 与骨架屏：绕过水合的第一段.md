---
url: /posts/ba019a87c44bf59e9f05d5538a6c69f8/
title: ClientOnly 与骨架屏：绕过水合的第一段
date: 2027-02-14T10:00:00+08:00
updated: 2027-02-14T10:00:00+08:00
author: cmdragon

summary:
  摘要：374 有一类 mismatch 干脆没法人工对齐——内容就是只能在浏览器里出。Nuxt 的第一招就是把那块包进 `<ClientOnly>`：它**故意只在客户端渲染**，服务端那趟直接跳过，于是根本不存在"两端比一比"。服务端占位由 `#fallback` 插槽或 `fallback`/`placeholder` 属性给（这就是骨架屏）。两条要注意的官方提醒：默认插槽内容会被从服务端构建里 tree-shake 掉（里面组件用到的 CSS 可能不会被内联进首屏 HTML）；包在里面的组件挂载后才存在，想拿它的 DOM 得 watch 一个模板 ref。这一篇讲清 ClientOnly 的定位、骨架屏怎么写、以及它和水合的关系。

categories:
  - 前端开发

tags:
  - nuxt4
  - ClientOnly
  - 骨架屏
  - 水合
  - fallback
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

374 教的都是"想办法让两端算出一样"。但有一类内容做不到——它天生只能在浏览器里出（依赖浏览器 API 的地图、图表，或就是想在首屏之后才加载的模块）。对这种，与其硬凑一致，不如让服务端**别渲染它**。Nuxt 的第一招就是 `<ClientOnly>`。

## 它只跳过一件事：服务端渲染

官方定义：

> The `<ClientOnly>` component is used for purposely rendering a component only on client side.

关键词是 **purposely（有意）**。默认插槽里的内容不会在服务端渲染，所以也就**没有两端 HTML 可比**——373 那种 mismatch 从根上被绕开：既然服务端不出这块，水合时它只在客户端出现一次，谈不上"对不上"。

这就是它和后面两招的分界：
- `<ClientOnly>`（376，本篇）：**总是**不在服务端渲染，服务端只留占位。
- `<NuxtClientFallback>`（377）：**照常尝试**服务端渲染，只有孩子抛错时才退回客户端。
- 延迟水合（378）：服务端照常出 HTML，只是把**水合**推迟到某个时机。

## 一条要记住的提醒：CSS 可能不内联

官方 `::note`：

> The content of the default slot will be tree-shaken out of the server build. (This does mean that any CSS used by components within it may not be inlined when rendering the initial HTML.)

默认插槽内容会从服务端构建里被 tree-shake 掉。附带后果：里面那些组件用到的 CSS，**渲染首屏 HTML 时可能不会被内联**。也就是说首屏那一刻，这块本来就没有、样式也可能缺——反正它是等客户端才补上的，这是用它的代价，别指望它参与首屏。

## 骨架屏：`#fallback` 插槽与两个属性

服务端不出内容，那首屏这段时间显示什么？由"占位"决定。官方给了两种写法。

**属性**（props）：

- `placeholderTag` | `fallbackTag`：指定服务端渲染时用哪个标签包裹。
- `placeholder` | `fallback`：指定服务端渲染的内容。

```vue
<template>
  <ClientOnly
    fallback-tag="span"
    fallback="Loading comments..."
  >
    <Comment />
  </ClientOnly>
</template>
```

**插槽**（更常用，能放任意结构）：

- `#fallback`：指定一段"在服务端渲染、直到 `<ClientOnly>` 在浏览器挂载前都显示"的内容——这就是骨架屏本体。

```vue
<template>
  <ClientOnly fallback-tag="span">
    <Comments />
    <template #fallback>
      <!-- 这段在服务端渲染，水合前显示 -->
      <p>Loading comments...</p>
    </template>
  </ClientOnly>
</template>
```

`#fallback` 和 `fallback` 属性都能用；结构化的骨架（灰块、占位卡）用插槽，纯一句文案用 `fallback` 属性即可。

## 想拿里面的 DOM：watch 一个模板 ref

因为包在里面的组件"挂载之后才渲染"，你在 setup 顶层拿不到它。官方给的做法是 watch 模板 ref，等它出现：

```vue
<script setup lang="ts">
const nuxtWelcomeRef = useTemplateRef('nuxtWelcomeRef')

// 组件可用时这个 watch 会触发
watch(nuxtWelcomeRef, () => {
  console.log('<NuxtWelcome /> mounted')
}, { once: true })
</script>

<template>
  <ClientOnly>
    <NuxtWelcome ref="nuxtWelcomeRef" />
  </ClientOnly>
</template>
```

`{ once: true }` 是因为它只在第一次出现时有意义。

## 自查

- `<ClientOnly>` 故意只在客户端渲染，服务端不碰默认插槽 → 没有两端比对，从根上避开 mismatch。
- 默认插槽会被 tree-shake 出服务端构建，里面组件的 CSS 首屏可能不内联。
- 服务端占位＝骨架屏：`#fallback` 插槽或 `fallback`/`placeholder`（+ `fallbackTag`/`placeholderTag` 包标签）。
- 和 377 NuxtClientFallback（出错才退回）、378 延迟水合（照常 SSR 只推迟水合）是三种不同"绕法"。
- 里面的组件挂载后才存在，取它要 watch 模板 ref（`{ once: true }`）。

## 下一篇

下一篇 377 讲 `<NuxtClientFallback>`：它不像 ClientOnly 那样无条件跳过，而是"先正常 SSR，孩子报错才退回客户端"，还带 `@ssr-error` 事件和 raw-HTML 的 XSS 警告。

## 本文依据

- `<ClientOnly>` 定义（purposely only client side）、默认插槽 tree-shake + CSS 可能不内联的 `::note`、props `placeholderTag`/`fallbackTag` 与 `placeholder`/`fallback`、`#fallback` 插槽（服务端渲染、挂载前显示）、两段示例、以及"内部组件挂载后才渲染、要 watch 模板 ref"的 Example（`api/components/client-only`）。
