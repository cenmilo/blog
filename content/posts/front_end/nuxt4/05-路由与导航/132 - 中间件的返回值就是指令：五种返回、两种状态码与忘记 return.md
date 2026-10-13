---
url: /posts/753f00d317a3ee7e31738049e91e21cd/
title: 132 - 中间件的返回值就是指令：五种返回、两种状态码与忘记 return
date: 2026-11-25T09:00:00+08:00
updated: 2026-11-25T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇把中间件的输出面收口。第一条是结构差异：**"与 vue-router 的导航守卫不同，不会传入第三个 `next()` 参数，**重定向或取消导航是由中间件的返回值来处理的**。**"第二条是那张五种返回值的官方清单：什么都不返回（**简单 `return` 或干脆不返回**）——"**不阻塞导航，会走向下一个中间件函数（如果有），或完成这次路由导航**"；`return navigateTo('/')`——"**重定向到给定路径，如果重定向发生在服务端会把重定向码设为 `302 Found`**"；`return navigateTo('/', { redirectCode: 301 })`——"**重定向到给定路径，如果重定向发生在服务端会把重定向码设为 `301 Moved Permanently`**"；`return abortNavigation()`——"**停止当前导航**"；`return abortNavigation(error)`——"**用一个错误拒绝当前导航**"。第三条是官方那条 important：**"我们建议使用上面的 helper 函数来执行重定向或停止导航。vue-router 文档里描述的其他可能返回值也许有效，但未来可能会有破坏性变更。"** 第四条与调用姿势有关，官方在 `navigateTo` 那一节给了一个 ❌ 反例：在中间件里只写 `navigateTo('/search', { redirectCode: 301 })` 然后 `return`，官方的判定是"**这种情况下 `navigateTo` 会被执行但不会被返回，这可能导致意外的行为**"，并把规则写成"**在路由中间件里使用 `navigateTo` 时，你必须返回它的结果，以确保中间件执行流程正确工作**"。第五条是状态码的作用域：`redirectCode` 官方明确限定在"**当重定向发生在服务端时**"才生效，默认 `302`，"`301 Moved Permanently` 可以用于永久重定向"；而 `defineNuxtRouteMiddleware` 的返回类型正是 `ReturnType<NavigationGuard>`，所以"返回什么"在类型层面就已经是导航守卫的语义。另外两个输出动作不在返回值里：**抛错误页**（官方示例 `throw createError({ status: 404, statusText: 'Page Not Found' })`，"上面的路由中间件会把用户重定向到在 `~/error.vue` 文件里定义的自定义错误页，并把从中间件传过去的错误信息与状态码暴露出来"）与**在中间件里做条件判断时的 `useError`**（官方 note："渲染错误页是一次完全独立的页面加载，这意味着任何已注册的中间件都会再运行一次。你可以在中间件里使用 `useError` 来检查是否正在处理一个错误。"）。最后一条顺序前提：中间件先跑全局（全局之间默认按文件名字母序），再按页面 `middleware` 数组声明的顺序跑，而父子路由的 middleware 是**合并**关系。

categories:
  - 前端开发

tags:
  - nuxt4
  - 中间件
  - 导航守卫
  - 重定向
  - 状态码

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


中间件里最常见的 bug 不是判断写错，而是**该 return 的时候没 return**。这一篇把"返回什么 = 发生什么"排成一张可以贴在显示器旁边的表。

## 为什么是返回值：没有 `next()`

官方给的第一句就改了习惯：

> **与 vue-router 的导航守卫不同，不会传入第三个 `next()` 参数，重定向或取消导航是由中间件的返回值来处理的。**

```ts
// vue-router 守卫的老写法——在 Nuxt 中间件里不存在
export default defineNuxtRouteMiddleware((to, from, next) => { next() }) // ❌
```

对应到类型上，官方在 `defineNuxtRouteMiddleware` 那一节写的接口就是：

```ts
interface RouteMiddleware {
  (to: RouteLocationNormalized, from: RouteLocationNormalized): ReturnType<NavigationGuard>
}
```

