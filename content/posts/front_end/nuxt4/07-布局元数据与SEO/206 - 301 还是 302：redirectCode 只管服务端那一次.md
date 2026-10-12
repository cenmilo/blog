---
url: /posts/d74786866a2c77c31fe5442dfebdd02e/
title: 301 还是 302：redirectCode 只管服务端那一次
date: 2026-12-19T11:00:00+08:00
updated: 2026-12-19T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：改地址的时候我以为"重定向就是重定向"，直到关心 SEO 才知道 301 与 302 是两回事（301 = 永久搬走，302 = 临时转一下；这两个词是 HTTP 状态码的含义，不在 Nuxt 文档里，文档只给了 MDN 链接）。Nuxt 侧能发起重定向的地方文档写了四处，**只有 `navigateTo` 的那个参数讲到了状态码语义**："navigateTo redirects to the given path and sets the redirect code to 302 Found **by default when the redirection takes place on the server side**. This default behavior can be modified by providing different redirectCode. Commonly, 301 Moved Permanently can be used for permanent redirections."——**默认 302，且这句话的前提是"发生在服务端"**。另三处各有键名与默认值：`routeRules` 用 `redirect: { to: '/new-page', statusCode: 302 }`（官方注释 "Redirection to avoid 404"）、模块侧 `extendRouteRules('/preview', { redirect: { to: '/preview-new', statusCode: 302 } })`、页面里 `definePageMeta({ redirect })`（类型是 `RouteRecordRedirectOption`，**它早于任何导航守卫执行**，文档没给它状态码参数）。另外 `navigateTo` 那一页专门给了中间件里的正反例：必须 **return**，"In this case, `navigateTo` will be executed but not returned, which may lead to unexpected behavior."

categories:
  - 前端开发

tags:
  - nuxt4
  - 重定向
  - 301
  - routeRules
  - SEO

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先把术语说人话：状态码是服务器响应里那个三位数字。`301` 表示"这个地址永久搬走了，以后请直接记新地址"，`302` 表示"这次临时转到别处，原地址还是主人"。搜索引擎对两者的处理不同，所以旧地址要不要保留权重、要不要被替换，取决于你写的是哪一个。这两个数字的语义不在 Nuxt 文档里——文档只做了一件事：给每个键挂了 MDN 的状态码页面链接。

## 四处能发起重定向的地方

| 位置 | 写法 | 状态码怎么给 |
| --- | --- | --- |
| route middleware / 页面代码 | `navigateTo('/search', { redirectCode: 301 })` | `redirectCode`，**默认 `302`** |
| `nuxt.config` 的 `routeRules` | `'/old-page': { redirect: { to: '/new-page', statusCode: 302 } }` | `statusCode`，示例给的是 302 |
| 模块里的 `extendRouteRules` | `extendRouteRules('/preview', { redirect: { to: '/preview-new', statusCode: 302 } })` | 同上，`statusCode` |
| 页面里的 `definePageMeta` | `definePageMeta({ redirect: ... })` | 文档**没给**状态码参数 |

注意键名不统一：`navigateTo` 那边叫 `redirectCode`，`routeRules` 那边叫 `statusCode`。我在改的时候因为记混白跑了一次构建，所以把这一行单独写出来。

`routeRules` 那条示例来自服务端那一页，它的注释很有信息量：

```ts twoslash [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    // Generated at build time for SEO purpose
    '/': { prerender: true },
    // Cached for 1 hour
    '/api/*': { cache: { maxAge: 60 * 60 } },
    // Redirection to avoid 404
    '/old-page': {
      redirect: { to: '/new-page', statusCode: 302 },
    },
    // ...
  },
})
```

同一节还有一句解释这套规则在哪生效："Nuxt has a powerful feature called `routeRules` which allows you to define a set of rules to customize how each route of your Nuxt app is rendered (and more). Nuxt compiles the rules, and the default server builder (Nitro) applies the server-side ones."——**规则是构建期编译、由服务端应用的**。这条正好和 `redirectCode` 那句"发生在服务端时"对上：想在 HTTP 层面拿到状态码，得让重定向真的发生在服务端。

## navigateTo 的 redirectCode 原文

`api/utils/navigate-to` 的参数说明里，这一条完整是这样的：

- `redirectCode`
  - **Type**: `number`
  - **Default**: `302`
  - `navigateTo` redirects to the given path and sets the redirect code to [`302 Found`](...) by default when the redirection takes place on the server side.

    This default behavior can be modified by providing different `redirectCode`. Commonly, [`301 Moved Permanently`](...) can be used for permanent redirections.

我把 "**when the redirection takes place on the server side**" 划成重点，因为它同时说了两件事：默认 302 是服务端那次的默认；客户端导航本来就没有 HTTP 状态码可设。对照同节 `replace` 那条更容易看清两侧分工：

- `replace`
  - **Type**: `boolean`
  - **Default**: `false`
  - By default, `navigateTo` pushes the given route into the Vue Router's instance on the client side.

    This behavior can be changed by setting `replace` to `true`, to indicate that given route should be replaced.

