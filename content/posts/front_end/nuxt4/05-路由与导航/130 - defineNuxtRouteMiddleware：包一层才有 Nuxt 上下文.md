---
url: /posts/2949441a5fb04f554610b5f9b0f676e3/
title: defineNuxtRouteMiddleware：包一层才有 Nuxt 上下文
date: 2026-11-24T10:00:00+08:00
updated: 2026-11-24T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：`defineNuxtRouteMiddleware` 的官方用途是"**用 `defineNuxtRouteMiddleware` helper 函数来创建命名路由中间件**"，并紧接着说明位置："路由中间件存放在你 Nuxt 应用的 `app/middleware/` 里（除非另行设置）。"它的签名把"包一层"这件事写在类型里：`export function defineNuxtRouteMiddleware (middleware: RouteMiddleware): RouteMiddleware`，即**收一个中间件函数、返回一个中间件函数**；而 `interface RouteMiddleware { (to: RouteLocationNormalized, from: RouteLocationNormalized): ReturnType<NavigationGuard> }` 说明**参数是两个 Vue Router 的路由位置对象**——官方原文："**一个函数，它接收两个 Vue Router 的路由位置对象作为参数：下一个路由 `to` 作为第一个，当前路由 `from` 作为第二个。**"为什么要专门包这一层，答案在 NuxtApp 那一页：**"许多组合式函数与工具，无论是内置的还是用户自己写的，都可能需要访问 Nuxt 实例。这个实例并不是在你应用的任何地方都存在的，因为每一次请求都会创建一个全新的实例。"** 而官方给出的可访问范围里明确带着一个条件——"**目前，Nuxt 上下文只能在以下位置访问：插件、Nuxt 钩子、Nuxt 中间件（如果用 `defineNuxtRouteMiddleware` 包裹起来的话），以及 setup 函数（页面与组件中）。**"也就是说**"在中间件里"并不自动等于"有上下文"，包了这层宏才算**。官方还写了撞墙时的报错与出路："**如果一个组合式函数在被调用时无法访问上下文，你可能会得到一个错误，内容是 'A composable that requires access to the Nuxt instance was called outside of a plugin, Nuxt hook, Nuxt middleware, or Vue setup function.' 在这种情况下，你也可以通过 `nuxtApp.runWithContext` 显式地在这个上下文内调用函数。**"上下文之内能直接用哪两个 helper，官方在示例末尾点名："**`navigateTo` 与 `abortNavigation` 都是全局可用的 helper 函数，你可以在 `defineNuxtRouteMiddleware` 内部使用它们。**"两个官方示例分别演示"抛错误页"与"按登录状态重定向"，后者正是靠上下文才能用 `useState`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 中间件
  - nuxt-上下文
  - useState
  - runWithContext

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


中间件文件里那行 `export default defineNuxtRouteMiddleware(...)`，看着像只是写法约定。其实这一层包装决定了一件事：**你的中间件能不能用 `useState`、能不能拿到 Nuxt 实例。**

## 签名：收一个函数，返回一个函数

官方的用途句很朴素：

> **用 `defineNuxtRouteMiddleware` helper 函数来创建命名路由中间件。**
>
> 路由中间件存放在你 Nuxt 应用的 `app/middleware/` 里（除非另行设置）。

类型定义是这样的：

```ts
export function defineNuxtRouteMiddleware (middleware: RouteMiddleware): RouteMiddleware

interface RouteMiddleware {
  (to: RouteLocationNormalized, from: RouteLocationNormalized): ReturnType<NavigationGuard>
}
```

三个信息点：

| 部分 | 含义 |
| --- | --- |
| 入参 = 出参类型 | 它是个**包装器**：把一个普通函数登记成中间件，本身不改变你的逻辑 |
| `(to, from)` | 官方说明："**一个函数，它接收两个 Vue Router 的路由位置对象作为参数：下一个路由 `to` 作为第一个，当前路由 `from` 作为第二个。**" |
| 返回类型 `ReturnType<NavigationGuard>` | **中间件的返回值遵循导航守卫的返回语义**——这正是 132 那张"返回值就是指令"的类型来源 |

关于 `to`/`from` 这两个参数，官方在中间件那一页还写了为什么必须有它们：**中间件里没有"当前路由"的概念**，因为中间件可以中止一次导航或重定向到另一个路由（128）。**`to` 与 `from` 不是便利，而是唯一正确的取法。**

## 为什么要包这一层：上下文只在四个地方有

官方在 NuxtApp 那一页先解释了实例的来源：

> 许多组合式函数与工具，无论是内置的还是用户自己写的，**都可能需要访问 Nuxt 实例。这个实例并不是在你应用的任何地方都存在的，因为每一次请求都会创建一个全新的实例。**

然后是那份清单，注意它在"中间件"上加了一个括号条件：

> 目前，**Nuxt 上下文只能在以下位置访问：插件、Nuxt 钩子、Nuxt 中间件（如果用 `defineNuxtRouteMiddleware` 包裹起来的话），以及 setup 函数（页面与组件中）。**

**这一句就是本篇的全部答案**：不是"在中间件文件里"就有上下文，而是"**被这个宏包住的函数**"才在官方支持的范围内。

