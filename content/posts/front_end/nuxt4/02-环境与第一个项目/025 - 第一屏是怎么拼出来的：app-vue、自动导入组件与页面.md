---
url: /posts/30d8bd3e2fce763c4850fd422111b35a/
title: 第一屏是怎么拼出来的：app.vue、自动导入组件与页面
date: 2026-10-20T10:00:00+08:00
updated: 2026-10-20T10:00:00+08:00
author: cmdragon

summary:
  摘要：官方的 Views 页把界面拆成四层：app.vue、components、pages、layouts。其中最关键的一句是"默认情况下，Nuxt 会把 app.vue 当作入口点，并为应用的每一个路由渲染它的内容"——它替代了你在 Vue 里熟悉的那个 main.js，官方明说"创建 Vue 应用的那件事由 Nuxt 在幕后完成"。组件放在 app/components/ 目录下就能在全应用直接使用，不需要显式 import；页面放在 app/pages/ 目录里，每个文件代表一条路由，但要让页面真的显示出来，你需要在 app.vue 里放一个 <NuxtPage />（或者干脆删掉 app.vue，用默认入口）。

categories:
  - 前端开发

tags:
  - nuxt4
  - app-vue
  - 组件
  - 页面
  - 自动导入

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


浏览器打开 `127.0.0.1:3000` 之后，你看到的是"第一屏"。这一篇只回答一个问题：**这一屏的内容，是从哪个文件开始拼出来的。** 官方在 Views 页给了四层，我们从最外那层讲起。

（先提一个阅读细节：官方文档里这些文件全都写成 `app/app.vue`、`app/components/`、`app/pages/`，带 `app/` 前缀。这是目录结构的事，050、051 会讲清楚；这一篇照文档原样写。）

## 第一层：`app.vue` 是入口点

官方的那句话值得完整抄下来：

> 默认情况下，Nuxt 会把这个文件当作**入口点（entrypoint）**，并为应用的**每一个路由**渲染它的内容。

内容可以简单到什么程度：

```vue [app/app.vue]
<template>
  <div>
    <h1>Welcome to the homepage</h1>
  </div>
</template>
```

这个 `h1` 就是你第一屏看到的东西。注意"为每一个路由渲染它的内容"这半句——**在你还没引入页面之前，不管你访问哪个路径，渲染的都是这一个文件**。这是后面很多"为什么我的页面不显示"的根因。

## 你找不到的那个 `main.js`

Vue 老用户到这一步一定会问一句，官方直接把问题替你说出来了：

> 如果你熟悉 Vue，你可能会好奇 `main.js` 在哪（那个通常用来创建 Vue 应用的文件）。Nuxt 在幕后完成这件事。

所以这不是"约定文件名"，而是**根本没有这个文件**——创建应用实例、挂载、注册插件这些动作被框架接管了（013 讲过同构渲染下这些代码在两端各跑一次）。你在 `main.js` 里习惯写的 `app.use(...)`、`app.mount('#app')`，在 Nuxt 里对应的是插件（061）和这个 `app.vue`。

## 第二层：组件放进目录就能用

官方的定义：

> 大多数组件是可复用的界面片段，比如按钮和菜单。在 Nuxt 里，你可以在 `app/components/` 目录中创建这些组件，它们会**在你整个应用中自动可用，无需显式导入**。

文档给的例子就两个文件：

```vue [app/app.vue]
<template>
  <div>
    <h1>Welcome to the homepage</h1>
    <AppAlert>
      This is an auto-imported component.
    </AppAlert>
  </div>
</template>
```

```vue [app/components/AppAlert.vue]
<template>
  <span>
    <slot />
  </span>
</template>
```

看第二份文件：**没有 `<script>`，没有 `export default`，没有定义 `AppAlert` 这个名字**。名字是从文件名来的。

而 `app.vue` 里没有一行 `import { AppAlert } ...`——它直接把 `<AppAlert>` 写进模板就能用，文档在那段代码里标注的就是"This is an auto-imported component"。

