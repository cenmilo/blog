---
url: /posts/221bb4f7f269ee85a79bd9194f120f04/
title: navigateTo 为什么必须 await 或 return：三种位置的三种写法
date: 2026-11-22T10:00:00+08:00
updated: 2026-11-22T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：`navigateTo` 的官方定位是"一个程序化导航用户的 helper 函数"，而它取代 `router.push()` 的资本写在第一句里：**"`navigateTo` 在服务端与客户端都可用。它可以在 Nuxt 上下文内使用，也可以直接使用，来执行页面导航。"** 使用上有一条命令式规则：**"务必在调用它时对它的结果使用 `await` 或 `return`。"** 还有一条边界：**"`navigateTo` 不能在 Nitro 路由里使用。要在 Nitro 路由里做服务端重定向，请改用 `sendRedirect`。"** 三种落点的写法官方各给了一份：① 在 Vue 组件里，`to` 可以是字符串 `await navigateTo('/search')`、路由对象 `await navigateTo({ path: '/search' })`，或带查询参数的路由对象 `await navigateTo({ path: '/search', query: { page: 1, sort: 'asc' } })`；② 在路由中间件里，正例是 **`return navigateTo('/search', { redirectCode: 301 })`**，官方明确写"**在路由中间件里使用 `navigateTo` 时，你必须返回它的结果，以确保中间件执行流程正确工作**"，并给了一个标注 ❌ 的反例——只调用不返回（`navigateTo(...)` 之后 `return`），官方解释是"**这种情况下 `navigateTo` 会被执行但不会被返回，这可能导致意外的行为。**"；③ Nitro 路由里改用 `sendRedirect`。组件侧还有一条容易被忽略的行为，官方写成 note：**"调用 `navigateTo` 不会停止你 `<script setup>` 其余代码的执行。如果你希望一次成功的导航从 `setup()` 提前返回，请启用 `experimental.navigateToEarlyReturn`。"** 最后看它的签名，就能理解为什么"必须 await/return"不是风格问题：`navigateTo(to: RouteLocationRaw | undefined | null, options?: NavigateToOptions): Promise<void | NavigationFailure | false> | false | void | RouteLocationRaw`——**返回值形态随调用位置变化**（可能是 Promise、可能是 `false`、可能是裸的 `RouteLocationRaw`），丢掉它就等于丢掉"这次导航到底成没成"。

categories:
  - 前端开发

tags:
  - nuxt4
  - navigateTo
  - 编程式导航
  - 中间件
  - 重定向

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


上一篇说官方建议用 `navigateTo` 代替 `router.push()`。这一篇讲怎么用对——它的规则只有一条（必须 `await` 或 `return`），但三个位置里"对"的写法各不相同。

## 官方定位：为什么是它

> `navigateTo` 是一个程序化导航用户的 helper 函数。
>
> **`navigateTo` 在服务端与客户端都可用。它可以在 Nuxt 上下文内使用，也可以直接使用，来执行页面导航。**

这两句就是它胜过 `router.push()` 的地方：**服务端也有它**。`router.push()` 是浏览器里的历史栈操作，服务端渲染阶段没有"历史"；而服务端阶段的跳转恰恰是最需要写对的一类（未登录、URL 规范化、资源搬家）。

配套的一条边界：

> **`navigateTo` 不能在 Nitro 路由里使用。要在 Nitro 路由里做服务端重定向，请改用 `sendRedirect`。**

也就是说 `server/api/*`、`server/routes/*` 那些处理器里不要调它——那里是 h3 的地盘。

## 唯一的硬性规则

> **务必在调用它时对它的结果使用 `await` 或 `return`。**

为什么这不是风格问题，看签名就明白：

```ts
export function navigateTo (
  to: RouteLocationRaw | undefined | null,
  options?: NavigateToOptions,
): Promise<void | NavigationFailure | false> | false | void | RouteLocationRaw
```

