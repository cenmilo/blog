---
url: /posts/52cd5db953d719110e34510f7daf9edd/
title: 两个 addRouteMiddleware：插件里注册函数、模块里登记文件
date: 2026-11-24T11:00:00+08:00
updated: 2026-11-24T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：文档里有**两个**叫 `addRouteMiddleware` 的函数，同名、不同来源、传的东西也不同，选错了会得到"注册了但没生效"或"模块里传了个函数"。**第一个是运行时的**（`api/utils/add-route-middleware`，官方描述为"**`addRouteMiddleware()` 是一个 helper 函数，用于在你的应用里动态添加中间件**"），签名是两条重载：`function addRouteMiddleware (name: string, middleware: RouteMiddleware, options?: AddRouteMiddlewareOptions): void` 与 `function addRouteMiddleware (middleware: RouteMiddleware): void`，其中 `interface AddRouteMiddlewareOptions { global?: boolean }`；参数 `name` 的类型是 `string | RouteMiddleware`（"**可以是一个字符串，或一个类型为 `RouteMiddleware` 的函数。函数接收下一个路由 `to` 作为第一个参数、当前路由 `from` 作为第二个参数，两者都是 Vue 路由对象**"），`middleware` 那项官方写明"**如果 `addRouteMiddleware()` 的第一个参数已经传的是函数，它就变成可选的**"，`options` 则"**让你把 `global` 的值设为 `true`，用来表明这个路由中间件是否是全局的（默认设为 `false`）**"。它的官方用法全部写在插件里，并且有一条覆盖规则："**当在插件里定义时，它会覆盖 `app/middleware/` 目录里任何同名的既有中间件。**"**第二个是构建期的**（`api/kit/pages`，官方的定位句是："**注册路由中间件，使其可用于所有路由或特定路由。**"，并补一句"**路由中间件也可以通过插件里的 `addRouteMiddleware` 组合式函数来定义**"以区分两者），签名是 `function addRouteMiddleware (input: NuxtMiddleware | NuxtMiddleware[], options?: AddRouteMiddlewareOptions): void`，`input` 是"**一个中间件对象，或一个中间件对象数组**"，三个属性 `name`（`string`，必需，中间件的名字）、`path`（`string`，必需，"**中间件的文件路径**"）、`global`（`boolean`，非必需，"**设为 `true` 时把中间件应用到所有路由**"），`options` 里是 `override`（`boolean`，默认 `false`，"**若为 `true`，替换同名的中间件**"）与 `prepend`（`boolean`，默认 `false`，"**若为 `true`，把中间件前置到既有中间件之前**"）。官方给的用法示例是一个模块 + 一个运行时文件成对出现的。

categories:
  - 前端开发

tags:
  - nuxt4
  - 中间件
  - 插件
  - 模块
  - kit

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


搜索 `addRouteMiddleware` 会出来两页文档，签名却对不上——一个是 `(name, 函数, 选项)`，一个是 `({ name, path })`。这不是笔误，是**两个不同的函数**。

## 一张表先分清

| | 运行时的那个 | 构建期的那个 |
| --- | --- | --- |
| 文档位置 | `api/utils/add-route-middleware` | `api/kit/pages` |
| 谁在用 | **应用代码**（官方示例全在插件里） | **模块作者**（`defineNuxtModule` 的 `setup` 里） |
| 注册的是 | 一个**函数** | 一个**文件路径**（`path`） |
| 官方定位句 | "`addRouteMiddleware()` 是一个 helper 函数，用于在你的应用里**动态添加**中间件。" | "**注册路由中间件**，使其可用于所有路由或特定路由。" |
| `global` 怎么给 | 第三个参数 `{ global: true }`，或首参直接传函数 | 对象里的 `global` 属性 |
| 顺序控制 | 无（只有"同名覆盖"这一条规则） | `override` 与 `prepend` 两个开关 |
| 构建期文档里怎么提到它 | —— | 官方补了一句："**路由中间件也可以通过插件里的 `addRouteMiddleware` 组合式函数来定义。**" |

最后一行就是官方给这两页互相指认的方式。

## 运行时：插件里注册一个函数

签名是两条重载：