撞墙时官方给的报错原文与出路：

> 如果一个组合式函数在被调用时无法访问上下文，你可能会得到一个错误，内容是 **"A composable that requires access to the Nuxt instance was called outside of a plugin, Nuxt hook, Nuxt middleware, or Vue setup function."** 在这种情况下，你也可以通过 **`nuxtApp.runWithContext`** 显式地在这个上下文内调用函数。

| 你在中间件里做的事 | 需要上下文吗 | 靠什么满足 |
| --- | --- | --- |
| 读 `to.path`、`to.params`、`from.fullPath` | 不需要 | 函数参数（128、121） |
| `return navigateTo(...)` / `abortNavigation(...)` | 官方点名为"全局可用 helper" | 见下一节 |
| `useState('auth')` 这类需要实例的组合式函数 | **需要** | 用 `defineNuxtRouteMiddleware` 包住 |
| 调用一个内部用了 `useRoute()` 的组合式函数 | 不只是上下文问题 | **官方禁止在中间件里用 `useRoute()`**（128 那条 warning、错误码 E2005），要把 route 作为参数传进去 |
| 上下文仍然取不到时的兜底 | —— | `nuxtApp.runWithContext(() => ...)` |

## 包起来之后，官方点名的两个 helper

宏的示例那一节末尾原话：

> **`navigateTo` 与 `abortNavigation` 都是全局可用的 helper 函数，你可以在 `defineNuxtRouteMiddleware` 内部使用它们。**

中间件页那边也给了同一句话的另一种表述："**Nuxt 提供两个全局可用的 helper，它们可以直接从中间件里返回。**"

这两条与 127 的边界对上：**`abortNavigation` 只能用在路由中间件处理器里**——所以"被这个宏包住"既是拿到上下文的条件，也是使用这个 helper 的前提。写在组件里的 `abortNavigation()` 会撞上官方错误码 E2003："`abortNavigation()` was called outside a route middleware. It can only cancel navigation from within a middleware handler, so calling it from a component, plugin, composable, or a callback that lost the middleware context fails."（它的解决方式也正是"把这个调用移到 `defineNuxtRouteMiddleware()` 处理器的函数体里"。）

## 两个官方示例

### 抛一个错误页

官方的说明："**你可以使用路由中间件来抛出错误并显示有用的错误信息：**"

```ts [app/middleware/error.ts]
export default defineNuxtRouteMiddleware((to) => {
  if (to.params.id === '1') {
    throw createError({ status: 404, statusText: 'Page Not Found' })
  }
})
```

> 上面的路由中间件会把用户重定向到在 `~/error.vue` 文件里定义的自定义错误页，并把从中间件传过去的错误信息与状态码暴露出来。

这条与 127 的分工很值得对照记：**`return abortNavigation(error)` 是"这次导航作废并抛错"；`throw createError({ status: 404 })` 是"送用户去错误页"。**

### 按登录状态重定向

官方的说明："**把 `useState` 与 `navigateTo` helper 函数在路由中间件里结合使用，可以根据用户的认证状态把他们重定向到不同的路由：**"

```ts [app/middleware/auth.ts]
export default defineNuxtRouteMiddleware((to, from) => {
  const auth = useState('auth')

  if (!auth.value.isAuthenticated) {
    return navigateTo('/login')
  }

  if (to.path !== '/dashboard') {
    return navigateTo('/dashboard')
  }
})
```

这段就是"需要上下文"的具体样子：**`useState` 取的是这一次请求的 Nuxt 实例里的状态**。另外注意第二个判断——官方先比了 `to.path`，这与中间件页那句提醒一致：**重定向前要检查 `to.path`，否则可能得到无限重定向循环。**

## 与匿名（内联）中间件的写法差别

官方对三种中间件的分类里，第一种是"**匿名（或内联）路由中间件，直接定义在页面内部**"，示例写法是这样（**没有用这个宏**）：

```vue
<script setup lang="ts">
definePageMeta({
  middleware: [
    function (to, from) {
      // Custom inline middleware
    },
    'auth',
  ],
})
</script>
```

| | 命名中间件 | 匿名（内联）中间件 |
| --- | --- | --- |
| 写在哪 | `app/middleware/*.ts` | `definePageMeta({ middleware: [function (to, from) {}] })` |
| 是否用这个宏 | 官方这一页的用途就是"**创建命名路由中间件**" | 不需要，直接给函数 |
| 参数 | `to`、`from`（同 `RouteMiddleware`） | 同上 |
| 谁能引用它 | 任何页面按名字点 | 只有这一个页面 |

两者最终都进入同一条执行链（129 的顺序规则），返回值语义也相同（132）。

## 需要显式导入时

官方所有中间件示例里都没有为 `defineNuxtRouteMiddleware` 写 import——它是自动导入的。要把这种自动导入写成显式形式，官方的说法是"**Nuxt 用 `#imports` 别名暴露每一个自动导入，必要时可以用它把导入写成显式的**"（105 讲过这个说明符）。

## 自查

