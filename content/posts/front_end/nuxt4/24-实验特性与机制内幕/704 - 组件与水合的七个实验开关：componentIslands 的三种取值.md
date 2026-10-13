---
url: /posts/3dbc9c2ea97341dba28d8b4b2ec083e8/
title: 704 - 组件与水合的七个实验开关：componentIslands 的三种取值
date: 2027-06-03T11:00:00+08:00
updated: 2027-06-03T11:00:00+08:00
author: cmdragon

summary:
  摘要：这一组里最容易配错的是 `componentIslands`：它不是布尔值就够，文档给的注释是 `true`、`false` 或 `'local+remote'` 三种，配套 `<NuxtIsland>` 组件与 `.island.vue` 文件。另外两条默认开的开关解决的是同一类"名字对不上"问题——组件名与自动导入名不一致（`normalizeComponentNames`：`components/SomeFolder/MyComponent.vue` 在 Vue 眼里叫 `MyComponent`，自动导入要写 `SomeFolderMyComponent`）、模板里的 `$route` 与 Nuxt 管理的 `useRoute()` 不同步（`templateRouteInjection` 靠注入 mixin 来同步）。

categories:
  - 前端开发

tags:
  - nuxt4
  - 实验特性
  - 组件岛屿
  - 水合策略
  - 模板
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## `componentIslands`：取值比布尔多一档

原文：启用实验性的组件岛屿支持，配套的是 `<NuxtIsland>` 组件与 `.island.vue` 文件。示例那行注释就是三种取值：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    componentIslands: true, // false or 'local+remote'
  },
})
```

文档没有在这一节解释 `'local+remote'` 与 `true` 的差别，只把细节指向服务端组件那一节。**这一节还有一条容易忽略的 note：在非 SFC 组件（例如 `<NuxtLink>`）上要跳过 `nuxt-client`**，选择性水合那条规则在概念页的 selective hydration with `nuxt-client` 小节里。

它与流式渲染的交互已经写在渲染那篇：岛屿槽内容本来靠渲染后一次缝合，流式下改成在文档末尾发一个惰性 `<template>`、用水合前运行的内联脚本挪位；**同时用 `features.noScripts` 与岛屿的应用会退回缓冲渲染器**。

服务端的路线图入口在这节给了 GitHub issue #19772。

## `lazyHydration` 与 `clientFallback`：一个默认开、一个没写默认值

`lazyHydration` 的原文是"为 `<Lazy>` 组件启用水合策略，通过把组件的水合推迟到真正需要时来提升性能"，并且明确写着**懒水合默认开启**，可以关。要理解它的边界得回到组件目录那一节的 delayed/lazy hydration 小节，这一节只给了入口。

`clientFallback` 则**整节没有声明默认值**，只说它启用 `<NuxtClientFallback>` 组件，用于"在 SSR 出错时在客户端渲染内容"。文档没有列出它适用哪类错误——从描述读，它是给 SSR 抛错准备的兜底渲染，不是性能开关。

## `normalizeComponentNames`：两个名字系统对不上时开它

这节讲得很具体，值得照着排查一次。默认情况下如果你没手动设置，Vue 会给组件起一个与文件名一致的名字。文档给的目录例子：

```bash [Directory structure]
├─ components/
├─── SomeFolder/
├───── MyComponent.vue
```

原文的结论分三句：就 Vue 而言这个组件名是 `MyComponent`；**如果你想对它用 `<KeepAlive>`，或在 Vue DevTools 里辨认它，用的就是这个名字**；**但要自动导入它，你得写 `SomeFolderMyComponent`**。`normalizeComponentNames` 的作用就是把这两个值对齐，让 Vue 生成的组件名符合 Nuxt 的组件命名规则。

所以这个开关影响的是"按名字操作组件"的所有场景（`<KeepAlive>` 的 include/exclude、DevTools 里的定位）。而它默认是开的——原文那句是"如果你遇到问题，可以关掉这个特性"。

## `templateRouteInjection`：模板里的 `$route` 是靠 mixin 同步的

原文先把默认行为说清：自动导入的 `useRoute()` 返回的路由对象**会与 `<NuxtPage>` 里当前看到的页面保持同步**，但这对 `vue-router` 导出的 `useRoute`、以及 Vue 模板里默认可用的 `$route` 对象**并不成立**。

开启这个选项后，**会注入一个 mixin，把 `$route` 模板对象与 Nuxt 管理的 `useRoute()` 保持同步**。默认开，可关。

实际含义：如果你在模板里读 `$route`，或在组件里混用 vue-router 的 `useRoute`，两者的同步保证来自这个开关；关掉之后模板里的 `$route` 就可能不再是那个被 Nuxt 管理的对象。文档没有给出关闭后的具体表现。

## `templateImportResolution`：模块加进去的模板，import 从哪里解析

它的定位原文：**关掉"从添加该模板的模块的路径去解析 Nuxt 模板里的导入"**。默认行为是 Nuxt 会尝试以"添加它的模块"为基准来解析模板里的 import；设成 `false` 就禁用这个行为，文档给的用途是——**在某些环境里遇到解析冲突时可能有用**。默认开，另有指向实现细节的 PR #31175。

这一条对模块作者是直接相关的：你的 `addTemplate` 生成的文件里写了相对导入，能不能解析到取决于这个开关，而基准路径是你模块自己的位置。它跟 `dependsOn`、`updateTemplates` 那套重生成逻辑无关，只管解析起点。

## `asyncEntry`：模块联邦那条路

一句话：为 Vue 打包产物启用**异步入口**的生成，用于帮助 module federation 支持。这一节**没有任何关于它影响首屏、chunk 结构或水合顺序的说明**，只给了 `asyncEntry: true` 的开启示例。

## 七个开关的默认状态

| 开关 | 默认 | 文档给的关闭或开启理由 |
| --- | --- | --- |
| `componentIslands` | 未声明 | 需要 `<NuxtIsland>` 与 `.island.vue` 就开；三种取值含 `'local+remote'` |
| `lazyHydration` | 开 | 关掉就不再延迟 `<Lazy>` 组件的水合 |
| `clientFallback` | 未声明 | 需要 SSR 出错时在客户端渲染内容 |
| `normalizeComponentNames` | 开 | 遇到名字相关问题可关 |
| `templateRouteInjection` | 开 | 关掉后模板里的 `$route` 不再由 mixin 同步 |
| `templateImportResolution` | 开 | 某些环境解析冲突时关掉 |
| `asyncEntry` | 未声明 | 需要 module federation 支持时开 |

这一组里四条的默认值是"开"，两条整节没写默认值，`componentIslands` 的三种取值里也只有一种是默认——**这三类信息在同一页上的写法完全不统一，读的时候必须逐节确认**，不能推断。

下一篇是构建与模块那一组十个：`buildCache` 的九个一动就全量重建的文件、`watcher` 的取值、`enforceModuleCompatibility` 什么时候该开、`viteEnvironmentApi` 与 `compatibilityVersion` 的联动。

## 下一篇

下一篇 705：构建与模块的十个实验开关：watcher 的四种取值与 buildCache 的九个文件

## 本文依据

- https://nuxt.com/docs/4.x/guide/going-further/experimental-features