**只有两个参数，返回值承担全部指令**。这就是"忘记 return"为什么必然出错的原因：没有别的通道表达"我要改道"。

## 五种返回值：官方清单逐项

| 你写的 | 官方描述的效果 | 用户看到 |
| --- | --- | --- |
| **什么都不返回**（简单 `return` 或干脆不返回） | "**不阻塞导航，会走向下一个中间件函数（如果有），或完成这次路由导航**" | 正常进入目标页 |
| `return navigateTo('/')` | "**重定向到给定路径，如果重定向发生在服务端会把重定向码设为 `302 Found`**" | 落到别的页面 |
| `return navigateTo('/', { redirectCode: 301 })` | "**重定向到给定路径，如果重定向发生在服务端会把重定向码设为 `301 Moved Permanently`**" | 同上，且是永久码 |
| `return abortNavigation()` | "**停止当前导航**" | **页面纹丝不动** |
| `return abortNavigation(error)` | "**用一个错误拒绝当前导航**" | 留在原地，但有一个错误被抛出 |

这张表里有三个读法值得单拎：

1. **`return` 与不 `return` 不是"有没有返回值"的差别，而是"放行 / 改道"两种指令。** 什么都不返回是**合法的放行**——它不等于漏写。
2. **`abortNavigation()` 与"什么都不返回"完全不同**：前者停止这次导航（用户留在原页），后者让导航继续。
3. **两种 `abortNavigation` 的差别在错误**："停止"与"用一个错误拒绝"，官方给的是两句不同的话。

## 最重要的那条：必须 return

官方在 `navigateTo` 那一节把规则与反例都写全了。

规则句：**"在路由中间件里使用 `navigateTo` 时，你必须返回它的结果，以确保中间件执行流程正确工作。"**

反例（官方原样标注 ❌）：

```ts
export default defineNuxtRouteMiddleware((to, from) => {
  if (to.path !== '/search') {
    // ❌ This will not work as expected
    navigateTo('/search', { redirectCode: 301 })
    return
  }
})
```

> 这种情况下 `navigateTo` 会被执行但不会被返回，这可能导致意外的行为。

这段的杀伤力在于**它的表现很像"成功"**：`navigateTo` 确实被调用了，但中间件交给导航流程的返回值是 `undefined`——按上面那张表，那一条的语义是**放行**。于是"你想改道，Nuxt 读到的是不阻塞"。

对照正确写法：

```ts
export default defineNuxtRouteMiddleware((to, from) => {
  if (to.path !== '/search') {
    // setting the redirect code to '301 Moved Permanently'
    return navigateTo('/search', { redirectCode: 301 })
  }
})
```

顺带一条同族规则，来自 `navigateTo` 使用那节：**"务必在调用它时对它的结果使用 `await` 或 `return`。"** 在中间件里，能满足这一条的只有 `return`。

| 位置 | 该用 |
| --- | --- |
| 组件 / 组合式函数 / 插件 | `await navigateTo(...)` |
| 路由中间件 | **`return navigateTo(...)`** |

## `redirectCode` 只在服务端那一趟有意义

`redirectCode` 的官方定义带了一个明确的限定条件："**`navigateTo` 会重定向到给定路径，并且当重定向发生在服务端时，默认把重定向码设为 `302 Found`。**"

再连上 128 那条"初始页面的中间件会跑两趟（渲染时一次、客户端一次）"：

| 那一趟 | `redirectCode` 有没有意义 |
| --- | --- |
| 服务端渲染（或生成）时 | **有**——这是真正写进 HTTP 响应的那次 |
| 客户端导航 / 水合后那一趟 | 没有 HTTP 状态码可言 |

所以 301 与 302 的选择影响的是**服务端响应与缓存**，而不是页面上看到的差别：

| 码 | 官方的推荐语气 |
| --- | --- |
| `302 Found` | 默认 |
| `301 Moved Permanently` | "**通常，301 Moved Permanently 可以用于永久重定向**" |

