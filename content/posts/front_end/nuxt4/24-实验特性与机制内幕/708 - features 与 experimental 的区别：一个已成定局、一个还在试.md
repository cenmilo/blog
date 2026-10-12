---
url: /posts/f1c46ae625a8c79c5d0eeadc6281eda6/
title: features 与 experimental 的区别：一个已成定局、一个还在试
date: 2027-06-05T09:00:00+08:00
updated: 2027-06-05T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：`nuxt.config` 里有三节都在开关能力，但含义不同。`features` 只有三项（`devLogs`、`inlineStyles`、`noScripts`），它管的是**已经完成、可以选择不用**的能力；`experimental` 有 57 项，官方对它的声明是"未来可能被移除或修改"；`future` 三项（`compatibilityVersion`、`multiApp`、`typescriptBundlerResolution`）的位置介于两者之间——**提前采用那些将来会变成默认的行为**。这三节最容易混用的一点是：`future.compatibilityVersion: 5` 会**反过来改写 `experimental` 里一批开关的默认值**。

categories:
  - 前端开发

tags:
  - nuxt4
  - features
  - 实验特性
  - future
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 三节的定位，各取官方那句原话

| 节 | 条目数 | 官方定位 |
| --- | --- | --- |
| `features` | 3 | Nuxt 的某些特性是 opt-in 的，或者可以按你的需要关掉 |
| `experimental` | 57 | 这些特性是实验性的，**未来可能被移除或修改** |
| `future` | 3 | 一个用于**提前选择加入那些将在未来（可能是一个大版本）成为默认**的新特性的命名空间 |

也就是说：`features` 关的是"已经给你的东西"，`experimental` 开的是"还没定的东西"，`future` 是把"以后会变默认的东西"提前拉到现在。

## `features` 三项

**`devLogs`**：开发时把服务端日志流到客户端，这些日志可以在 `dev:ssr-logs` 钩子里处理。默认值是"在开发模式下启用（**当测试模式未激活时**）"；设为 `'silent'` 时日志**不打印到浏览器控制台**。

**`inlineStyles`**：渲染 HTML 时内联样式，**目前只在使用 Vite 时可用**。除了布尔值，还能传一个函数——接收 Vue 组件的路径、返回是否内联它的样式。文档给了默认函数：`(id) => id.includes('.vue')`。这个默认值正好解释了 `experimental.ssrStreaming` 那节里"开了 `inlineStyles` 就把路由样式内联成 `<style>`"的行为。

**`noScripts`**：关掉 Nuxt 脚本与 JavaScript 资源提示的渲染，让页面以纯 HTML 加 CSS 交付。三种取值：

- `false`（默认）：照常渲染脚本。
- `'production'`（或 `true`，**会被归一化成 `'production'`**）：只在生产构建里省略脚本，所以 `nuxt dev` 里的热更新等开发特性继续工作。
- `'all'`：开发与生产都省略。

除了 app 级设置，还能按路由用 `noScripts` 路由规则关掉，**而路由规则在所有模式下都生效**。

## `noScripts` 打开以后，HTML 里少了什么

文档列了四类被省略的内容：Nuxt 入口的 `<script>` 标签、入口 chunk 的 import map、内联的 payload `<script>`（以及启用 payload 提取时的 payload 预加载链接）、JavaScript 资源提示（JS chunk 的 `preload` 与 `prefetch` 链接）。

同时明确了两件事：**CSS 不受影响**，样式表链接与内联样式照常渲染，页面看起来一模一样；**页面永远不水合**，所以其上没有任何可交互的东西。

导航行为也跟着变，这部分文档写得最细：

