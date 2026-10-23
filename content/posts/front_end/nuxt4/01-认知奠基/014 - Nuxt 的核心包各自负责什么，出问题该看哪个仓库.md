---
url: /posts/9b0b6554377c3512966a1dff3dd662d9/
title: 014 - Nuxt 的核心包各自负责什么，出问题该看哪个仓库
date: 2026-10-16T11:00:00+08:00
updated: 2026-10-16T11:00:00+08:00
author: cmdragon

summary:
  摘要：官方列出的核心包是五个：核心引擎 nuxt；打包器 @nuxt/vite-builder、@nuxt/rspack-builder、@nuxt/webpack-builder；命令行工具 @nuxt/cli；服务端引擎 nitro；开发工具包 @nuxt/kit。Nitro 自带的能力被列成六条：Node.js、浏览器、service worker 等的跨平台支持、开箱即用的 Serverless 支持、API 路由支持、自动代码分割与异步加载 chunk、静态+serverless 站点的混合模式、带热模块替换的开发服务器。服务端 API 层由 Nitro 提供而它内部使用 h3；直接调用路由靠全局可用的 $fetch，它在浏览器里向服务器发 API 请求、在服务端则直接调用相关函数从而省掉一次额外的 API 调用，$fetch 用的是 ofetch；只要返回一个值而不是用 res.end() 发送响应，Nitro 就会为这些路由生成类型，可用 $fetch 或 useFetch 访问。Nitro 产出的是独立于 node_modules 的独立服务端 dist，Nuxt 在运行 nuxt build 时把它输出到 .output 目录。

categories:
  - 前端开发

tags:
  - nuxt4
  - nitro
  - 架构
  - 排错
  - h3

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

012 讲说了产物，这一讲讲制造产物的那些包。它的实际用途只有一个，出问题时知道该去哪个仓库搜 issue，而不是在 `nuxt/nuxt` 里翻半天。

五个核心包，官方在介绍页末尾列的清单。Nuxt 由不同的核心包组成，核心引擎是 `nuxt`，打包器是 `@nuxt/vite-builder`、`@nuxt/rspack-builder`、`@nuxt/webpack-builder`，命令行工具是 `@nuxt/cli`，服务端引擎是 `nitro`，开发工具包是 `@nuxt/kit`。

给每张脸配一句它的锅长什么样。

`nuxt`，核心引擎，读配置、扫目录、生成 `.nuxt`、协调构建，典型症状是约定不生效、路由、自动导入、布局没生成。

`@nuxt/vite-builder`，默认打包器，管 HMR 与生产打包，典型症状是热更新异常、产物体积、别名解析。

`@nuxt/webpack-builder` 和 `@nuxt/rspack-builder`，可选打包器，症状只在切换 `builder` 后出现的新问题。

`@nuxt/cli`，命令行，`dev`、`build`、`generate`、`prepare`、`typecheck`，症状是命令不认识某个 flag、CLI 参数行为变了。

`nitro`，服务端引擎，API 路由、中间件、产物与部署预设，症状是接口 404、`server/` 里的东西不生效、部署产物形态不对。

`@nuxt/kit`，写模块用的工具函数，症状是模块注册的东西没进项目。

注意第二行，打包器是多个，官方列了三个。Vite 是默认，002 讲那条我们默认使用 Vite，换 builder 时官方要求显式安装对应包，构建配置那一段会讲。

Nitro，这一层的能力比你想的多。官方对它的定位与自带功能是，在构建 Nuxt 时，我们创建了一个新的服务端引擎，Nitro，它随附许多特性。Node.js、浏览器、service worker 等等的跨平台支持。开箱即用的 Serverless 支持。API 路由支持。自动代码分割与异步加载的 chunk。静态加 serverless 站点的混合模式。带热模块重载的开发服务器。

最后一条特别值得看，连 HMR 都属于 Nitro 的能力清单，不只是打包器的事。

API 层，Nitro 内部用的是 h3。服务端 API 端点与中间件由 Nitro 添加，而它内部使用 h3，关键特性包括，处理器可以直接返回对象或数组，作为自动处理的 JSON 响应，处理器可以返回 Promise，它们会被 await，`res.end()` 与 `next()` 也受支持，以及为请求体解析、cookie 处理、重定向、响应头等等提供的辅助函数。

