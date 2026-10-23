---
url: /posts/1949c84f1bd522662482c1383ec4f8ef/
title: 353 - features 节：文档里那几个可选特性开关
date: 2027-02-06T11:00:00+08:00
updated: 2027-02-06T11:00:00+08:00
author: cmdragon

summary:
  摘要：nuxt.config 的 features 节是一组"可打开的特性开关"，和 future（提前启用未来默认）、experimental（可能移除的实验特性）是三个不同命名空间。features 这一节目前文档列了三项：devLogs（把服务端日志流到浏览器，可在 dev:ssr-logs 钩子处理，设 silent 则不打印到控制台）、inlineStyles（渲染 HTML 时内联样式、仅 Vite 可用，可传函数按组件决定）、noScripts（不输出脚本让页面纯 HTML+CSS，三档 false/'production'/'all'）。这一篇讲这三个各干什么、默认值、和它们的连带影响（比如 noScripts 后页面永不水合全无交互）。

categories:
  - 前端开发

tags:
  - nuxt4
  - features
  - noScripts
  - inlineStyles
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


配置里三个命名空间容易混：`future`（352，未来会变默认的）、`experimental`（263/267，不保证留）、还有这个 `features`。nuxt-config 的 `## features` 节本身就一句 read-more 指向 features 指南——真正的内容在那一节，列了三个开关。这一篇讲它们。

## 开关一：devLogs

> Stream server logs to the client as you are developing. These logs can be handled in the `dev:ssr-logs` hook.
> By default, this is enabled in development (when test mode is not active). If set to `silent`, the logs will not be printed to the browser console.

```ts
features: { devLogs: true }
```

作用：开发时把服务端日志流到浏览器、由 `dev:ssr-logs` 钩子处理（275 那张表里就有这个钩子）。默认在开发开启（测试模式下不开）。设 `'silent'` 则日志不打到浏览器控制台。排"服务端 console.log 去哪了"时有用。

## 开关二：inlineStyles

> Inlines styles when rendering HTML. This is currently available only when using Vite.
> You can also pass a function that receives the path of a Vue component and returns a boolean indicating whether to inline the styles for that component. It defaults to `(id) => id.includes('.vue')`.

```ts
features: { inlineStyles: false } // 或一个函数决定要不要内联
```

渲染 HTML 时把样式内联进页面（能少一些样式外链）。两条要记：只 Vite 可用；默认对任何 `.vue` 组件都内联。324 讲流式时提过"内联样式随路由 CSS 一起进 post-shell 块"，就是靠这个机制。

## 开关三：noScripts（最需要一个整节讲）

> Turn off rendering of Nuxt scripts and JavaScript resource hints, so pages ship as pure HTML and CSS.

三档取值：

- `false`（默认）：正常输出脚本。
- `'production'`（或 `true`，被归一成 `'production'`）：只在生产构建省掉脚本，dev 下 HMR 等照常。
- `'all'`：开发和生产都省掉脚本。

它的连带后果（都来自同页），决定你到底要不要开：

- **页面永不水合、全无交互**："CSS is unaffected ... the page simply never hydrates, which means nothing on it is interactive." 看着一样，但按钮表单都不动了。
- 导航变成**整文档加载**；为救速度，Nuxt 在这些页发一个 speculation rules 标签（声明式 JSON、不跑 JS）让浏览器预取目标；且规则限定在你的页面路由内（`/logout` 这类服务端路由不会被预取）——但 catch-all 页会把范围放宽到所有同源链接。
- 被 `noScripts` 覆盖且构建期可知的页面，会**整个从客户端 bundle 排除**；`noScripts` 路由总是缓冲（非流式）渲染；`nuxt-client` 与交互式岛屿插槽不会变可交互。

它也能**按路由**开（routeRules 的 `noScripts`，全模式生效）：

```ts
routeRules: { '/blog/**': { noScripts: true } }
```

省掉的 HTML 内容官方列了四类：入口 `<script>`、import map、内联 payload `<script>`（及 payload 预取链接）、JS 资源提示（preload/prefetch）。CSS 不受影响。适合"内容为主、几乎不需交互"的纯静态/博客页——这也是它和 264/流式那一串取舍连起来的地方。

## 三个命名空间别混

| 命名空间 | 语义 |
| --- | --- |
| `features`（本篇） | 可选特性开关：devLogs、inlineStyles、noScripts |
| `future`（352） | 未来会变默认的行为，提前打开 |
| `experimental`（263/267） | 实验特性，可能被改或被删 |

## 自查

- 想在浏览器看服务端日志：features.devLogs（'silent' 只收不打印）。
- 想内联样式减外链：features.inlineStyles（仅 Vite，可给函数按组件决定）。
- 想让页面纯 HTML+CSS、不要 JS：features.noScripts 或按路由 routeRules.noScripts，但记住代价是页面不水合、无交互。
- features / future / experimental 是三组，别当一个。

## 下一篇

下一篇 354 讲 telemetry：nuxt 的匿名使用数据上报，怎么关、去哪看它收集了什么。