**返回值的形态随调用位置变化**：可能是 Promise（要 await）、可能是 `false`（比如被 `abortNavigation` 那类语义取消时）、也可能是一个裸的 `RouteLocationRaw`（中间件里 `return` 出去，交给导航流程处理）。丢掉返回值，等于丢掉"这次导航成没成、以及该把它交给谁"。

## 位置一：Vue 组件里 —— 用 await

官方给的三种 `to` 形态：

```vue
<script setup lang="ts">
// passing 'to' as a string
await navigateTo('/search')

// ... or as a route object
await navigateTo({ path: '/search' })

// ... or as a route object with query parameters
await navigateTo({
  path: '/search',
  query: {
    page: 1,
    sort: 'asc',
  },
})
</script>
```

`to` 的完整口径在参数表里："**`to` 可以是一个普通字符串或一个要重定向到的路由对象。当传入 `undefined` 或 `null` 时，它会默认为 `'/'`。**"（类型 `RouteLocationRaw | undefined | null`，默认 `'/'`）——**这一条默认值很有用**：拿到一个可能为空的跳转目标时，`navigateTo(maybeUrl)` 不会跳到奇怪的地方，而是回根路径。

对象形态还能按名字与参数跳（官方 `to` 示例）：

```ts
// Passing the URL directly will redirect to the '/blog' page
await navigateTo('/blog')

// Using the route object, will redirect to the route with the name 'blog'
await navigateTo({ name: 'blog' })

// Redirects to the 'product' route while passing a parameter (id = 1) using the route object.
await navigateTo({ name: 'product', params: { id: 1 } })
```

### await 之后代码还会继续跑

组件侧那条 note 必须知道，否则会写出"跳转后还在改状态"的代码：

> **调用 `navigateTo` 不会停止你 `<script setup>` 其余代码的执行。如果你希望一次成功的导航从 `setup()` 提前返回，请启用 `experimental.navigateToEarlyReturn`。**

也就是说，**默认的语义是"导航被安排下去，但本文件后面的语句照常执行"**。要提前返回得开那个实验开关，而不是靠 `await` 本身。

## 位置二：路由中间件里 —— 必须 return

正例（官方原文）：

```ts
export default defineNuxtRouteMiddleware((to, from) => {
  if (to.path !== '/search') {
    // setting the redirect code to '301 Moved Permanently'
    return navigateTo('/search', { redirectCode: 301 })
  }
})
```

配套的规则句：**"在路由中间件里使用 `navigateTo` 时，你必须返回它的结果，以确保中间件执行流程正确工作。"**

然后官方专门给了一份**不工作**的写法：

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

这一处是"await 还是 return"最典型的分岔：**中间件里 `await navigateTo(...)` 然后 `return` 一个 `undefined`，等于告诉 Nuxt"这次导航我不管了、继续原目标"**——因为中间件的返回值本身就是指令（132 整篇讲这个）。所以中间件里的写法记成一句：**`return navigateTo(...)`，不要 `await` 之后再空 `return`。**

## 位置三：Nitro 路由里 —— 换函数

再看一遍那条边界：**"`navigateTo` 不能在 Nitro 路由里使用。要在 Nitro 路由里做服务端重定向，请改用 `sendRedirect`。"**

| 你在写 | 该用 |
| --- | --- |
| 页面组件 / 组合式函数 / 插件 | `await navigateTo(...)` |
| 路由中间件（Vue 侧） | `return navigateTo(...)` |
| `server/api/*.ts`、`server/routes/*.ts`、Nitro 事件处理器 | **`sendRedirect`** |

这也解释了一个常见误判："我在 API 路由里 `navigateTo` 了，浏览器却没跳"——那个位置根本不归它管。

## 与相邻几个手段的分界

| 目的 | 用什么 |
| --- | --- |
| 模板里的可点击链接 | `<NuxtLink>`（真 `href`、进入视口预取，110） |
| 代码里换地址 | `navigateTo` |
| 历史里前后移动 | `router.back()` / `forward()` / `go(n)`（123） |
| 运行时加/删路由记录 | `router.addRoute()` / `removeRoute()`（123） |
| 这次导航**别发生** | `abortNavigation()`（127） |
| 给用户一个错误页 | `createError` / `validate` 返回 `false`（111） |

