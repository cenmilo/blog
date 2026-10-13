---
url: /posts/d38d11e20bbe621d390acb7698da2e97/
title: 093 - NuxtWelcome：新项目第一屏那块欢迎界面是什么
date: 2026-11-12T09:00:00+08:00
updated: 2026-11-12T09:00:00+08:00
author: cmdragon

summary:
  摘要：用脚手架新建项目、跑起开发服务器之后看到的那一屏，不是 Nuxt 的某个内置页面，而是一个组件：官方给 `<NuxtWelcome>` 的定义是"向用 starter 模板创建的新项目中的用户问好"，并说明**它包含指向 Nuxt 文档、源代码和社交媒体账号的链接**。用法只有一行——在 `app/app.vue` 的模板里写 `<NuxtWelcome />`。官方的定位说得更清楚：**这个组件属于 `@nuxt/ui-templates` 包**，源码位置在 `packages/ui-templates/templates/welcome/index.html`，也就是说它是一份 UI 模板而不是一个有配置项的组件——官方这一节没有给它任何 prop、事件或插槽。所以处理它的正确方式是替换而不是配置：把 `app/app.vue` 里那一行换成你自己的内容，`app.vue` 仍然是应用的入口点、仍然会为每一个路由渲染它的内容；如果你之后加了 `app/pages/` 目录，就要在里面改用 `<NuxtPage>` 显示当前页面，而没有 pages 目录时 Nuxt 根本不会引入 vue-router 依赖。另外官方在安装页写明了第一屏的来源：跑完开发命令后"浏览器窗口应当会自动为 http://localhost:3000 打开"，并给了 Windows 的一条提示——本地开发服务器不要用 `localhost:3000`，改用 `127.0.0.1` 加载会快得多（Windows 慢 DNS 解析）。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxtwelcome
  - starter
  - 首屏
  - 组件

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


新建项目跑起来，浏览器里那一大块带链接的欢迎屏——它是什么？很多人以为那是 Nuxt 的默认页面，于是要么找不到地方改它，要么不敢删。这一篇讲清楚：**它是一个组件，属于一个独立的包，正确处理方式是替换掉。**

## 官方怎么定义它

| 官方原话 | 要点 |
| --- | --- |
| "`<NuxtWelcome>` 组件向用 starter 模板创建的新项目中的用户问好。" | 它是给**starter 新项目**用的 |
| "它包含指向 Nuxt 文档、源代码和社交媒体账号的链接。" | 你在第一屏点到的那几个链接就是它带来的 |
| "这个组件是 `@nuxt/ui-templates` 包的一部分。" | 出处：一个专门放 UI 模板的包 |
| 源码链接指向 `packages/ui-templates/templates/welcome/index.html` | 它是一份 **HTML 模板** |

用法官方只给了一种：

```vue [app/app.vue]
<template>
  <NuxtWelcome />
</template>
```

官方还给了一个预览地址，写着"Preview the `<NuxtWelcome />` component."——想先看清楚那是什么东西，可以在不建项目的情况下先看预览。

**这一页没有 props、没有事件、没有插槽。** 这不是遗漏，而是它的定位：一份成品模板，用来给刚建好的项目一个像样的第一屏，不参与你的业务结构。

## 第一屏是怎么出现的

流程上有两个部分，别混在一起：

| 环节 | 发生了什么 |
| --- | --- |
| 起服务 | 官方在安装页写的是："做得好！浏览器窗口应当会自动为 http://localhost:3000 打开。" |
| 渲染内容 | 打开后看到的，是 `app/app.vue` 里那份 `<NuxtWelcome />` |

也就是说：**自动打开浏览器是开发服务器的事，那一屏内容是你项目里的一个组件。** 前者不会因为你换了组件而消失，后者完全由你决定。

顺带一条官方给出的 Windows 提示，正好和"第一屏打开得慢"有关：**Windows 慢 DNS 解析——在 Windows 上，本地开发服务器不要用 `localhost:3000`，改用 `127.0.0.1` 可以获得快得多的加载体验。**

## 替换掉它的正确做法

`app.vue` 是你的 Nuxt 应用的主组件，默认情况下 Nuxt 把这个文件当作**入口点**，并为应用的**每一个路由**渲染它的内容。所以把欢迎屏换成自己的内容，不涉及任何注销或关闭操作，直接改模板即可。

