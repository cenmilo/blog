---
url: /posts/064a9158d5baf7e66a6af2044643833b/
title: 175 - DevOnly 与生产瘦身：内容不进包、替代要测与五个常见病因
date: 2026-12-09T10:00:00+08:00
updated: 2026-12-09T10:00:00+08:00
author: cmdragon

summary:
  摘要：`<DevOnly>` 与 `<ClientOnly>` 都叫"只在某个环境渲染"，但官方给的两句范围说明**说的是两件不同的事**：`<DevOnly>` 是 "Nuxt provides the `<DevOnly>` component to render a component only during development." + "**The content will not be included in production builds.**"；`<ClientOnly>` 是 "The content of the default slot will be **tree-shaken out of the server build**."（096、165）——一个是**不进生产包**，一个是**不进服务端构建**。官方那条示例里还藏着一个提醒，两处都写了同一句："**if you ever require to have a replacement during production / be sure to test these using `nuxt preview`**"，因为 `#fallback` 的定义正是"if you ever require to have a replacement during production"。至于"生产包里为什么这么大"，性能指南给的是五条常见问题，其中第一条的机制句最值得抄："**A large number of plugins can cause performance issues, especially if they require expensive computations or take too long to initialize. Since plugins run during the hydration phase, inefficient setups can block rendering and degrade the user experience.**"，解法是"Inspect your plugins and see if some of them could be implemented rather as a composable or utility function instead."；第二条针对"未使用的依赖"："Inspect your `package.json` for unused dependencies and analyze your code for unused utils/composables/functions."。**验证手段官方也列了四件**：`nuxt analyze`（"leverages `vite-bundle-visualizer` … to generate a visual representation of your application's bundle"）、DevTools 的 Timeline/Assets/Render Tree/Inspect 四项、Chrome DevTools 的 `Performance`（直接给出本地 LCP 与 CLS，交互后还能看 INP）与 `Lighthouse`、PageSpeed Insights 与 Web Page Test。**最省的一条组合官方写成一句**："For content and marketing sites that need little or no client-side interactivity, you can go further and **combine prerendering, the `noScripts` route rule, server components and lazy hydration to ship near-zero JavaScript**."

categories:
  - 前端开发

tags:
  - nuxt4
  - DevOnly
  - 性能
  - 生产构建
  - 打包分析

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


调试面板我习惯用 `<ClientOnly>` 包，因为"它不该在服务端跑"。但这俩的官方说明差得很远：一个让内容**不进生产包**，另一个只是**不进服务端构建**。这一篇把这两个范围分清，再顺一遍官方给的生产瘦身清单。

## 两句范围说明对照

| 组件 | 官方范围句 | 生产包里有没有 |
| --- | --- | --- |
| `<ClientOnly>` | "The content of the default slot will be **tree-shaken out of the server build**." | **有**（客户端才渲染，所以代码必须在包里） |
| `<DevOnly>` | "The content **will not be included in production builds**." | **没有** |

`<DevOnly>` 的定位句：

> Nuxt provides the `<DevOnly>` component to render a component only during development.

**这决定了选择标准是"这段东西生产环境到底存不存在"**：调试面板、埋点提示、只在本地有用的实验开关——用 `<DevOnly>`，包能真变小；浏览器 API、需要水合后才有意义的交互——用 `<ClientOnly>`，包不会变小。

## 官方示例里的三个细节

```vue [app/pages/example.vue]
<template>
  <div>
    <Sidebar />
    <DevOnly>
      <!-- this component will only be rendered during development -->
      <LazyDebugBar />

      <!-- if you ever require to have a replacement during production -->
      <!-- be sure to test these using `nuxt preview` -->
      <template #fallback>
        <div><!-- empty div for flex.justify-between --></div>
      </template>
    </DevOnly>
  </div>
</template>
```

三个细节都在示例自己身上：

| 细节 | 说明 |
| --- | --- |
| 组件名写成 `LazyDebugBar` | 官方在 `<DevOnly>` 里用了 `Lazy` 前缀（161）——开发期也少下载一块 |
| 插槽定义 | "`#fallback`: if you ever require to have a replacement during production." |
| 那两行注释被写了两遍 | "**be sure to test these using `nuxt preview`**" 在示例和 Slots 一节各出现一次 |

**最后一条我照做了**：`#fallback` 里的东西是**生产里真正被渲染的那一份**，而 `nuxt dev` 下你看不到它——只有 `nuxt build` 之后用 `nuxt preview` 才会跑到。官方把这句话重复写两遍，说明这类"生产替代内容在开发期看不见"的坑很常见。

那个空 `<div>` 的注释（"empty div for flex.justify-between"）也在提示同一件事：生产布局里可能仍需一个占位节点来撑住 flex 结构。

## `<DevOnly>` 与 `<ClientOnly>` 能叠加吗

官方没有讨论两者嵌套的组合语义。我按两句范围句自己排了一下影响，并把结论标成待验证：

| 组合 | 按范围句应发生什么 |
| --- | --- |
| `<DevOnly>` 里放 `<ClientOnly>` | 生产时外层整体不进包，内层那句"摇出服务端构建"已无意义 |
| `<ClientOnly>` 里放 `<DevOnly>` | 开发期：内容进服务端构建被摇掉、且只在 dev 存在；生产期：外层进客户端包、内层不进包 → **客户端包里只剩 fallback** |

