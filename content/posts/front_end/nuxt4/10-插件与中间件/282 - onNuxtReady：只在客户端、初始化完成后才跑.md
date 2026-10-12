---
url: /posts/4bb422cbaa5aec3023f6b0cd7267dd0b/
title: onNuxtReady：只在客户端、初始化完成后才跑
date: 2027-01-14T09:00:00+08:00
updated: 2027-01-14T09:00:00+08:00
author: cmdragon

summary:
  摘要：有些活不该挡住首屏——大的分析库、额外的埋点、可以在页面出来之后再做的初始化。onNuxtReady 就是把这些推迟到"应用完成初始化之后"再跑的回调，而且它只在客户端执行。官方示例是在一个 .client 插件里 onNuxtReady(async () => { await import('my-big-analytics-library') })。这一篇讲清它的两个限定（只客户端、不阻塞首屏）、"即使应用已经初始化完也安全——会排到下一个空闲回调"那句、以及它和水合期/ onMounted 的关系（什么时候用哪个）。

categories:
  - 前端开发

tags:
  - nuxt4
  - onNuxtReady
  - 客户端
  - 懒加载
  - 插件
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

首屏要快，靠的是"先让用户看见内容、别被次要逻辑拖住"。有一类活天生就该往后放：大的分析/统计库、和首屏无关的初始化。`onNuxtReady` 是 Nuxt 给这类活的专用挂载点。

## 它是什么、只在哪跑

官方定位：

> The onNuxtReady composable allows running a callback after your app has finished initializing.

紧接着两条限定，官方放在同一个 important 里：

> `onNuxtReady` only runs on the client-side.
> It is ideal for running code that should not block the initial rendering of your app.

两条要一起记：

1. **只在客户端跑**。服务端渲染那一趟里它不执行（"only runs on the client-side"）。所以你往它里塞的东西，天然是浏览器侧的。
2. **用途是不该挡首屏的代码**。它排在"应用完成初始化之后"，正好把重活从关键渲染路径上挪开。

## 官方示例：推迟加载一个大的库

典型用法——在插件里等应用就绪后，再动态 `import` 一个体积大的分析库：

```ts [app/plugins/ready.client.ts]
export default defineNuxtPlugin(() => {
  onNuxtReady(async () => {
    const myAnalyticsLibrary = await import('my-big-analytics-library')
    // 用 myAnalyticsLibrary 做点什么
  })
})
```

两个细节都在官方示例里，别忽略：

- 文件名是 `ready.client.ts`——带 `.client` 后缀（270）。虽然 `onNuxtReady` 本身只在客户端跑，但把整个插件也写成 `.client` 更干净：省得服务端那一趟还去解析这个用不上的插件。
- 里层是 `await import(...)` 动态导入——"大库首屏不加载、就绪后按需拉"。这是 `onNuxtReady` 和懒加载配合的标准姿势。

## 一个安全保证：初始化完成后再调也不会丢

官方还补了一句，处理"你调用它的时候应用其实早就初始化完了"这种情况：

> It is 'safe' to run even after your app has initialized. In this case, then the code will be registered to run in the next idle callback.

意思是：即使回调注册的时刻应用已经初始化完毕，`onNuxtReady` 不会静默吞掉它，而是把它安排到"下一个空闲回调（next idle callback）"里执行。所以你可以放心地在较晚的位置调用 `onNuxtReady`，不用担心"错过那个 ready 时机就永远不跑了"。

## 和 onMounted、isHydrating 怎么分

这一章 278 刚讲过 `isHydrating`，加上 Vue 自带的 `onMounted`，三个都沾"客户端、之后"。分工其实不同，摆一起：

| 你想做的事 | 用什么 |
| --- | --- |
| 一个组件挂载完，对这个组件的 DOM/状态做点事 | `onMounted`（组件级、随该组件） |
| 判断"现在还在不在水合期"来分流（比如区分错误） | `nuxtApp.isHydrating`（278） |
| 全局、不挡首屏的重活（动态加载大库、埋点初始化） | `onNuxtReady` |

关键区别在**作用域和时机语义**：

- `onMounted` 是"这个组件挂好了"，绑定在组件实例上。而且 278 那页给第三方库的官方正解就是放 `onMounted`（"Initialise libraries after hydration has completed"）——水合完成之后。
- `onNuxtReady` 是"整个应用初始化完成"，不绑组件，官方强调它 ideal 的对象是 "code that should not block the initial rendering"。它更偏全局、且专门服务于"别拖首屏"。

对纯第三方库初始化，官方 hydration 那页给的是 `onMounted`；`onNuxtReady` 这条示例给的是"动态 import 一个大分析库"。两者都算"往后放"，选谁看作用域：组件内挂载后用 `onMounted`，全局且要避开首屏关键路径用 `onNuxtReady`。

## 自查

- `onNuxtReady` 只在客户端跑，专门放"不该挡首屏"的活（动态 import 大库、埋点）；别拿它做需要 SSR 输出的事。
- 配套插件建议写 `.client.ts`，和服务端那一趟彻底无关。
- 它"错过 ready 也安全"——晚注册会排到下一个空闲回调，不会静默不执行。
- 别和 `onMounted`（组件级、水合后）混：全局、非阻塞首屏用 `onNuxtReady`；针对某组件挂载后的逻辑用 `onMounted`。

## 下一篇

P10 插件这一段先到这里。下一篇 283 讲 `useRuntimeHook`：在组件（不只是插件）里订阅运行时钩子的组合式。

## 本文依据

- onNuxtReady（`api/utils/on-nuxt-ready`）：定位句 "The onNuxtReady composable allows running a callback after your app has finished initializing."；important "`onNuxtReady` only runs on the client-side." 与 "It is ideal for running code that should not block the initial rendering of your app."；`ready.client.ts` 示例（`onNuxtReady(async () => { const myAnalyticsLibrary = await import('my-big-analytics-library') })`）；"It is 'safe' to run even after your app has initialized. In this case, then the code will be registered to run in the next idle callback."
- Nuxt and Hydration（`guide/best-practices/hydration`）：第三方库的官方对策 "Initialise libraries after hydration has completed"（`onMounted`）——用于对比作用域。
- useNuxtApp（`api/composables/use-nuxt-app`）：`isHydrating` 一节（278 已展开）——用于区分"判断水合期"与"就绪后再跑"。
