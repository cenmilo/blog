---
url: /posts/0fd5f48299eceafc47bc662070825fa1/
title: 路由生成推演：从 pages 目录到 vue-router 记录
date: 2026-11-17T11:00:00+08:00
updated: 2026-11-17T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇做一次完整推演：**一个 `app/pages/` 目录，怎么变成 vue-router 能用的路由记录。** 官方给的第一句是"Nuxt 文件系统路由为 `pages/` 目录里的每一个文件创建一条路由"，第二句把机制与收益一起说了："Nuxt 的一个核心特性是文件系统路由器。`app/pages/` 目录里的每一个 Vue 文件都创建一个对应的 URL（或路由）来显示该文件的内容。通过对每个页面使用动态导入，Nuxt 利用代码分割只发送所请求路由所需的最少量 JavaScript。"生成规则的依据也写清了："Nuxt 路由基于 vue-router，并从 `app/pages/` 目录里创建的每一个组件**依据其文件名**生成路由"，而"这套文件系统路由使用命名约定来创建动态与嵌套路由"。官方给的那组输入输出是本次推演的核心：目录是 `pages/about.vue`、`pages/index.vue`、`pages/posts/[id].vue`，生成的路由表是三条记录——`path: "/about"` 对应 `component: "pages/about.vue"`、**`path: "/"` 对应 `component: "pages/index.vue"`**、`path: "/posts/:id"` 对应 `component: "pages/posts/[id].vue"`（方括号在这一步变成 vue-router 的冒号参数）。代码分割默认开启且官方称"对大多数应用是推荐做法"，要关掉得写 `vite.$client.build.rolldownOptions.output.codeSplitting: false`，而官方的劝退原话是"这会把所有 JavaScript 打进一个文件。它很少有益——即使连接较慢也通常会增加首次下载的体积——所以只有在你的场景里**实测过它有帮助**才关闭代码分割"。导航侧：`<NuxtLink>` "渲染一个 `<a>` 标签，其 `href` 属性设为页面的路由。**一旦应用完成水合，页面切换是通过 JavaScript 更新浏览器 URL 来执行的。这避免了整页刷新，并允许带动画的过渡。**"以及"当 `<NuxtLink>` 在客户端进入视口时，Nuxt 会自动**提前预取**被链接页面的组件与 payload（生成的页面），从而带来更快的导航"。取参数用 `useRoute()`，官方示例注释是"When accessing /posts/1, route.params.id will be 1"。

categories:
  - 前端开发

tags:
  - nuxt4
  - 路由
  - 文件式路由
  - 代码分割
  - nuxtlink

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


前面写了很多页面，但一直没回答"URL 是从哪儿来的"。这一篇就从三个文件开始，一路推到浏览器地址栏里那条路径——**每一步都用官方给的原话对上。**

## 官方给的两句话

| 官方原话 | 要点 |
| --- | --- |
| "Nuxt 文件系统路由为 `pages/` 目录里的每一个文件创建一条路由。" | 一条文件对应一条路由 |
| "Nuxt 的一个核心特性是文件系统路由器。`app/pages/` 目录里的每一个 Vue 文件都创建一个对应的 URL（或路由），用来显示该文件的内容。**通过对每个页面使用动态导入，Nuxt 利用代码分割只发送所请求路由所需的最少量 JavaScript。**" | 生成方式 + 附带收益 |
| "Nuxt 路由**基于 vue-router**，并从 `app/pages/` 目录里创建的每一个组件**依据其文件名**生成路由。" | 依据是文件名，底座是 vue-router |
| "这套文件系统路由使用**命名约定**来创建动态与嵌套路由。" | 方括号、目录层级这些约定 |

最后一条是这一篇的重点：命名约定就是"文件名 → URL"的那套规则，后面几篇专门讲它（动态段、可选段、catch-all、命名视图、路由组）。

## 推演第一步：给官方那组输入

```bash [Directory Structure]
-| pages/
---| about.vue
---| index.vue
---| posts/
-----| [id].vue
```

三个文件、两层目录、一对中括号。先别看结果，自己按"每个文件一条路由"推一遍。

## 推演第二步：官方给出的输出