```ts
function addRouteMiddleware (name: string, middleware: RouteMiddleware, options?: AddRouteMiddlewareOptions): void
function addRouteMiddleware (middleware: RouteMiddleware): void

interface AddRouteMiddlewareOptions {
  global?: boolean
}
```

参数官方逐条：

| 参数 | 官方说明 |
| --- | --- |
| `name` | 类型 `string \| RouteMiddleware`；"**可以是一个字符串，或一个类型为 `RouteMiddleware` 的函数。函数接收下一个路由 `to` 作为第一个参数、当前路由 `from` 作为第二个参数，两者都是 Vue 路由对象。**" |
| `middleware` | 类型 `RouteMiddleware`；"**第二个参数是一个类型为 `RouteMiddleware` 的函数……如果 `addRouteMiddleware()` 的第一个参数已经传的是函数，它就变成可选的。**" |
| `options` | "**一个可选的 `options` 参数，让你把 `global` 的值设为 `true`，用来表明这个路由中间件是否是全局的（默认设为 `false`）。**" |

### 命名注册

官方的写法与场景说明：

> 命名路由中间件通过把**字符串作为第一个参数、函数作为第二个参数**来定义：

```ts [app/plugins/my-plugin.ts]
export default defineNuxtPlugin(() => {
  addRouteMiddleware('named-middleware', () => {
    console.log('named middleware added in Nuxt plugin')
  })
})
```

> **当在插件里定义时，它会覆盖 `app/middleware/` 目录里任何同名的既有中间件。**

这条覆盖规则是**默认行为**，没有开关——想接管一个已存在的名字，注册同名即可。

### 全局注册的两种写法

官方原文："全局路由中间件可以用两种方式定义：把函数**直接作为第一个参数、不带名字**传入。**它会被自动视为全局中间件，并在每一次路由变化时应用。**"

```ts [app/plugins/my-plugin.ts]
export default defineNuxtPlugin(() => {
  addRouteMiddleware((to, from) => {
    console.log('anonymous global middleware that runs on every route change')
  })
})
```

"**或者设置一个可选的第三个参数 `{ global: true }` 来表明该路由中间件是全局的。**"

```ts [app/plugins/my-plugin.ts]
export default defineNuxtPlugin(() => {
  addRouteMiddleware('global-middleware', (to, from) => {
    console.log('global middleware that runs on every route change')
  },
  { global: true },
  )
})
```

中间件那一页给的插件示例，正是这两条的并排演示（一条 `global-test` 带 `{ global: true }`、一条 `named-test`，后者注释写着"会覆盖任何同名既有中间件"，见 129）。

## 构建期：模块里登记一个文件

Kit 版本的签名传的是对象：

```ts
function addRouteMiddleware (input: NuxtMiddleware | NuxtMiddleware[], options?: AddRouteMiddlewareOptions): void
```

官方的 `input` 说明："**一个中间件对象，或一个中间件对象数组**"，属性三项：

| 属性 | 类型 | 必需 | 官方说明 |
| --- | --- | --- | --- |
| `name` | `string` | **是** | 中间件的名字 |
| `path` | `string` | **是** | **中间件的文件路径** |
| `global` | `boolean` | 否 | **设为 `true` 时把中间件应用到所有路由** |

`options` 两项：

| 属性 | 类型 | 默认 | 官方说明 |
| --- | --- | --- | --- |
| `override` | `boolean` | `false` | **若为 `true`，替换同名的中间件** |
| `prepend` | `boolean` | `false` | **若为 `true`，把中间件前置到既有中间件之前** |

官方的示例是"模块 + 中间件文件"成对给出：

```ts [module.ts]
import { addRouteMiddleware, createResolver, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  setup () {
    const { resolve } = createResolver(import.meta.url)

    addRouteMiddleware({
      name: 'auth',
      path: resolve('runtime/auth'),
      global: true,
    }, { prepend: true })
  },
})
```

```ts [runtime/auth.ts]
export default defineNuxtRouteMiddleware((to, from) => {
  // isAuthenticated() is an example method verifying if a user is authenticated
  if (to.path !== '/login' && isAuthenticated() === false) {
    return navigateTo('/login')
  }
})
```

