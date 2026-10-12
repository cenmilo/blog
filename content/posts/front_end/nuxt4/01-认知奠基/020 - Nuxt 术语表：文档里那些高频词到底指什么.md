---
url: /posts/e5fcd9d314b60e4b17f7c47bc2325b75/
title: Nuxt 术语表：文档里那些高频词到底指什么
date: 2026-10-18T11:00:00+08:00
updated: 2026-10-18T11:00:00+08:00
author: cmdragon

summary:
  摘要：这一篇把文档里反复出现、但很少被集中解释的词按官方的说法对齐：渲染（浏览器与服务端都能把 Vue 组件变成 HTML 这一步）、同构渲染（同一份代码两侧都跑）、水合（服务端跑过的同一份 JS 在浏览器再跑一次以绑定监听器）、预渲染与静态托管（nuxt generate 或 nuxt build --prerender 把每页渲染成单独静态 HTML 文件）、混合渲染（按路由定义缓存规则并决定服务端如何响应新请求）、边缘侧渲染（更接近用户的 CDN 边缘服务器生成 HTML，官方说它更像一个部署目标而非渲染模式）、服务端引擎 Nitro（产出独立于 node_modules 的独立服务端 dist，构建进 .output）、自动导入、约定与配置的分界、模块（在 dev 与 build 时顺序运行的异步函数）、图层（一个子目录就是一层、可共享代码组件组合式函数与配置且被自动注册）、中间件（运行在 Vue 部分，与运行在 Nitro 部分的服务端中间件完全不同）、载荷 payload（服务端把取数结果与 useState 序列化后交给客户端水合复用的那份数据）、实验特性（官方允许其中出现破坏性变更的唯一区域）。

categories:
  - 前端开发

tags:
  - nuxt4
  - 术语
  - 概念
  - 渲染
  - 速查

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


这一篇不引入新知识，是一份**用官方原话写成的术语对照**。建议做法是：读后面的篇目时遇到词不确定的，回这一页查一遍，查到你能用自己的话说出来为止。

## 渲染类

| 词 | 官方的说法 |
| --- | --- |
| **渲染** | "浏览器和服务端都能解释 JavaScript 代码，把 Vue.js 组件变成 HTML 元素。这一步称为**渲染**。" |
| **同构渲染（universal）** | "Nuxt 支持**同构**与**客户端**两种渲染方式"；服务端跑一遍得到 HTML，浏览器里"同一份 JavaScript 代码……再运行一次"以启用交互，因此称为同构 |
| **水合（Hydration）** | "此时它负责启用交互性（因此称为同构渲染），把监听器绑定到 HTML 上。这一步称为**水合**。水合完成后，页面才能享受动态界面与页面过渡这类好处。" |
| **SSR** | "Nuxt 内置服务端渲染（SSR）能力，所以你不必自己配置一个服务器"；默认即开启 |
| **预渲染 / 静态托管** | "如果你用 `nuxt generate` 或 `nuxt build --prerender` 命令把应用部署到静态托管，那么默认情况下，Nuxt 会把每个页面渲染成单独的静态 HTML 文件。" |
| **混合渲染（hybrid）** | "混合渲染允许**按路由**使用不同的缓存规则，并决定服务器对新请求如何响应。" |
| **边缘侧渲染（ESR）** | "让渲染通过 CDN 的边缘服务器、更接近你的用户来进行"；并且"ESR 更像一个**部署目标**，而不是一个真正的渲染模式" |

**最容易混的一对**：预渲染 vs 混合渲染。前者是"整站构建期变 HTML"，后者是"按 URL 决定这一条怎么响应"。

## 架构与产物类

| 词 | 官方的说法 |
| --- | --- |
| **Nitro / 服务端引擎** | "Nuxt 由一个新的服务端引擎驱动：Nitro"；"Nitro 生成一个独立于 `node_modules` 的独立服务端 dist"；构建时输出到 `.output` |
| **`.output`** | "Nuxt 在为生产构建你的应用时创建 `.output/` 目录"；用它来部署；不应触碰里面的文件 |
| **`.nuxt`** | 自动生成的类型与配置所在处，"运行 dev 服务器或构建应用时生成"，也可用 `nuxt prepare` 手动生成 |
| **h3** | "服务端 API 端点与中间件由 Nitro 添加，而它内部使用 h3" |
| **`$fetch`** | "如果运行在浏览器里，它会向服务器发起一次 API 调用；如果运行在服务端，它会直接调用相关的函数，从而省掉一次额外的 API 调用" |
| **载荷（payload）** | "当 Nuxt 在服务端渲染一个页面时，它会把你的取数结果（`useAsyncData` 与 `useFetch`）与应用状态（`useState`）序列化进一个 payload，好让客户端能在不重新取数的情况下水合" |
| **岛屿 / 服务端组件** | "服务端组件（也被称为岛屿组件）……它的 HTML 被嵌入页面，而它的任何 JavaScript 都不会发给客户端" |
| **模块** | "Nuxt 模块是异步函数，它们在通过 `nuxt dev` 启动开发或通过 `nuxt build` 构建生产项目时**顺序运行**" |
| **图层（layer）** | "`layers/` 目录允许你组织并共享可复用的代码、组件、组合式函数与配置。这个目录内的层会被自动注册进你的项目。" |