```json [Generated Router File]
{
  "routes": [
    {
      "path": "/about",
      "component": "pages/about.vue"
    },
    {
      "path": "/",
      "component": "pages/index.vue"
    },
    {
      "path": "/posts/:id",
      "component": "pages/posts/[id].vue"
    }
  ]
}
```

三处细节值得单独指出来：

| 观察 | 说明 |
| --- | --- |
| **`index.vue` 的 path 是 `/`** | 官方在页面目录那一页也单独写过："The `app/pages/index.vue` file will be mapped to the `/` route of your application." |
| `about.vue` → `/about` | 文件名直接当路径，没有 `.vue` |
| `[id].vue` → **`/posts/:id`** | 中括号在这一层被翻译成 **vue-router 的冒号参数** |
| `component` 是 `pages/...` | 记录里指的是**文件**，不是模块名 |
| 目录 `posts/` 自己**没有**多出一条路由 | 只有文件会成为路由；`posts/` 只是路径前缀（要一个 `posts` 页得写 `posts/index.vue`） |

**最后一行是新手最常问的："我建了目录为什么没有页面。"** 官方给的映射表里从来没有"目录本身"这一条——只有 `about.vue`、`index.vue`、`[id].vue` 这三个文件产生记录。

## 推演第三步：动态导入与代码分割

第一句里那条收益是真实的：官方说 Nuxt **对每个页面使用动态导入**，因此只发送所请求路由需要的最少 JavaScript。

要关掉它，官方给的配置相当长——这也侧面说明它不打算让你随手关：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  vite: {
    $client: {
      build: {
        rolldownOptions: {
          output: {
            codeSplitting: false,
          },
        },
      },
    },
  },
})
```

官方的态度写在同一段里：

> 代码分割默认启用，并且对大多数应用是推荐做法。**这会把所有 JavaScript 打进一个文件。它很少有益——即使连接较慢也通常会增加首次下载的体积——所以只有在你的场景里实测过它有帮助才关闭代码分割。**

## 第四步：链接怎么把这条路由用起来

官方对导航的定义：

> `<NuxtLink>` 组件用于把页面彼此链接起来。**它渲染一个 `<a>` 标签，其 `href` 属性设为该页面的路由。一旦应用完成水合，页面切换是通过 JavaScript 更新浏览器 URL 来执行的。这避免了整页刷新，并允许带动画的过渡。**

这里有两段时机，值得分开记：

| 时机 | 行为 |
| --- | --- |
| 水合之前 | 它是一个**真的 `<a href="...">`**——所以链接能被点击、能被爬虫索引、能被"在新标签页打开" |
| 水合之后 | 换页由 JS 接管：**更新浏览器 URL**，不整页刷新，因此可以做过渡 |

第二句里还有预取：**"当 `<NuxtLink>` 在客户端进入视口时，Nuxt 会自动提前预取被链接页面的组件与 payload（生成的页面），从而带来更快的导航。"**

官方的用法示例就是最朴素的字符串 `to`：

```vue [app/pages/index.vue]
<template>
  <header>
    <nav>
      <ul>
        <li><NuxtLink to="/about">About</NuxtLink></li>
        <li><NuxtLink to="/posts/1">Post 1</NuxtLink></li>
        <li><NuxtLink to="/posts/2">Post 2</NuxtLink></li>
      </ul>
    </nav>
  </header>
</template>
```

注意 103 说过的那件事——**预取在服务端-only 页面场景里会连带预取岛屿响应**；而这里 `/posts/1` 与 `/posts/2` 命中的是同一条记录 `/posts/:id`，只是参数不同。

## 第五步：`/posts/1` 里的 1 怎么取

官方给的入口是 `useRoute()`："可以在一个 Vue 组件的 `<script setup>` 块或 `setup()` 方法里使用 `useRoute()` 组合式函数来访问当前路由的详情。"

```vue [pages/posts/[id].vue]
<script setup lang="ts">
const route = useRoute()

// When accessing /posts/1, route.params.id will be 1
console.log(route.params.id)
</script>
```

串起来看这次推演就完整了：

```
posts/[id].vue  →  记录 /posts/:id  →  访问 /posts/1
                                     →  route.params.id === 1