看这两段就能明白两者的分工：**构建期登记的是"文件在哪、叫什么、是不是全局"**，被登记的那个文件本身仍然是用 `defineNuxtRouteMiddleware` 写的普通中间件（130）。

## 还有第三种：构建期改页面元数据

如果目标不是"注册一个中间件"，而是"给某些页面挂上中间件"，官方在中间件那一页给的是 `pages:extend` 钩子里改 `page.meta.middleware`，并且注了一句它会**覆盖**页面里 `definePageMeta` 的设置（129）。

Kit 的 `extendPages` 也能改这一层——它的 `meta` 属性官方说明是"**路由的自定义元数据。可以用在布局、中间件或导航守卫里。**"类型是 `NuxtPageMeta`。要给自定义字段类型安全，官方给的是增强 `@nuxt/schema` 里的 `NuxtPageMeta`，示例末尾同样带 `export {}`：

```ts [index.d.ts]
declare module '@nuxt/schema' {
  interface NuxtPageMeta {
    requiresAuth?: boolean
  }
}

export {}
```

## 该用哪一个

| 你的场景 | 用 |
| --- | --- |
| 应用代码里按条件加一条中间件（例如登录后再注册） | 运行时版，写在插件里 |
| 模块要随包发一个中间件文件 | 构建期 Kit 版，`{ name, path, global }` |
| 模块要抢在既有中间件前面跑 | 构建期版加 `{ prepend: true }` |
| 模块要替换同名的一个 | 构建期版加 `{ override: true }`（运行时版则是**天然覆盖**） |
| 给一批页面统一挂某个中间件 | `pages:extend` 或 `extendPages` 改 `meta.middleware` |

## 一个共同的坑：注册位置不等于执行上下文

两个函数都只是"登记"。**运行时那一个写在插件里，不代表你能在插件体里直接调用 `abortNavigation`**——官方错误码 E2003 说得很清楚：`abortNavigation()` 只能从中间件处理器内部取消导航，"**从一个组件、插件、组合式函数，或一个已经失去中间件上下文的回调里调用都会失败**"，解决方式是"把这个调用移进 `defineNuxtRouteMiddleware()` 处理器的函数体里"。

同理 E2005：中间件里（直接或间接）调用 `useRoute()` 会拿到意外值，**要用作为参数收到的 `to` / `from`**（128、130）。

**换句话说：注册在哪都行，但中间件函数体里能做什么，取决于它被当作中间件执行的那一刻。**

## 自查

1. 两个 `addRouteMiddleware` 分别来自哪两份文档？各自传的是函数还是文件路径？
2. 运行时版本里，"不带名字的函数作为第一个参数"会产生什么效果？另一种全局写法是什么？
3. 运行时注册的同名中间件，官方说会发生什么？需要开关吗？
4. 构建期版本的 `input` 三个属性里，哪两个是必需的？`override` 与 `prepend` 各默认什么、各改变什么？
5. 想让模块注册的中间件排在最前，应该用哪个开关？运行时版有这个能力吗？
6. 给一批页面挂中间件，官方给的两种构建期手段是什么？其中哪一种会覆盖 `definePageMeta`？
7. "在插件里注册中间件"能不能顺手在插件里调用 `abortNavigation()`？依据是哪条错误码？

## 下一篇

132 把中间件的返回值一次讲透：为什么没有 `next()`、五种返回值分别发生什么、`302` 与 `301` 什么时候才用得上，以及"忘记 return"为什么是最常见的一种 bug。

## 本文依据