也就是说，**客户端那侧你能控制的是"要不要留在历史记录里"（`replace`），服务端那侧你能控制的是状态码（`redirectCode`）**。这两件事不是一个开关，我一开始误以为 `replace: true` 就等价于永久跳转。

## 中间件里必须 return，否则状态码根本发不出去

同一页给的正反例，正好是 SEO 场景里最常写错的形式：

```ts
export default defineNuxtRouteMiddleware((to, from) => {
  if (to.path !== '/search') {
    // setting the redirect code to '301 Moved Permanently'
    return navigateTo('/search', { redirectCode: 301 })
  }
})
```

> When using `navigateTo` within route middleware, you must **return its result** to ensure the middleware execution flow works correctly.

反例页面上也标了：

```ts
export default defineNuxtRouteMiddleware((to, from) => {
  if (to.path !== '/search') {
    // ❌ This will not work as expected
    navigateTo('/search', { redirectCode: 301 })
    return
  }
})
```

> In this case, `navigateTo` will be executed but not returned, which may lead to unexpected behavior.

这就是 132 讲过的"忘记 return 表现像成功"。对 SEO 来说它更隐蔽：**浏览器可能确实跳过去了，但服务端返回的状态码不是你写的那个**——因为那个 `301` 从没被当成中间件的返回值交出去。判据很直接：`curl -I` 看服务端响应的状态行，别只看浏览器地址栏。

顺带把这条链补全：Nitro 服务端路由里不能用 `navigateTo`，官方明说改用 `sendRedirect`（124 记过那条 warning）。

## definePageMeta 的 redirect：它抢在守卫之前

`PageMeta` 接口里这一行的类型是 `redirect?: RouteRecordRedirectOption`。关于它的执行时机，官方在字段说明里写的是"**redirect 早于任何导航守卫**"（135 抄过），这带来一个和中间件方案的实质差别：写在页面元数据里的重定向，你自己注册的中间件都还没跑；而中间件里的 `navigateTo` 是守卫链里的一环。

但同一处文档**没有给状态码参数**——`RouteRecordRedirectOption` 是 vue-router 那边的类型，Nuxt 这一页没有说明它最终返回哪个状态码。所以在需要 301 的场合，我不会用 `definePageMeta({ redirect })`，而是把这条规则放到 `routeRules` 或中间件里，因为那两处文档明确写了状态码。理由的强度要说清：**"没有状态码参数"是文档层面的空白，不是"它一定返回 302"**，要用就 `curl -I` 实测一次。

## 我按这四条排的选择顺序

1. 纯静态搬迁、地址永久改：`routeRules` 的 `redirect` + `statusCode: 301`（服务端、声明式、不占中间件）。
2. 需要按用户/权限判断才转：中间件里 `return navigateTo(to, { redirectCode: 301 })`。
3. 模块作者给宿主应用加规则：`extendRouteRules`，形状同 `routeRules`。
4. 页面自己知道要去哪、且不在乎状态码：`definePageMeta({ redirect })`，但要记住它早于守卫。
5. 临时同页跳转、不想留历史记录：客户端 `navigateTo(..., { replace: true })`——**这一条不该用于 SEO 场景的地址迁移**，因为它压根不涉及状态码。

## 自查

1. `redirectCode` 的默认值是多少，它的前提句是什么？（答：`302`；原文条件是"when the redirection takes place on the server side"。）
2. 中间件里漏写 return 会怎样，怎么快速发现？（答：`navigateTo` 执行了但没返回，行为不符合预期；用 `curl -I` 看服务端状态行。）
3. `routeRules` 与 `navigateTo` 两边控制状态码的键分别叫什么？（答：`statusCode` 与 `redirectCode`。）

## 下一篇

207 - 错误页那一层要不要套外壳：它不是路由所以没有页面元数据

## 本文依据

- `api/utils/navigate-to`：`redirectCode` 条目全文（Type `number`、Default `302`、两句说明含 MDN 状态码链接）；`replace` 条目全文；`NavigateToOptions` 四键接口；Within Route Middleware 的正反两段示例与三句说明（"you must **return its result**"、"❌ This will not work as expected"、"will be executed but not returned, which may lead to unexpected behavior"）；`to` 参数默认 `'/'`。
- `getting-started/server`：`routeRules` 示例三行（`'/': { prerender: true }` 的注释 "Generated at build time for SEO purpose"、`cache: { maxAge: 60 * 60 }`、`redirect: { to: '/new-page', statusCode: 302 }` 的注释 "Redirection to avoid 404"）与原话 "Nuxt compiles the rules, and the default server builder (Nitro) applies the server-side ones."
- `api/kit/pages`：`extendRouteRules('/preview', { redirect: { to: '/preview-new', statusCode: 302 } })` 与 `extendPages` 并置的模块示例。
- `api/utils/define-page-meta`：`redirect?: RouteRecordRedirectOption`；字段说明中"早于任何导航守卫"这一条（与 135 同源）。
- `directory-structure/app/middleware`：`navigateTo` 与 `sendRedirect` 的分工（124 已引 Nitro 路由那条 warning）。