## 两种不在返回值表里的输出动作

### 抛一个错误页

`throw` 不是"返回"，但它是最常用的第三种结局。官方示例：

```ts [app/middleware/error.ts]
export default defineNuxtRouteMiddleware((to) => {
  if (to.params.id === '1') {
    throw createError({ status: 404, statusText: 'Page Not Found' })
  }
})
```

> 上面的路由中间件会把用户重定向到在 `~/error.vue` 文件里定义的自定义错误页，并把从中间件传过去的错误信息与状态码暴露出来。

**三个结局对照，选型就看这张表**：

| 想要的结局 | 写法 |
| --- | --- |
| 让他进目标页 | 什么都不返回 |
| 让他去别的页 | `return navigateTo('/login')` |
| 这次导航作废 | `return abortNavigation()` |
| 作废并抛出一个错误 | `return abortNavigation(error)` |
| 给他一个错误页 | `throw createError({ status: 404, statusText })` |

（另外别忘了 111 那条：`statusText` 只允许水平制表符、空格与可见 ASCII，中文要放 `message`。）

### 在中间件里判断"这是不是一次错误页加载"

官方那条 note：**"渲染错误页是一次完全独立的页面加载，这意味着任何已注册的中间件都会再运行一次。你可以在中间件里使用 `useError` 来检查是否正在处理一个错误。"**

配合 128 的另一句（即使你在服务端的中间件里抛出错误并渲染了错误页，中间件仍会在浏览器里再跑一次），这条判断就成了避免"错误页上又触发一次重定向"的手段。

## 顺序与合并：决定谁先返回

返回值语义要以执行链为前提，官方的两档顺序是：

```
① 全局中间件（之间默认按文件名字母序；要控制就用 01./02. 编号前缀）
② 页面定义的中间件顺序（数组声明的顺序即执行顺序）
```

再加一条：`middleware` 字段"**它会与任何匹配的父/子路由里用到的所有其他中间件合并**"。

也就是说：**排在前面的那个中间件一旦 `return navigateTo(...)` 或 `return abortNavigation(...)`，后面的就没机会按原计划把用户送到它想去的地方**——`abortNavigation()` 尤其要放在最前，否则用户会先经历一遍后面那些中间件的副作用。

## 还有一条官方限制

important 那句是给你"想走捷径"时的护栏：

> **我们建议使用上面的 helper 函数来执行重定向或停止导航。vue-router 文档里描述的其他可能返回值也许有效，但未来可能会有破坏性变更。**

所以别把 vue-router 守卫里那些"返回 `false`、返回一个 `RouteLocationRaw`、混合写法"当成可依赖的接口——**要改道就用 `navigateTo`，要作废就用 `abortNavigation`**，这两个是官方在这个上下文里点名的全局 helper（130）。

## 自查

1. Nuxt 中间件与 vue-router 导航守卫在"怎么表达指令"上差在哪？类型上如何体现？
2. 五种返回值分别是什么？"什么都不返回"是漏写还是合法放行？
3. `abortNavigation()` 与"什么都不返回"的用户可见结果差别是什么？
4. 官方的 ❌ 反例为什么危险？它的表现为什么像成功？
5. `redirectCode` 在哪一趟才有意义？302 与 301 分别按官方语气用在什么场合？
6. 要给错误页而不是留在原地，应该写什么？中间件里怎么判断"这是错误页那一次加载"？
7. 为什么 `abortNavigation()` 这种返回要格外注意它在链上的位置？

## 下一篇

133 讲另一道门：路由中间件保护不到 `/api/*`，服务端中间件才管得到接口——两条链的位置、能力与"为什么前端权限判断不能当安全边界"。

## 本文依据