| 你的项目形态 | `app.vue` 应该写成什么 |
| --- | --- |
| 一个落地页、没有 `app/pages/` 目录 | 直接写你自己的模板，例如 `<h1>Hello World!</h1>`；官方说明此时**Nuxt 不会包含 vue-router 依赖** |
| 已经有 `app/pages/` 目录 | `<NuxtPage />`（官方：当你有 pages 目录时，你需要使用 `<NuxtPage>` 组件来显示当前页面） |
| 想留全局页头页脚 | `<header>` + `<NuxtPage />` + `<footer>` 三段并列写在 `app.vue` 里 |
| 干脆不写 `app.vue` | 可以：官方写明如果你有 `app/pages/` 目录，`app.vue` 文件是可选的，Nuxt 会自动包含一个默认的 `app.vue` |

要注意的只有一件：**你往 `app.vue` 里加的任何东西（JS 和 CSS）都会是全局的，并包含在每一个页面里。** 欢迎屏本身没有内容量问题，但替换它时很容易顺手把只在某一页需要的东西塞进来。

## 什么时候它还会再出现

| 情形 | 结果 |
| --- | --- |
| 每次用 starter 脚手架新建一个项目 | 又会出现一次——它是 starter 项目里的内容，不是全局开关 |
| 你已经把 `app.vue` 改成自己的内容 | 不会再出现；这个组件不在你的渲染链上 |
| 你留着 `<NuxtWelcome />` 但下面还写了别的 | 会一起渲染——它是普通组件，和其他内容并列，不是"未初始化提示" |

第三条是常见的误解：它不是一个"检测到空壳才显示"的东西。它出现在你的屏幕上，只是因为 `app.vue` 的模板里写了那一行。

## 自查

1. `<NuxtWelcome>` 属于哪个包？官方在这一节给了它哪些配置项？
2. 第一屏那几个链接分别指向什么？这是官方哪一句说的？
3. 自动打开浏览器是组件行为还是开发服务器行为？
4. 没有 `app/pages/` 目录时，Nuxt 少了哪个依赖？
5. 有 pages 目录时，把欢迎屏换掉之后 `app.vue` 里应该出现什么？
6. Windows 上官方对第一屏打开慢给的具体建议是什么？

## 下一篇

094 展开 `<NuxtLayout>`：`name` 可以给 `false`、布局名会被规范化成 kebab-case、给了不存在的布局名时什么都不渲染（这才轮到 `fallback` 出场），以及做过渡时官方为什么建议它**不要**当页面的根元素。

## 本文依据

- `https://nuxt.com/docs/4.x/api/components/nuxt-welcome`
  - 原文："The \<NuxtWelcome> component greets users in new projects made from the starter template."
  - 原文："It includes links to the Nuxt documentation, source code, and social media accounts."
  - 原文示例（文件标注 `app/app.vue`）：`<template><NuxtWelcome /></template>`
  - 原文（read-more）："Preview the `<NuxtWelcome />` component."，指向 `https://templates.ui.nuxtjs.org/templates/welcome`
  - 原文（tip）："This component is part of `@nuxt/ui-templates` package."
  - 原文 Source 链接：`https://github.com/nuxt/nuxt/blob/main/packages/ui-templates/templates/welcome/index.html`
  - 说明：本页未列出任何 prop、事件或插槽
- `https://nuxt.com/docs/4.x/getting-started/installation`
  - 原文（tip）："Well done! A browser window should automatically open for <http://localhost:3000>."
  - 原文（additional notes for an optimal setup）："**Windows slow DNS resolution**: Instead of using `localhost:3000` for local dev server on Windows, use `127.0.0.1` for much faster loading experience on browsers."
  - 原文：页面描述 "Get started with Nuxt quickly with our online starters or start locally with your terminal."；在线沙箱链接指向 `https://stackblitz.com/github/nuxt/starter?file=app%2Fapp.vue`
- `https://nuxt.com/docs/4.x/directory-structure/app/app`
  - 原文："The app.vue file is the main component of your Nuxt application."
  - 原文（tip）："If you have a `app/pages/` directory, the `app.vue` file is optional. Nuxt will automatically include a default `app.vue`, but you can still add your own to customize the structure and content as needed."
  - 原文（Minimal Usage）："With Nuxt, the `app/pages/` directory is optional. If it is not present, Nuxt will not include the vue-router dependency. This is useful when building a landing page or an application that does not require routing."；示例 `<template><h1>Hello World!</h1></template>`
  - 原文（Usage with Pages）："When you have a `app/pages/` directory, you need to use the `<NuxtPage>` component to display the current page"；"You can also define the common structure of your application directly in `app.vue`. This is useful when you want to include global elements such as a header or footer"
  - 原文（note）："Remember that `app.vue` acts as the main component of your Nuxt application. Anything you add to it (JS and CSS) will be global and included in every page."
- `https://nuxt.com/docs/4.x/getting-started/views`
  - 原文："By default, Nuxt will treat this file as the **entrypoint** and render its content for every route of the application."