- 没有脚本的页面用整页加载导航。为了让导航仍然快，Nuxt 会在这类页面上发出一个 [speculation rules](https://developer.mozilla.org/en-US/docs/Web/API/Speculation_Rules_API) 标签，让支持的浏览器在用户点链接之前预取与预渲染目标；这个标签是声明式 JSON、**不执行任何 JavaScript**。
- 规则的作用域是**你的页面路由**（这些可以安全 `GET`），而不是所有同源链接，所以像 `/logout` 这种指向服务端路由的链接不会被 speculative fetch。**但 catch-all 页面（例如 `pages/[...slug].vue`）匹配所有路径，会把作用域重新放宽到所有同源链接。**
- 客户端导航到一个被 `noScripts` 规则覆盖的路由时，改为整文档加载，好让那条路由真的是"无脚本被服务端出来"的，而不是交给客户端路由渲染。**构建期已知会被这类规则覆盖的页面，会被完全从客户端 bundle 里剔除。**
- 带脚本的页面也会发出自己的 speculation rules 标签，作用域限定在 `noScripts` 的路由模式上；**启用了视图过渡时，这些页面还会额外采用声明式的跨文档视图过渡**来给导航做动画。

还有两处与其他能力的冲突写在文末：

- **带 `noScripts` 路由规则的路由总是用缓冲（非流式）渲染器渲染。**
- 只输出静态 HTML 的服务端组件没问题，但**用 `nuxt-client` 水合的组件与交互式岛屿槽依赖一段内联脚本来挪动 teleport 的内容，所以它们在 `noScripts` 路由上不会变得可交互**；当 `features.noScripts` 是全应用设置且组件岛屿处于启用状态时，Nuxt 也因为同样原因退回缓冲渲染。

想把这些组合起来用，文档指向 mostly-static 站点那篇配方（把 `noScripts`、预渲染、岛屿与懒水合合在一起）。

## `future` 三项，重点是它会把 defaults 改掉

**`compatibilityVersion`**：这一句是理解三节关系的关键——**把 `compatibilityVersion` 设为 `5` 会改变你整个 Nuxt 配置里的默认值，以选择加入 Nuxt v5 的行为**，其中就包括启用 Vite Environment API。示例只有 `future: { compatibilityVersion: 5 }` 一层，细节指向升级指南的 testing Nuxt 5 小节。

前面速查表里那批"默认值挂在 `compatibilityVersion` 上"的开关（`extractSerializablePageMeta`、`viteEnvironmentApi`、`clientNodePlaceholder`、`normalizePageNames`、`navigateToEarlyReturn`、`routeTypedFetch`、`payloadExtraction` 取值变 `'client'`、`watcher` 变 `'builder'`、`useState.resetOnClear` 变 `true`），**都是这一条造成的**。

**`multiApp`**：提前选择加入实验性的多应用支持，进度可以看 tracker issue #21635。这一节**没有说明"多应用"具体指什么形态**，要理解只能去跟那个 issue。

**`typescriptBundlerResolution`**：为 TypeScript 启用 `'Bundler'` 模块解析模式，文档称这是 Nuxt、Vite 这类框架的推荐设置，**在使用带 `exports` 的现代库时能改善类型支持**，并指向 TypeScript 的那个原始 PR。反过来设成 `false` 就用 legacy 的 `'Node'` 模式——**那才是 TypeScript 自己的默认**。

## 三节之间怎么分工

- 想关掉一个已经在跑的能力（日志流、样式内联、脚本渲染）→ `features`。
- 想试一个还没定的能力（流式渲染、类型化 fetch、岛屿、构建缓存）→ `experimental`，并接受"可能被改或删"。
- 想让一批默认值提前变成未来版本的样子 → `future.compatibilityVersion`，而不是逐个开 `experimental` 键；反过来，**只想采用其中一项时，就单独设那个 `experimental` 键**——文档对 `viteEnvironmentApi` 与 `routeTypedFetch` 都明说了显式设置可以"无论兼容版本如何都选定这一边"，也允许"先升到 5、之后再单独接"。

一个容易忽略的事实：**这三节都是构建期读取的配置**。模块能改它们（`nuxt.options.experimental ||= {}` 那种写法），也能读它们，但用户在 `nuxt.config` 里的设置是这些默认值的最终来源。

下一篇讲 Nuxt 的内部机制：构建管线与文件生成顺序，也就是 `.nuxt` 目录里那些东西是怎么被生成出来的。

## 本文依据

- https://nuxt.com/docs/4.x/guide/going-further/features
- https://nuxt.com/docs/4.x/guide/going-further/experimental-features