## 自查

1. 官方说 `navigateTo` 在服务端与客户端都可用——这正是它相对 `router.push()` 的什么优势？
2. 那条唯一硬性规则的原话是什么？从签名看，为什么丢掉返回值有问题？
3. 组件里 `await navigateTo('/search')` 之后，后面的语句还会执行吗？想提前返回要做什么？
4. 中间件里的正确写法是什么？官方的 ❌ 反例错在哪一步？
5. `to` 传 `undefined` 或 `null` 时会怎样？
6. Nitro 路由里要重定向该用什么？
7. "让这次导航根本不发生"该用哪个函数？

## 下一篇

125 拆 `navigateTo` 的 options 四个键：`replace` 默认 `false` 的含义、`redirectCode` 为什么默认 302、`external` 不设时跳外部地址会**抛错**，以及 `open` 只能在客户端生效。

## 本文依据

- `https://nuxt.com/docs/4.x/api/utils/navigate-to`
  - 原文（描述）："navigateTo is a helper function that programmatically navigates users."
  - 原文（Usage）："`navigateTo` is available on both server side and client side. It can be used within the Nuxt context, or directly, to perform page navigation."
  - 原文（warning）："Make sure to always use `await` or `return` on result of `navigateTo` when calling it."
  - 原文（note）："`navigateTo` cannot be used within Nitro routes. To perform a server-side redirect in Nitro routes, use `sendRedirect` instead."
  - 原文（Within a Vue Component）：`await navigateTo('/search')`、`await navigateTo({ path: '/search' })`、`await navigateTo({ path: '/search', query: { page: 1, sort: 'asc' } })` 及其注释
  - 原文（note）："Calling `navigateTo` does not stop execution of the rest of your `<script setup>` code. If you want a successful navigation to return early from `setup()`, enable `experimental.navigateToEarlyReturn`."
  - 原文（Within Route Middleware）：示例 `return navigateTo('/search', { redirectCode: 301 })` 与注释 `// setting the redirect code to '301 Moved Permanently'`；"When using `navigateTo` within route middleware, you must **return its result** to ensure the middleware execution flow works correctly."；反例 "// ❌ This will not work as expected" + `navigateTo('/search', { redirectCode: 301 })` 后跟 `return`；"In this case, `navigateTo` will be executed but not returned, which may lead to unexpected behavior."
  - 原文（Type）：`export function navigateTo (to: RouteLocationRaw | undefined | null, options?: NavigateToOptions): Promise<void | NavigationFailure | false> | false | void | RouteLocationRaw`；`interface NavigateToOptions { replace?: boolean; redirectCode?: number; external?: boolean; open?: OpenOptions }`
  - 原文（`to`）：Type `RouteLocationRaw | undefined | null`、Default `'/'`；"`to` can be a plain string or a route object to redirect to. When passed as `undefined` or `null`, it will default to `'/'`."；示例 `await navigateTo('/blog')`、`await navigateTo({ name: 'blog' })`、`await navigateTo({ name: 'product', params: { id: 1 } })`
  - 原文（`external` 一节）："The `external` parameter in `navigateTo` influences how navigating to URLs is handled: Without `external: true`: Internal URLs navigate as expected. External URLs throw an error. With `external: true`: Internal URLs navigate with a full-page reload. External URLs navigate as expected."
  - 原文 Source：`packages/nuxt/src/app/composables/router.ts`
- `https://nuxt.com/docs/4.x/api/composables/use-router`
  - 对照用（见 123）：`push()`/`replace()` 两处都写 "It is recommended to use `navigateTo` instead."；`addRoute()` 与 `push()` 的场景区分
- `https://nuxt.com/docs/4.x/api/utils/abort-navigation`
  - 对照用（见 127）：`abortNavigation` 只在路由中间件处理器里可用
- `https://nuxt.com/docs/4.x/guide/going-further/nuxt-app`
  - 对照用（见 130）：`navigateTo` 所说的 "Nuxt 上下文" 出处