这两格都是我从两句原文推的，不是文档给的组合规则。真要用嵌套时我只做一件事：**跑一次 `nuxt build && nuxt preview`，在生产产物里确认那个位置渲染的是谁**（官方那条"test these using `nuxt preview`"正好适用于这种验证）。

## 官方列的"包里多了东西"五类原因

性能指南的 "Common Problems" 一节，五条 Problem/Solution 都值得照抄自查：

| 问题 | 官方描述 | 官方解法 |
| --- | --- | --- |
| **插件用太多** | "A large number of plugins can cause performance issues, especially if they require expensive computations or take too long to initialize. **Since plugins run during the hydration phase, inefficient setups can block rendering and degrade the user experience.**" | "Inspect your plugins and see if some of them could be implemented rather as a **composable or utility function** instead." |
| **未使用的代码/依赖** | "With the development of the project, there can be a case where there will be some unused code or a dependency. This additional functionality may not be used or needed while it will be increase the bundle size of our project." | "Inspect your `package.json` for unused dependencies and analyze your code for unused utils/composables/functions." |
| 没用 Vue 的性能建议 | "[Vue documentation] lists several Performance improvements we can use in our Nuxt projects as well but as they are part of Vue documentation, developers tend to forget about it" | "Use concepts such as `shallowRef`, `v-memo`, `v-once`, etc" |
| 不遵循约定 | "Developers have a tendency to introduce new concepts they've seen in another project which can cause conflicts and problems with performance." | 在项目里建立规则与模式 |
| 一次加载所有东西 | "When a page is loaded and it is not correctly instructed about the order of loading elements it will result in fetching everything at the same time - which can be slow and result in bad User Experience." | "Use concepts such as **Progressive Enhancement** where core webpage content is set first, then more nuanced and technically rigorous layers of presentation and features are added on top" |

**第一条那句机制是这一节里最有信息量的一行**：插件跑在**水合阶段**——这与 149 那条"SSR 不跑 `onMounted`"、以及 061 的插件顺序规则合起来看，就能解释"为什么一个只做初始化的插件能拖慢首屏交互"。它也解释了官方为什么在 061 那类页面反复推荐用 composable 代替插件。

## 怎么确认：官方给的四件工具

| 工具 | 官方说明关键词 |
| --- | --- |
| `nuxt analyze` | "allows you to **analyze the production bundle**… leverages `vite-bundle-visualizer`（similar to `webpack-bundle-analyzer`）to generate a visual representation of your application's bundle" |
| Nuxt DevTools | 四项："**Timeline**"、"**Assets**"、"**Render Tree**"、"**Inspect** – Lists all files used in the Vue app with their size and evaluation time" |
| Chrome DevTools | `Performance` 面板"instantly shows your local **Largest Contentful Paint (LCP)** and **Cumulative Layout Shift (CLS)** scores"，交互后还捕获 **INP**；`Lighthouse` 审 performance/accessibility/SEO/best practices |
| PageSpeed Insights / Web Page Test | "reports on the user experience of a page on both mobile and desktop devices" |

关于 analyze 那张图，官方给的读法直接对应动作：

> When you see a large block in the visualization, it often signals an opportunity for optimization—whether by **splitting it into smaller parts, implementing lazy loading, or replacing it with a more efficient alternative**, especially for third-party libraries.
>
> Large blocks containing multiple elements can often be reduced by **importing only the necessary components rather than entire modules** while large standalone blocks may be better suited for lazy loading rather than being included in the main bundle.

**这两句正好把本篇前后接上**：`importing only the necessary components rather than entire modules` 对应的就是 `#components`/`#imports` 显式导入那一族写法（154、161），`lazy loading` 对应 `Lazy` 前缀（161）。

## 最省的那一条：官方给的组合拳

> For content and marketing sites that need little or no client-side interactivity, you can go further and **combine prerendering, the `noScripts` route rule, server components and lazy hydration to ship near-zero JavaScript**.

这一句把前面四篇串成一条路线：

| 手段 | 本篇之外的细读 |
| --- | --- |
| prerendering | 141（`prerender` 规则）、037-038 那条命令线 |
| `noScripts` | 141（不下发脚本、页面永不水合、CSS 不受影响） |
| server components | 169、171、174 |
| lazy hydration | 166、167 |

而 `<DevOnly>` 在这条路线上的位置很具体：**它是这几种手段里唯一"直接少写代码进包"的那个**——其它几种改的是加载或执行时机。

## 自查

1. `<DevOnly>` 与 `<ClientOnly>` 的两句范围句分别怎么说？"生产包里有没有"这一维上答案各是什么？
2. 该用哪个的判断标准是什么？给两个例子。
3. `#fallback` 的官方定义句是什么？为什么官方两处都要求用 `nuxt preview` 测？
4. 官方示例里那个组件名为什么带 `Lazy` 前缀？
5. 插件为什么会拖慢首屏交互？官方的机制句是哪个短语？推荐的替代是什么？
6. "未使用依赖"这一条官方让你查哪两处？
7. `nuxt analyze` 那块大图，官方给的三种处置分别是什么？其中哪一种对应显式导入？
8. "near-zero JavaScript" 那条组合拳由哪四个手段构成？各在哪一篇细讲？

## 下一篇

176 处理一个我改了三次决定的问题：组件到底要不要全局注册。官方的态度句是"though not recommended"，本篇把三种全局入口、手动导入的两种出口和"已经全局了怎么退回来"排成一条路径。
