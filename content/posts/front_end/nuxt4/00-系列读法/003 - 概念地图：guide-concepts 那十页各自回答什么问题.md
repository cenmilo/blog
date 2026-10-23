---
url: /posts/3b35bb7a5f77e06782204f26dee03bde/
title: 003 - 概念地图：guide/concepts 那十页各自回答什么问题
date: 2026-10-13T09:00:00+08:00
updated: 2026-10-13T09:00:00+08:00
author: cmdragon

summary:
  摘要：官方文档的 guide 分成六块：Key Concepts（讲自动导入、混合渲染、TypeScript 支持背后的原理）、Best Practices（开发时的最佳实践）、Working with AI（MCP Server 与 LLMs.txt）、Module Author Guide（怎么写模块）、Recipes（常见问题的解法）、Going Further（实验特性、钩子等进阶概念）。其中 Key Concepts 一共十页：vuejs-development、auto-imports、modules、rendering、server-engine、nuxt-lifecycle、server-components、typescript、esm、code-style。这一讲给每页一句话的问题定义，让你之后读到任何一页时都知道它在你学习路线的哪一站。

categories:
  - 前端开发

tags:
  - nuxt4
  - 文档
  - 概念
  - 学习路线
  - 入门

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

官方在介绍页留了一句建议，我们建议把每个概念都读一遍，以便对 Nuxt 的能力与各包的范围有一个完整的视野。

问题是这些概念页分布在十个不同主题里，读到第三页就容易迷路。这一讲给你一张地图，每一页回答什么问题、什么时候该读它。

guide 一共六块，官方在首页给出的分区是这样的。Key Concepts，官方描述是发现 Nuxt 背后的主要概念，从自动导入、混合渲染到它的 TypeScript 支持，你学原理的时候需要它，也就是本教程九成的内容。Best Practices，了解在用 Nuxt 开发时的最佳实践，项目要上线、要做性能与无障碍时回来查。Working with AI，通过 MCP Server 与 LLMs.txt 把 AI 工具整合进你的 Nuxt 工作流，想让 AI 读你项目版本的文档时用。Module Author Guide，学习如何创建 Nuxt 模块来集成、增强或扩展任何 Nuxt 应用，要写自己的模块时读。Recipes，找到常见问题的解法，并学习如何在你的 Nuxt 项目里实现它们，有具体需求时查。Going Further，掌握 Nuxt 的进阶概念，例如实验特性、钩子及更多内容，要改框架行为、写钩子、开实验开关时读。

看这六块你会发现一件事，入门阶段真正该反复读的只有第一块。其余五块都是遇到需求再回来查，硬背反而低效。

Key Concepts 那十页，按和你当前关系远近排一下，每页一个核心问题。

`vuejs-development`，Nuxt 里的 Vue 和纯 Vue 有什么不同。它回答我用 Vue 的那套习惯在 Nuxt 里能不能照搬，还包含 Vapor Mode 这一块内容。

`auto-imports`，为什么可以不写 import。除了讲目录来源，那一节最重要的其实是另一件事，组合式函数对调用位置和是否同步的硬性要求。你在 130 讲会回头重读它。

`modules`，扩展 Nuxt 的正道是什么。一句话，模块是构建期依次执行的异步函数。写模块本身在 Module Author Guide。

`rendering`，我的页面由谁渲染成 HTML。这是全套教程里最该读透的一节，universal、client-side、hybrid、edge-side 四种方式的取舍全在这里。下一篇就专门讲这一节。

`server-engine`，Nitro 是什么、`.output` 里有什么。后端能力从哪来的答案，服务端目录、类型化接口都挂在它下面。

`nuxt-lifecycle`，一次请求按什么顺序执行。它给你的不是 API，而是顺序。调试时最值钱的就是它，知道某段代码排在第几站，才知道它为什么没跑到、或者为什么跑了两次。

`server-components`，服务端组件与岛屿。某个组件的 JS 不想发给浏览器这件事，它是进阶话题，本套教程放在组件那一段。

`typescript`，类型从哪来、类型检查要不要开。新手最容易在这里踩空，类型是自动生成的，但检查默认不开。

`esm`，为什么引一个库就报语法错误。纯排障向，报错时读最划算。

`code-style`，官方推荐的 lint 怎么装。很短，但决定了你项目的规则来源，006 讲会带你看它。

还有个很多人不知道的事，概念页讲为什么，目录页讲放哪里。官方在目录那页也是这么组织的，把 `app/` 下每个子目录用一行说明职责，`components/` 是应用的 Vue 组件，`composables/` 添加你的 Vue 组合式函数，`layouts/` 包裹你的页面、并在页面之间避免重新渲染的 Vue 组件，`middleware/` 在导航到某个特定路由之前运行代码，`pages/` 是基于文件的路由，`plugins/` 在 Nuxt 应用创建时使用 Vue 插件及更多内容，`utils/` 添加可以在组件、组合式函数和页面里使用的函数。

同一件事在两处出现，读法应该是，先看目录页知道东西放哪，再进概念页知道为什么这么放。本套教程会按这个配对来讲。

最后分享一条使用文档的习惯。官方文档里每个概念那一节都会指向其它节，例如讲渲染的那一节末尾就把 SPA 加载模板、部署、边缘渲染各指了一次。跟着链接跳是好事，但请每跳一次就记一句我是为了什么跳过去的，否则十分钟你会停在第 5 层链接里，忘了原本要找什么。这是学框架最浪费时间的习惯。

## 自查

- [ ] 六大块里，你现在知道入门阶段只需要反复读哪一块吗？
- [ ] 十个概念页，你能说出「渲染方式」和「生命周期」这两页分别回答什么问题吗？
- [ ] 你能说清概念页与目录页的分工（为什么 / 放哪里）吗？
- [ ] 你上一次读文档跳失目标是在第几层链接？

## 下一篇

004 - 渲染方式先认全名字：同构渲染、客户端渲染、混合渲染与边缘渲染