这一条是 011 讲的"框架加的能力"里最容易忘的一条。它的边界（哪些目录会被扫、名字怎么生成、前缀规则）在 062 逐条讲；**现在只需要记住：把文件放进 `app/components/`，标签名就是文件名**。

## 第三层：页面要显式接上去

页面这一层的规则是：

> 页面代表每个具体路由模式的视图。`app/pages/` 目录里的**每个文件代表一条不同的路由**，显示它的内容。

但注意下面这句，它是有条件的：

> 要使用页面，先创建一个 `app/pages/index.vue` 文件，并在 `app/app.vue` 中添加 `<NuxtPage />` 组件（**或者移除 `app/app.vue`，使用默认入口**）。

这句话是这一篇最实用的部分。它给了两条路：

**路线 A：在 `app.vue` 里放 `<NuxtPage />`。**

```vue [app/pages/index.vue]
<template>
  <div>
    <h1>Welcome to the homepage</h1>
    <AppAlert>
      This is an auto-imported component
    </AppAlert>
  </div>
</template>
```

```vue [app/pages/about.vue]
<template>
  <section>
    <p>This page will be displayed at the /about route.</p>
  </section>
</template>
```

此时 `app.vue` 变成了"外壳"，`index.vue` 和 `about.vue` 的内容在 `<NuxtPage />` 那个位置渲染出来。`/about` 这个路径就是文件名给的——文档原话："This page will be displayed at the /about route."

**路线 B：删掉 `app.vue`。** 让框架用它自己的默认入口，页面直接渲染。

于是就有了那个新手最容易撞上的现象：**你建了 `pages/about.vue`，访问 `/about` 却还是看到首页那句 `Welcome to the homepage`。** 原因是你 `app.vue` 里没写 `<NuxtPage />`——回到开头那句"为每一个路由渲染它的内容"：没接上之前，`app.vue` 渲染的就是它自己。

## 第四层：布局（下一篇）

Views 页往下还有 layouts 这一层，官方给的定义是"布局是页面的包装器"，并且给了一个组合写法：

```vue [app/app.vue]
<template>
  <div>
    <NuxtLayout>
      <NuxtPage />
    </NuxtLayout>
  </div>
</template>
```

这一层的规则（默认布局叫什么、怎么给某个页面换布局、只有一层布局时官方反而建议别用它）单独放下一篇。

## 自查

- [ ] 在你还没引入页面之前，访问任意路径看到的是同一个内容——这个现象对应官方哪句话？
- [ ] Nuxt 项目里为什么没有 `main.js`？那部分工作去哪了？
- [ ] `app/components/AppAlert.vue` 这个名字是在文件的哪一行定义的？
- [ ] 建了 `pages/about.vue` 但访问 `/about` 显示的还是首页，第一个要检查的是什么？
- [ ] 官方给的"使用页面"的两条路分别是什么？

## 下一篇

026 讲布局这一层：`<NuxtLayout>` 包 `<NuxtPage />` 的写法、`default.vue` 为什么是默认，以及官方那条"只有一个布局时反而建议用 app.vue"的建议。

## 本文依据

- `getting-started/views`（app.vue 一节：默认情况下 Nuxt 把该文件当作 entrypoint 并为应用的每一个路由渲染其内容、示例 app/app.vue 的 h1、tip"If you are familiar with Vue, you might wonder where main.js is（那个通常创建 Vue 应用的文件）。Nuxt does this behind the scene"；Components 一节：组件放 app/components/ 目录后在全应用自动可用而无需显式导入、示例 app/app.vue 直接使用 <AppAlert> 并标注"This is an auto-imported component"、app/components/AppAlert.vue 只有含 <slot /> 的 template；Pages 一节：pages 目录里每个文件代表一条不同路由、要使用页面需创建 app/pages/index.vue 并在 app/app.vue 中添加 <NuxtPage />（或移除 app/app.vue 使用默认入口）、示例 index.vue 与 about.vue 且 about 的内容为"This page will be displayed at the /about route."；Layouts 一节：布局是页面包装器、app/app.vue 中 <NuxtLayout> 包 <NuxtPage /> 的示例）