**载荷那个词最值得先记**：**它解释了为什么水合不会重复请求，也解释了为什么 `Set`、`Map`、函数、组件实例进不了它**——后面做性能与类型时都会撞上这条。

## 约定与代码组织类

| 词 | 官方的说法 |
| --- | --- |
| **约定 / 目录结构** | "Nuxt 使用约定和有主见的目录结构来自动化重复性工作……配置文件仍然可以定制并覆盖它的默认行为。" |
| **自动导入** | "在各自目录里写 Vue 组合式函数与组件，然后不使用 import 就能用它们，同时享受 tree-shaking 与优化后的 JS 包。" |
| **中间件（应用侧）** | "路由中间件运行在你 Nuxt 应用的 **Vue 部分**。尽管名字相似，它们和运行在你应用 Nitro 服务端部分的服务端中间件完全不同。" |
| **服务端中间件** | "`server/middleware/` 下的中间件对每个请求执行"；"处理器不应返回任何东西（也不应关闭或响应请求），只能检查或扩展请求上下文，或者抛出一个错误" |
| **布局** | "`layouts/`：包裹你的页面、并在页面之间**避免重新渲染**的 Vue 组件" |
| **组合式函数** | "`composables/`：添加你的 Vue 组合式函数" |
| **`app.vue`** | "根组件"；`app/app.vue` 是应用根组件的位置 |
| **错误页** | "`error.vue`：你的 Nuxt 应用的错误页"（它不是路由页，这一区分后面专讲） |
| **别名 `~` / `~~` / `#imports` / `#build`** | 生成的 tsconfig 里包含"像 `#imports`、`~/file` 或 `#build/file` 这样的路径别名" |
| **水合不匹配** | 官方指向 Vue 文档的 hydration mismatch 一节，并提醒"保持服务端与客户端数据一致很重要" |

## 版本与实验类

| 词 | 官方的说法 |
| --- | --- |
| **semver 节奏** | "遵循 semver……每年一次大版本，预期每周一左右的补丁与每月左右的次要版本" |
| **实验特性（experimental）** | "除了明确标记为 `experimental` 的选项之内，它们不应包含破坏性变更"——即**唯一允许破坏性变更的地方** |
| **`future.compatibilityVersion`** | 官方在版本那页的说法是"Nuxt 4 将是一个以稳定性为目标的发布，包含所有 `compatibilityVersion: 4` 的特性"；配置侧也用它来提前采用未来默认行为 |
| **nightly** | "由仓库最新提交直接构建"；且"仅在 nightly 频道可用的特性会在文档中用 alert 标出" |
| **EOL / Maintenance** | 状态表定义："Unsupported：不再维护、不会收到安全补丁"；"Maintenance：只会收到安全补丁" |

## 三组"词长得很像但不是同一物"

这个系列后面最容易混的，我先在这里点出来，等你读到对应篇目时会有印象：

1. **模块 vs 插件**：模块是构建期顺序执行的异步函数；插件是应用创建时运行的东西（`app/plugins/`）。
2. **路由中间件 vs 服务端中间件**：前者在 Vue 侧、不会为 `/api/*` 运行；后者在 Nitro 侧、对每个请求运行且不应返回值。
3. **`runtimeConfig` vs `app.config.ts`**：前者可被环境变量覆盖、私有键只在服务端；后者是构建期可确定的公开变量、**不能**被环境变量覆盖。

## 自查

- [ ] 随机挑五个词，你能不看表用一句自己的话说清吗？
- [ ] "预渲染"和"混合渲染"的差别，你说得出关键那两个字（整站 / 按路由）吗？
- [ ] 载荷（payload）里为什么放不下函数和 `Map`？
- [ ] 三组同名词的区分，你各有几个字的判据？
- [ ] `error.vue` 为什么"不是路由页"（这个词先记住，后面专讲）？

## 下一篇

021 - 项目从哪来：create nuxt 的模板、参数与非交互模式

## 本文依据

- `guide/concepts/rendering`（渲染的定义、同构与水合、SSR 好处与代价、客户端渲染、`nuxt generate`/`nuxt build --prerender` 逐页静态与无服务端接口的警告、混合渲染与 ESR 的定位）
- `guide/concepts/server-engine`（Nitro 六条能力、h3 与 API 层、`$fetch` 的两侧走法、类型化路由、独立于 node_modules 的 dist 与 `.output`、原生存储层）
- `guide/concepts/nuxt-lifecycle`（服务端七步与浏览器五步、水合过程不含 Server Components、app:rendered 与 render:html、重定向与状态重置）
- `getting-started/data-fetching`（payload 的定义：取数结果与 useState 被序列化以便水合复用）
- `getting-started/introduction`（七条自动化与约定原文；"同时提供前端与后端功能"；五个核心包）
- `guide/concepts/modules`、`directory-structure`、`directory-structure/app/layouts`、`directory-structure/app/middleware`、`directory-structure/server`、`directory-structure/app/error`（模块、图层、布局、两类中间件、错误页的官方一行定义）
- `guide/concepts/auto-imports`、`guide/concepts/typescript`、`getting-started/configuration`、`guide/concepts/server-components`、`directory-structure/output`、`directory-structure/nuxt`（自动导入、别名与生成物、runtimeConfig 与 app.config、岛屿、.output 规则、.nuxt 生成物）
- `community/roadmap`、`guide/going-further/nightly-release-channel`（发布节奏与实验特性例外、版本状态定义、nightly 定义与 alert 标记）