- `https://nuxt.com/docs/4.x/api/utils/add-route-middleware`
  - 原文（描述）："addRouteMiddleware() is a helper function to dynamically add middleware in your application."
  - 原文（note）："Route middleware are navigation guards stored in the `app/middleware/` directory of your Nuxt application (unless set otherwise)."
  - 原文（Type）：`function addRouteMiddleware (name: string, middleware: RouteMiddleware, options?: AddRouteMiddlewareOptions): void`；`function addRouteMiddleware (middleware: RouteMiddleware): void`；`interface AddRouteMiddlewareOptions { global?: boolean }`
  - 原文（`name`）："**Type:** `string | RouteMiddleware`"；"Can be either a string or a function of type `RouteMiddleware`. Function takes the next route `to` as the first argument and the current route `from` as the second argument, both of which are Vue route objects."
  - 原文（`middleware`）："The second argument is a function of type `RouteMiddleware`. Same as above, it provides `to` and `from` route objects. It becomes optional if the first argument in `addRouteMiddleware()` is already passed as a function."
  - 原文（`options`）："An optional `options` argument lets you set the value of `global` to `true` to indicate whether the router middleware is global or not (set to `false` by default)."
  - 原文（Named Route Middleware）："Named route middleware is defined by providing a string as the first argument and a function as the second"；示例 `addRouteMiddleware('named-middleware', () => { console.log('named middleware added in Nuxt plugin') })`；"When defined in a plugin, it overrides any existing middleware of the same name located in the `app/middleware/` directory."
  - 原文（Global Route Middleware）："Pass a function directly as the first argument without a name. It will automatically be treated as global middleware and applied on every route change."；示例 `addRouteMiddleware((to, from) => { console.log('anonymous global middleware that runs on every route change') })`；"Set an optional, third argument `{ global: true }` to indicate whether the route middleware is global."；示例 `addRouteMiddleware('global-middleware', (to, from) => {...}, { global: true })`
- `https://nuxt.com/docs/4.x/api/kit/pages`
  - 原文（`addRouteMiddleware`）："Registers route middlewares to be available for all routes or for specific routes."；"Route middlewares can be also defined in plugins via `addRouteMiddleware` composable."；Type `function addRouteMiddleware (input: NuxtMiddleware | NuxtMiddleware[], options?: AddRouteMiddlewareOptions): void`；input 表 `name`/`path`/`global` 三项（`name` Required `true` "The name of the middleware."、`path` Required `true` "The file path to the middleware."、`global` Required `false` "If set to `true`, applies middleware to all routes."）；options 表 `override`（`boolean`，default `false`，"If `true`, replaces middleware with the same name."）与 `prepend`（`boolean`，default `false`，"If `true`, prepends middleware before existing middlewares."）；示例 `addRouteMiddleware({ name: 'auth', path: resolve('runtime/auth'), global: true }, { prepend: true })` 与 `runtime/auth.ts` 里 `export default defineNuxtRouteMiddleware((to, from) => { if (to.path !== '/login' && isAuthenticated() === false) { return navigateTo('/login') } })`；Source `packages/kit/src/pages.ts`
  - 原文（`extendPages`）：`meta` — "Custom metadata for the route. Can be used in layouts, middlewares, or navigation guards."；类型 `NuxtPageMeta`；"You should modify the provided pages array directly, as changes made to a copied array will not be reflected in the configuration."；Typing Custom Page Metadata 示例 `declare module '@nuxt/schema' { interface NuxtPageMeta { requiresAuth?: boolean } }` + `export {}`
- `https://nuxt.com/docs/4.x/directory-structure/app/middleware`
  - 对照用（见 128、129）：插件示例 `addRouteMiddleware('global-test', ..., { global: true })` 与 `addRouteMiddleware('named-test', ...)`（注释 "would override any existing middleware of the same name"）；`pages:extend` 示例及注释 "Note that this will override any middleware set in `definePageMeta` in the page"
- `https://nuxt.com/docs/4.x/api/utils/define-nuxt-route-middleware`
  - 对照用（见 130）：被登记的那个文件仍是 `defineNuxtRouteMiddleware` 写的中间件
- `https://nuxt.com/docs/4.x/errors/e2003` 与 `https://nuxt.com/docs/4.x/errors/e2005`
  - 原文（E2003）："`abortNavigation()` was called outside a route middleware. It can only cancel navigation from within a middleware handler, so calling it from a component, plugin, composable, or a callback that lost the middleware context fails."；Resolution："Move the `abortNavigation()` call into the body of a `defineNuxtRouteMiddleware()` handler."
  - 原文（E2005）："`useRoute()` was called inside a route middleware (directly or via another composable)... middleware receives the `to` and `from` routes as arguments instead."