1. `defineNuxtRouteMiddleware` 的签名是什么？入参与返回类型有什么关系？
2. 参数 `to` 与 `from` 分别是什么？官方对它们的类型怎么描述？
3. 官方的 Nuxt 上下文清单里，"中间件"那一项带了一个什么条件？
4. 缺少上下文时官方给出的报错原文是什么？兜底手段是什么？
5. 宏内部官方点名的两个全局 helper 是哪两个？这与 `abortNavigation` 的使用范围有什么关系？
6. `throw createError({ status: 404 })` 与 `return abortNavigation(error)` 的区别是什么？
7. 为什么官方那个 `auth` 示例里第二个判断要先比 `to.path`？
8. 匿名（内联）中间件需要这个宏吗？

## 下一篇

131 把两个同名的 `addRouteMiddleware` 分开讲：插件里那一个注册的是函数、构建期 Kit 那一个注册的是文件路径，`global`、`override`、`prepend` 各自怎么写、各自影响什么。

## 本文依据

- `https://nuxt.com/docs/4.x/api/utils/define-nuxt-route-middleware`
  - 原文（描述）："Create named route middleware using defineNuxtRouteMiddleware helper function."
  - 原文："Route middleware are stored in the `app/middleware/` of your Nuxt application (unless set otherwise)."
  - 原文（Type）：`export function defineNuxtRouteMiddleware (middleware: RouteMiddleware): RouteMiddleware`；`interface RouteMiddleware { (to: RouteLocationNormalized, from: RouteLocationNormalized): ReturnType<NavigationGuard> }`
  - 原文（Parameters - `middleware`）："A function that takes two Vue Router's route location objects as parameters: the next route `to` as the first, and the current route `from` as the second."；"Learn more about available properties of `RouteLocationNormalized` in the Vue Router docs"
  - 原文（Showing Error Page）："You can use route middleware to throw errors and show helpful error messages:"；示例 `throw createError({ status: 404, statusText: 'Page Not Found' })`（文件标注 `app/middleware/error.ts`）；"The above route middleware will redirect a user to the custom error page defined in the `~/error.vue` file, and expose the error message and code passed from the middleware."
  - 原文（Redirection）："Use `useState` in combination with `navigateTo` helper function inside the route middleware to redirect users to different routes based on their authentication status:"；示例 `const auth = useState('auth')`、`if (!auth.value.isAuthenticated) { return navigateTo('/login') }`、`if (to.path !== '/dashboard') { return navigateTo('/dashboard') }`
  - 原文（结尾句）："Both `navigateTo` and `abortNavigation` are globally available helper functions that you can use inside `defineNuxtRouteMiddleware`."
  - 原文 Source：`packages/nuxt/src/app/composables/router.ts`
- `https://nuxt.com/docs/4.x/guide/going-further/nuxt-app`
  - 原文（描述）："In Nuxt, you can access runtime app context within composables, components and plugins."
  - 原文（The Nuxt Context）："Many composables and utilities, both built-in and user-made, may require access to the Nuxt instance. This doesn't exist everywhere on your application, because a fresh instance is created on every request."
  - 原文（范围清单）："Currently, the Nuxt context is only accessible in plugins, Nuxt hooks, **Nuxt middleware (if wrapped in `defineNuxtRouteMiddleware`)**, and setup functions (in pages and components)."
  - 原文（报错与兜底）："If a composable is called without access to the context, you may get an error stating that 'A composable that requires access to the Nuxt instance was called outside of a plugin, Nuxt hook, Nuxt middleware, or Vue setup function.' In that case, you can also explicitly call functions within this context by using `nuxtApp.runWithContext`."
  - 原文（read-more）："In Nuxt 2, this was referred to as **Nuxt context**."
- `https://nuxt.com/docs/4.x/directory-structure/app/middleware`
  - 对照用（见 128、129）："Nuxt provides two globally available helpers that can be returned directly from the middleware. 1. `navigateTo` - Redirects to the given route 2. `abortNavigation` - Aborts the navigation, with an optional error message."；内联中间件示例 `middleware: [function (to, from) {...}, 'auth']`；注释 "In a real app you would probably not redirect every route to `/` however it is important to check `to.path` before redirecting or you might get an infinite redirect loop"；"Always use the `to` and `from` parameters in your middleware... The `useRoute()` composable will always be inaccurate in this context."
- `https://nuxt.com/docs/4.x/errors/e2003`
  - 原文："`abortNavigation()` was called outside a route middleware. It can only cancel navigation from within a middleware handler, so calling it from a component, plugin, composable, or a callback that lost the middleware context fails."；Resolution："Move the `abortNavigation()` call into the body of a `defineNuxtRouteMiddleware()` handler."
- `https://nuxt.com/docs/4.x/errors/e2005`
  - 原文："`useRoute()` was called inside a route middleware (directly or via another composable). At that point the target route is not yet resolved, so `useRoute()` can return unexpected values; middleware receives the `to` and `from` routes as arguments instead."
- `https://nuxt.com/docs/4.x/guide/concepts/auto-imports`
  - 对照用（见 105）："Nuxt exposes every auto-import with the `#imports` alias that can be used to make the import explicit if needed"
