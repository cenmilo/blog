---
url: /posts/e17dc44edd87f2aa05b6445ddd44b4e7/
title: 286 - 插件最佳实践：能组合就别插件、异步就开 parallel
date: 2027-01-15T10:00:00+08:00
updated: 2027-01-15T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：官方专门有一页 Nuxt Plugins 讲插件的三条最佳实践。这一篇把它们逐条落地并接上前面铺过的机制：一、别把插件 setup 写重（插件跑在水合期，初始化慢会挡渲染）；二、能组合就别用插件，多数工具直接写成 composable 更轻更好维护；三、异步插件要开 parallel: true，让多个插件并发加载、互不阻塞。每条都对应到本系列已经讲过的具体出处，帮你判断"这个功能到底该不该做成插件"。

categories:
  - 前端开发

tags:
  - nuxt4
  - 插件
  - 最佳实践
  - composable
  - parallel
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

前面 270–285 一直在讲插件"怎么写"，官方另有一页专门讲"什么时候别这么写"。它开头一句定了调：

> Plugins in Nuxt allow you to extend your application with additional functionality. However, improper use can lead to performance bottlenecks.

也就是说插件能干很多事，但用错会拖性能。这页就三条，我逐条落地，并把它们和前面铺过的机制连起来。

## 一、别把插件 setup 写重

官方第一条 "Avoid Costly Plugin Setup"：

> A large number of plugins can cause performance issues, especially if they require expensive computations or take too long to initialize. Since plugins run during the hydration phase, inefficient setups can block rendering and degrade the user experience.

要理解这条的分量，关键是那句 **"plugins run during the hydration phase"**（270 也引过这句）。它说明插件不是按需跑的——它跟着一次页面加载一起跑、卡在水合这段时间里。于是两个直接后果：

- **数量**：插件越多，这段固定开销越大。
- **单个成本**：某个插件 setup 里做了重计算或慢初始化，就会**挡住渲染**，用户体感变差。

这条其实是整篇的根因，另外两条都是它的解法。落到判据上：**一个功能如果只是"给别处调用的工具/逻辑"，不该塞插件——塞进插件就把它从"按需"变成了"每次加载都跑"。**

## 二、能组合就别用插件

官方第二条 "Use Composition Whenever Possible"：

> Whenever possible, favor composition over plugins. Just like in Vue, many utilities and composables can be used directly without the need for a plugin. This keeps your project lightweight and improves maintainability.

这条正是本系列反复出现的官方建议的最凝练版。前面至少两处呼应过：

- 270（Providing Helpers 那句 important）："we highly recommend using composables instead of providing helpers to avoid polluting the global namespace and keep your main bundle entry small."
- 286 这条从"性能"角度再次强调，理由是 "keeps your project lightweight and improves maintainability"。

合起来看，官方对"小工具/共享逻辑"的一贯口径是：**优先放进 `app/composables/`（自动导入、按需使用）**。只有当某个能力确实要"在应用创建那一刻、作为应用级配置"去做——比如装 Vue 插件、注册全局指令（273）、往 NuxtApp 上挂全局 `$helper`（274）——才轮到插件。

一个实际的选择清单：

| 你想做的 | 该用 |
| --- | --- |
| 一个函数/一段可复用逻辑 | composable（自动导入） |
| 响应式共享状态 | `useState`（251） |
| 装第三方 Vue 插件、注册全局指令 | Nuxt 插件 + `nuxtApp.vueApp` |
| 应用级一次性副作用 | Nuxt 插件 |

## 三、异步插件要开 parallel

官方第三条 "If `async`, Enable `parallel`"：

> By default, all plugins loads synchronously.
> When defining asynchronous plugins, setting `parallel: true` allows multiple plugins to load concurrently, improving performance by preventing blocking operations.

机制细节在 272 已经讲过——默认顺序加载，"Nuxt won't wait until the end of the plugin's execution before loading the next plugin" 是 `parallel` 的效果。这页把它归为一条**最佳实践**：只要你的插件 `setup` 是异步的（里面有 `await`，比如 `await $fetch(...)`），就考虑给它 `parallel: true`，让多个插件并发跑、别互相堵。

```ts [plugins/my-plugin.ts]
export default defineNuxtPlugin({
  name: 'my-plugin',
  parallel: true,
  async setup (nuxtApp) {
    // 下一个插件不会等这里跑完
  },
})
```

但 272 也提醒过另一面，这里再钉一次：`parallel` 让下游不等你了——如果**有别的插件依赖你的结果**，就别让它并行，改成那个下游插件用 `dependsOn: ['my-plugin']` 明确"我等它"。两条键经常配合用。

## 把三条收成一句话

官方这一节的逻辑链是：插件跑在水合期（一）→ 所以少用、能用组合式就用组合式（二）→ 真要用的异步插件，让它并行别互相挡（三）。对新手最值钱的是第二条那句判据：**先问"这非得是插件吗"，多数时候答案是"不，写成 composable 就行"。**

## 自查

- 准备写的插件，是不是其实一个自动导入 composable 就够？官方第二条就是让你先做这个减法。
- 插件 setup 里有没有重计算或慢 `await`？它会挡水合（第一条），能不放就不放。
- 有异步 `setup` 的插件开了 `parallel` 吗？同时要确认没有别的插件在等它的结果（有的话那一个改用 `dependsOn`）。
- 插件数量做个盘点：每条"加载期"开销都是叠加的。

## 下一篇

下一篇 287 讲两类具体的插件用途——接入第三方 Vue 插件（`vueApp.use`）与全局指令（`directive`），以及另一类"服务端插件"（Nitro 插件）如何扩展 HTML 输出。

## 本文依据

- Nuxt Plugins（`guide/best-practices/plugins`）：定位句 "Plugins in Nuxt allow you to extend your application ... improper use can lead to performance bottlenecks."；三条小节原文——Avoid Costly Plugin Setup（"A large number of plugins can cause performance issues ... Since plugins run during the hydration phase, inefficient setups can block rendering"）、Use Composition Whenever Possible（"favor composition over plugins ... keeps your project lightweight and improves maintainability"）、If `async`, Enable `parallel`（"By default, all plugins loads synchronously. When defining asynchronous plugins, setting `parallel: true` allows multiple plugins to load concurrently"）。
- plugins 目录（`directory-structure/app/plugins`）：Providing Helpers 的 composable-over-provide important；Loading Strategy 的 `parallel: true` 与 `dependsOn` 示例。