- `https://nuxt.com/docs/4.x/directory-structure/app/middleware`
  - 原文（无 next）："Unlike navigation guards from `vue-router`, a third `next()` argument is not passed, and **redirect or route cancellation is handled by returning a value from the middleware**."
  - 原文（五种返回值）："nothing (a simple `return` or no return at all) - does not block navigation and will move to the next middleware function, if any, or complete the route navigation"；"`return navigateTo('/')` - redirects to the given path and will set the redirect code to `302 Found` if the redirect happens on the server side"；"`return navigateTo('/', { redirectCode: 301 })` - redirects to the given path and will set the redirect code to `301 Moved Permanently` if the redirect happens on the server side"；"`return abortNavigation()` - stops the current navigation"；"`return abortNavigation(error)` - rejects the current navigation with an error"
  - 原文（important）："We recommend using the helper functions above for performing redirects or stopping navigation. Other possible return values described in the vue-router docs may work but there may be breaking changes in future."
  - 原文（Usage 示例）：`if (to.params.id === '1') { return abortNavigation() }` 与 `if (to.path !== '/') { return navigateTo('/') }`，注释 "it is important to check `to.path` before redirecting or you might get an infinite redirect loop"
  - 原文（两个 helper）："Nuxt provides two globally available helpers that can be returned directly from the middleware. 1. `navigateTo` - Redirects to the given route 2. `abortNavigation` - Aborts the navigation, with an optional error message."
  - 原文（Middleware Order / Ordering Global Middleware）："Middleware runs in the following order: 1. Global Middleware 2. Page defined middleware order..."；"By default, global middleware is executed alphabetically based on the filename."；编号前缀示例与 "filenames are sorted as strings, not as numeric values"
  - 原文（When Middleware Runs）："middleware for the initial page will be executed both when the page is rendered and then again on the client"；"This is true even if you throw an error in your middleware on the server, and an error page is rendered. The middleware will still run again in the browser."；（note）"Rendering an error page is an entirely separate page load, meaning any registered middleware will run again. You can use `useError` in middleware to check if an error is being handled."
  - 原文（middleware 字段合并）："You can define middleware to apply before loading this page. It will be merged with all the other middleware used in any matching parent/child routes."
- `https://nuxt.com/docs/4.x/api/utils/navigate-to`
  - 原文（必须 return）："When using `navigateTo` within route middleware, you must **return its result** to ensure the middleware execution flow works correctly."；反例 "// ❌ This will not work as expected" + `navigateTo('/search', { redirectCode: 301 })` 后 `return`；"In this case, `navigateTo` will be executed but not returned, which may lead to unexpected behavior."
  - 原文（warning）："Make sure to always use `await` or `return` on result of `navigateTo` when calling it."
  - 原文（`redirectCode`）："`navigateTo` redirects to the given path and sets the redirect code to `302 Found` by default **when the redirection takes place on the server side**. This default behavior can be modified by providing different `redirectCode`. Commonly, `301 Moved Permanently` can be used for permanent redirections."
- `https://nuxt.com/docs/4.x/api/utils/abort-navigation`
  - 原文："abortNavigation is a helper function that prevents navigation from taking place and throws an error if one is set as a parameter."；（warning）"`abortNavigation` is only usable inside a route middleware handler."；Type `export function abortNavigation (err?: Error | string): false`；示例 `return abortNavigation('Insufficient permissions.')`、`catch (err) { return abortNavigation(err) }`
- `https://nuxt.com/docs/4.x/api/utils/define-nuxt-route-middleware`
  - 原文（接口）：`interface RouteMiddleware { (to: RouteLocationNormalized, from: RouteLocationNormalized): ReturnType<NavigationGuard> }`
  - 原文（抛错误页）：`throw createError({ status: 404, statusText: 'Page Not Found' })`；"The above route middleware will redirect a user to the custom error page defined in the `~/error.vue` file, and expose the error message and code passed from the middleware."
- `https://nuxt.com/docs/4.x/getting-started/error-handling`
  - 对照用（见 111）：`statusText` 只允许水平制表符、空格与可见 ASCII 字符（`[\t\u0020-\u007E]`），详细描述与非 ASCII 内容改用 `message`