```

参数是字符串还是数字，官方这句注释写的是 "will be 1"——注意路由参数来自 URL，实际值按字符串处理更稳妥，这条在参数那一节展开。

## 顺带两站（后面细讲）

推演走到这里，链路上还剩两处会碰到"路由"的配置项，先留个坐标：

**中间件。** 官方定义："Nuxt 提供了一个可定制的路由中间件框架，你可以在整个应用里使用它，非常适合抽取那些你想在导航到某个特定路由之前运行的代码。" 两条边界要现在就记住：

- （note）"**路由中间件运行在你 Nuxt 应用的 Vue 部分。尽管名字相似，它们与服务端 middleware 完全不同，后者运行在你应用的 Nitro 服务端部分。**"
- （important）"**路由中间件不会为服务端路由（例如 `/api/*`）或其他服务端请求运行。**"

三种类型：匿名（直接写在用到的页面里）、命名（放 `app/middleware/`，按需**异步导入**加载；名字**规范化为 kebab-case**，`someMiddleware` 变成 `some-middleware`）、全局（同目录加 `.global` 后缀，**在每次路由变化时自动运行**）。

官方的 `auth` 例子正好把两种写法配成一对：

```ts [middleware/auth.ts]
export default defineNuxtRouteMiddleware((to, from) => {
  // isAuthenticated() is an example method verifying if a user is authenticated
  if (isAuthenticated() === false) {
    return navigateTo('/login')
  }
})
```

```vue [pages/dashboard.vue]
<script setup lang="ts">
definePageMeta({
  middleware: 'auth',
})
</script>

<template>
  <h1>Welcome to your dashboard</h1>
</template>
```

**校验。** 官方说 Nuxt 通过 `definePageMeta()` 里的 `validate` 属性提供路由校验："该 `validate` 属性接受 `route` 作为参数。你可以返回一个布尔值来决定这是否是一个可以用这个页面渲染的有效路由。**如果你返回 `false`，会造成一个 404 错误。**你也可以直接返回一个带 `status`/`statusText` 的对象来自定义返回的错误。" 更复杂的用例官方给的替代就是匿名中间件。示例：

```vue [pages/posts/[id].vue]
<script setup lang="ts">
definePageMeta({
  validate (route) {
    // Check if the id is made up of digits
    return typeof route.params.id === 'string' && /^\d+$/.test(route.params.id)
  },
})
</script>
```

**这一段正好补上了前面那个"参数是什么类型"的问题**：官方这个校验同时判了 `typeof route.params.id === 'string'` 和"由数字组成"——也就是说 `/posts/1` 里拿到的 `id` 是字符串 `'1'`。

## 自查

1. 官方说路由是"依据什么"生成的？底座是哪个库？
2. 用官方那三个文件写出对应的三条 `path`。`index.vue` 的 path 是什么？
3. `[id].vue` 生成的记录里 `path` 长什么样？这一步发生了什么转换？
4. 建了 `pages/posts/` 目录为什么没有 `/posts` 这条路由？
5. 关掉代码分割的配置写在哪一层？官方的劝退理由是哪两句？
6. `<NuxtLink>` 在水合前后分别做什么？进入视口时 Nuxt 自动做了什么？
7. 命名路由中间件的名字怎么规范化？路由中间件对 `/api/*` 生效吗？

## 下一篇

111 讲首页与 404：`index.vue` 与 `index/` 目录两种写法的差别，`pages/[...slug].vue` 为什么常被当兜底，以及"未匹配路由"在不同渲染模式下的实际表现。

## 本文依据

- `https://nuxt.com/docs/4.x/getting-started/routing`
  - 原文（描述）："Nuxt file-system routing creates a route for every file in the pages/ directory."
  - 原文："One core feature of Nuxt is the file system router. Every Vue file inside the `app/pages/` directory creates a corresponding URL (or route) that displays the contents of the file. By using dynamic imports for each page, Nuxt leverages code-splitting to ship the minimum amount of JavaScript for the requested route."
  - 原文（note）："Code-splitting is enabled by default and is recommended for most apps. If you have a specific reason to ship a single bundle instead, you can disable it in your `nuxt.config`"；示例 `vite: { $client: { build: { rolldownOptions: { output: { codeSplitting: false } } } } }`；"This ships all JavaScript in a single file. It is rarely beneficial — it usually increases the initial download, even on slow connections — so only disable code-splitting if you have measured that it helps your case."
  - 原文（Pages）："Nuxt routing is based on vue-router and generates the routes from every component created in the `app/pages/` directory, based on their filename."；"This file system routing uses naming conventions to create dynamic and nested routes"
  - 原文（code-group）：目录结构 `pages/about.vue`、`pages/index.vue`、`pages/posts/[id].vue`；生成的路由文件三条记录 `{"path": "/about", "component": "pages/about.vue"}`、`{"path": "/", "component": "pages/index.vue"}`、`{"path": "/posts/:id", "component": "pages/posts/[id].vue"}`
  - 原文（Navigation）："The `<NuxtLink>` component links pages between them. It renders an `<a>` tag with the `href` attribute set to the route of the page. Once the application is hydrated, page transitions are performed in JavaScript by updating the browser URL. This prevents full-page refreshes and allows for animated transitions."；"When a `<NuxtLink>` enters the viewport on the client side, Nuxt will automatically prefetch components and payload (generated pages) of the linked pages ahead of time, resulting in faster navigation."
  - 原文示例：`<NuxtLink to="/about">About</NuxtLink>`、`<NuxtLink to="/posts/1">Post 1</NuxtLink>`、`<NuxtLink to="/posts/2">Post 2</NuxtLink>`
  - 原文（Route Parameters）："The `useRoute()` composable can be used in a `<script setup>` block or a `setup()` method of a Vue component to access the current route details."；示例 `const route = useRoute()` 与 `// When accessing /posts/1, route.params.id will be 1`、`console.log(route.params.id)`
  - 原文（Route Middleware）："Nuxt provides a customizable route middleware framework you can use throughout your application, ideal for extracting code that you want to run before navigating to a particular route."；（note）"Route middleware runs within the Vue part of your Nuxt app. Despite the similar name, they are completely different from server middleware, which are run in the Nitro server part of your app."；（important）"Route middleware does **not** run for server routes (e.g. `/api/*`) or other server requests. To apply middleware to these requests, use server middleware instead."
  - 原文三种中间件："Anonymous (or inline) route middleware, which are defined directly in the pages where they are used."；"Named route middleware, which are placed in the `app/middleware/` directory and will be automatically loaded via asynchronous import when used on a page. (**Note**: The route middleware name is normalized to kebab-case, so `someMiddleware` becomes `some-middleware`.)"；"Global route middleware, which are placed in the `app/middleware/` directory (with a `.global` suffix) and will be automatically run on every route change."
  - 原文示例：`export default defineNuxtRouteMiddleware((to, from) => { ... return navigateTo('/login') })`（注释 `// isAuthenticated() is an example method verifying if a user is authenticated`）与 `definePageMeta({ middleware: 'auth' })`
  - 原文（Route Validation）："Nuxt offers route validation via the `validate` property in `definePageMeta()` in each page you wish to validate."；"The `validate` property accepts the `route` as an argument. You can return a boolean value to determine whether or not this is a valid route to be rendered with this page. If you return `false`, this will cause a 404 error. You can also directly return an object with `status`/`statusText` to customize the error returned."；"If you have a more complex use case, then you can use anonymous route middleware instead."；示例 `validate (route) { // Check if the id is made up of digits; return typeof route.params.id === 'string' && /^\d+$/.test(route.params.id) }`
- `https://nuxt.com/docs/4.x/directory-structure/app/pages`
  - 原文："The `app/pages/index.vue` file will be mapped to the `/` route of your application."；"If you are using `app.vue`, make sure to use the `<NuxtPage/>` component to display the current page"
- `https://nuxt.com/docs/4.x/guide/concepts/rendering` 与 `https://nuxt.com/docs/4.x/api/components/nuxt-page`
  - 对照用（见 091、103）：水合定义；"the new page component is mounted **before** the previous one is unmounted"
- `https://nuxt.com/docs/4.x/guide/concepts/server-components`
  - 对照用（见 106）："When a `<NuxtLink>` to a server-only page is prefetched, its island response is prefetched too"