这段解释了一件很多人困惑的事，`server/api/` 里那些 `defineEventHandler`、`readBody`、`setCookie` 到底是谁提供的，不是 Nuxt，是 h3。这也和后面要讲的一条冲突有关，服务端同拼写的自动导入名字其实是 h3 的 helper，而不是 `nuxt/server` 的那批。

`$fetch`，同一次调用，两种走法。官方这一段建议多读两遍。Nitro 允许通过全局可用的 `$fetch` 辅助函数进行直接路由调用。如果运行在浏览器里，它会向服务器发起一次 API 调用，如果运行在服务端，它会直接调用相关的函数，从而省掉一次额外的 API 调用。`$fetch` 的 API 使用 ofetch，关键特性包括，自动解析 JSON 响应，需要时仍可访问原始响应，请求体与参数会被自动处理，并带上正确的 `Content-Type` 头。

这就是 SSR 首屏不会请求自己两次的实现原理。服务端渲染时调 `/api/x`，走的是函数调用而不是 HTTP 回环，浏览器端才真的发请求。看到服务端日志里出现对自己接口的 HTTP 请求，就该回头检查调用方式。

类型化路由，一个条件决定有没有类型。当你使用 API 路由或中间件时，Nitro 会为这些路由生成类型，只要你是返回一个值，而不是用 `res.end()` 来发送响应。使用 `$fetch()` 或 `useFetch()` 时就能访问这些类型。

这条把要不要返回类型变成一个写法选择，`return` 值，`$fetch` 有类型，`res.end()`，没有。官方在别处也重申过这个取向，`defineEventHandler` 保留处理器的返回类型，并建议给返回的值加标注，而不是给处理器加标注。

独立产物，为什么它能跑在任何地方。Nitro 生成一个独立于 `node_modules` 的独立服务端 dist。Nuxt 2 里的服务端不是独立的，它需要 `nuxt start` 时把 Nuxt 核心的一部分卷进来运行，使用 `nuxt-start` 或 `nuxt` 分发包，或自定义的程序化用法，这既脆弱又容易损坏，也不适合 serverless 与 service worker 环境。Nuxt 在运行 `nuxt build` 时把这个 dist 生成到 `.output` 目录里。输出包含在任何环境里运行你的 Nuxt 服务端的运行时代码，包括实验性的浏览器 service worker，并服务你的静态文件，使它成为 JAMstack 的真正混合框架。此外，Nuxt 实现了一个原生存储层，支持多源驱动与本地资源。

012 讲那句移除任何 Node.js 模块，除 polyfill，的由来就在这里，产物自带运行时，不依赖宿主 `node_modules`。这也是边缘和 Serverless 能跑同一个产物的技术基础。

顺带记一个新名词，原生存储层，官方说它支持多源驱动与本地资源，以后你要读构建期资源或做 KV 读写，入口在这一层，服务端那一段会展开。

一条查问题的路径。按报错文本出现在哪来选仓库，是我用过最省事的顺序。

报错栈里出现 `nitro`、`h3`、`ofetch`，去对应仓库，Nitro、h3。

报错发生在 `nuxt dev` 的参数解析、命令本身，去 `@nuxt/cli`。

约定不生效，路由、自动导入、布局、中间件，去 `nuxt/nuxt` 核心，先跑一次 dev 或 build 看 `.nuxt` 里生成了什么。

产物里少了什么、部署形态不对，先确认命令，`build` 还是 `generate`，再看 Nitro 预设。

你在写模块、`addServerHandler` 或 `addComponent` 之类没效果，去 `@nuxt/kit`。

## 自查

- [ ] 五个包你能各说一句「它管什么」吗？
- [ ] 知道默认打包器是 Vite、另外两个 builder 要显式切换吗？
- [ ] 你的接口返回的是值还是 `res.end()`？这决定了 `$fetch` 有没有类型。
- [ ] 服务端渲染时有没有真的对自己发 HTTP 请求（应该走直接调用）？
- [ ] 产物为什么能跑在无 `node_modules` 的环境上，你说得出来吗？
- [ ] 下次提 issue 之前，你能定位到具体是哪个包的行为吗？

## 下一篇

015 - ESM 与 CommonJS 互操作：库引进来就报语法错误的真实原因
