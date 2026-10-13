---
url: /posts/cecc5a49f0aab428d63953345abfc9ec/
title: 127 - abortNavigation：只在中间件里可用、签名返回 false 的三种写法
date: 2026-11-23T10:00:00+08:00
updated: 2026-11-23T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：`abortNavigation` 的官方定义同时说了两件事："**`abortNavigation` 是一个 helper 函数，它会阻止导航发生，并且如果作为参数设置了一个错误，它会抛出这个错误。**"配套一条使用范围限制（warning）：**"`abortNavigation` 只能在路由中间件处理器（route middleware handler）里使用。**"它的签名很短：`export function abortNavigation (err?: Error | string): false`——参数是可选的 `Error` 或 `string`，官方说明为"**由 `abortNavigation` 抛出的可选错误**"，而**返回类型是 `false`**。这个返回类型解释了为什么官方三处示例全都写成 `return abortNavigation(...)`：在中间件里，返回值本身就是指令（132 展开）。官方给的三个例子按三种参数形态排列。第一个是**不带参数**的用法，场景是"用一个在路由中间件里调用的 `abortNavigation` 来阻止未授权的路由访问"：`const user = useState('user')`、`if (!user.value.isAuthorized) { return abortNavigation() }`，后面还留着正常分支 `if (to.path !== '/edit-post') { return navigateTo('/edit-post') }`——**同一段里"取消这次导航"和"换个目标"两件事并排出现**。第二个是**字符串**形态：`return abortNavigation('Insufficient permissions.')`。第三个是**Error 对象**形态，官方的说明是"你可以把错误作为一个 `Error` 对象传入，例如从 `catch` 块里抓到的那种"：`try { /* code that might throw an error */ } catch (err) { return abortNavigation(err) }`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 中间件
  - abortNavigation
  - 导航控制
  - 错误

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


中间件里发现用户不该去这一页，有两种结局：把他送去登录页，或者**让这次导航压根没发生**。后者的工具就是 `abortNavigation`，它只有三行文档，但每一条都算一个限制。

## 官方定义里塞了两件事

> **`abortNavigation` 是一个 helper 函数，它会阻止导航发生，并且如果作为参数设置了一个错误，它会抛出这个错误。**

拆成两个能力：

| 能力 | 含义 |
| --- | --- |
| **阻止导航发生** | 用户留在原来的页面上——地址不变、当前页面组件不卸载 |
| 带参数时**抛出该错误** | 顺带把"为什么被挡"变成一个可被处理的错误 |

紧接着是使用范围那条 warning：

> **`abortNavigation` 只能在路由中间件处理器（route middleware handler）里使用。**

**这一条是硬边界。** 它不是一个"随处可调用"的取消 API：组件里点按钮、插件里、服务端路由里都不能用它。要"取消一次导航"这件事，本身只在导航流程中才有意义，而 Nuxt 给的正式导航流程入口就是路由中间件。

## 签名：返回类型是 `false`

```ts
export function abortNavigation (err?: Error | string): false
```

| 部分 | 官方说明 |
| --- | --- |
| `err` | 类型 `Error \| string`；"**由 `abortNavigation` 抛出的可选错误**" |
| 返回值 | **`false`** |

返回 `false` 这一点，决定了它的写法只有一种：**`return abortNavigation(...)`**。因为中间件的返回值就是指令（132），而 `false` 在这里的语义正是"这次导航不继续"。

反过来看官方对 `navigateTo` 的要求——"**在路由中间件里使用 `navigateTo` 时，你必须返回它的结果**"（124）——两者是同一条规则的两侧：**在中间件里，你不 `return`，就等于没做。**

## 三种写法：官方按参数形态给例子

### ① 不带参数：只是不让这次导航发生

官方的场景说明："**下面的例子展示了你可以如何在路由中间件里使用 `abortNavigation` 来阻止未授权的路由访问。**"

```ts [app/middleware/auth.ts]
export default defineNuxtRouteMiddleware((to, from) => {
  const user = useState('user')

  if (!user.value.isAuthorized) {
    return abortNavigation()
  }

  if (to.path !== '/edit-post') {
    return navigateTo('/edit-post')
  }
})
```

这段官方示例很值得多看一眼，因为它把两个动作**并排**放在一起：

| 分支 | 效果 |
| --- | --- |
| `return abortNavigation()` | 用户**留在原地**，这次点击/导航取消 |
| `return navigateTo('/edit-post')` | 用户被**改送到另一页** |

写中间件时，这两个结局要先选一个再谈实现：**"不让去"和"送到别处"是两条不同的路径。**

### ② 字符串形态：带上"为什么"

官方说明："**你可以把错误作为一个字符串传入：**"

```ts [app/middleware/auth.ts]
export default defineNuxtRouteMiddleware((to, from) => {
  const user = useState('user')

  if (!user.value.isAuthorized) {
    return abortNavigation('Insufficient permissions.')
  }
})
```

这里的取值是 `err` 那条能力——"如果作为参数设置了一个错误，它会抛出这个错误"。传字符串就是给这次中止配一句原因。

### ③ Error 对象形态：接住已发生的错误

官方说明："**你可以把错误作为一个 `Error` 对象传入，例如从 `catch` 块里抓到的那种：**"

```ts [app/middleware/auth.ts]
export default defineNuxtRouteMiddleware((to, from) => {
  try {
    /* code that might throw an error */
  } catch (err) {
    return abortNavigation(err)
  }
})
```

这一格的用法与前两格不同：它不是为了权限，而是**中间件自己跑出了异常，你决定"这次导航作废并把原错误抛出去"**，而不是让一个半完成的状态继续往下渲染。

## 别把它当成 404 或错误页

`abortNavigation` 与"给用户一个错误页"是两套机制，选错了用户会看到一个什么都没发生的页面：

| 你想要的结局 | 用什么 | 用户看到 |
| --- | --- | --- |
| 这次导航作废，留在原页 | `abortNavigation(err?)` | 页面**没变化** |
| 送到登录页/别的页 | `return navigateTo('/login')` | 换了一页 |
| 展示错误页（404/500） | `throw createError({ status, statusText })`（111） | 全屏错误页 |
| 参数非法直接判 404 | `definePageMeta({ validate })` 返回 `false`（111） | 404 |
| 只让某一块局部报错 | `<NuxtErrorBoundary>`（095） | 局部替身内容 |

**注意第二行的差别**：未授权场景里 `abortNavigation()` 让用户留在原地，他不知道点了什么；要让他知道，通常是 `navigateTo('/login')` 或用 `createError`——`abortNavigation` 的参数只解决"抛出一个错误"，不解决"页面上要有反馈"。

## 与相邻工具的位置关系

| 位置 | 能用的取消/改道手段 |
| --- | --- |
| 路由中间件（Vue 侧） | `abortNavigation`、`return navigateTo(...)`、`redirect` 记录属性 |
| 记录属性（编译期） | `definePageMeta({ redirect })`——官方写明"**重定向发生在任何导航守卫之前**"（113） |
| 组件内 | `onBeforeRouteLeave` / `onBeforeRouteUpdate`（134 展开） |
| 实例级守卫 | `router.beforeEach` / `beforeResolve` / `afterEach`（123；官方建议用中间件替代） |
| Nitro 路由（服务端） | 不归路由中间件管；`navigateTo` 也用不了，用 `sendRedirect`（124） |

一条容易混的顺序问题：`redirect` 跑在任何导航守卫之前，所以**中间件里写的 `abortNavigation` 拦不住一条已经带 `redirect` 的记录**——那一次导航在守卫之前就换目标了。

## 自查

1. 官方定义里 `abortNavigation` 同时做了哪两件事？
2. 它能用在哪里？官方那条 warning 原文怎么说？
3. 它的签名返回什么？这决定了调用写法必须是什么？
4. 不带参数、字符串、Error 对象三种形态各适合什么场景？
5. "让这次导航作废"与"送他去登录页"、"给他一个 404"三者区别是什么？
6. 为什么 `abortNavigation` 拦不住一条设了 `redirect` 的路由记录？

## 下一篇

128 讲路由中间件本体：三种写法（匿名、命名、全局）、它跑在 Vue 侧而不是 Nitro 侧、不为 `/api/*` 运行，以及命名与全局中间件各自的注册规则。

## 本文依据

- `https://nuxt.com/docs/4.x/api/utils/abort-navigation`
  - 原文（描述）："abortNavigation is a helper function that prevents navigation from taking place and throws an error if one is set as a parameter."
  - 原文（warning）："`abortNavigation` is only usable inside a route middleware handler."
  - 原文（Type）：`export function abortNavigation (err?: Error | string): false`
  - 原文（Parameters - `err`）：Type `Error | string`；"Optional error to be thrown by `abortNavigation`."
  - 原文（Example）："The example below shows how you can use `abortNavigation` in a route middleware to prevent unauthorized route access:"；示例（`app/middleware/auth.ts`）`const user = useState('user')`、`if (!user.value.isAuthorized) { return abortNavigation() }`、`if (to.path !== '/edit-post') { return navigateTo('/edit-post') }`
  - 原文（`err` as a String）："You can pass the error as a string:"；示例 `return abortNavigation('Insufficient permissions.')`
  - 原文（`err` as an Error Object）："You can pass the error as an Error object, e.g. caught by the `catch`-block:"；示例 `try { /* code that might throw an error */ } catch (err) { return abortNavigation(err) }`
  - 原文 Source：`packages/nuxt/src/app/composables/router.ts`
- `https://nuxt.com/docs/4.x/api/utils/navigate-to`
  - 对照用（见 124、125、126）："When using `navigateTo` within route middleware, you must **return its result** to ensure the middleware execution flow works correctly."；"Make sure to always use `await` or `return` on result of `navigateTo` when calling it."；"`navigateTo` cannot be used within Nitro routes."
- `https://nuxt.com/docs/4.x/api/utils/define-page-meta`
  - 对照用（见 113）：`redirect` —— "The redirection happens before any navigation guard and triggers a new navigation with the new target location."
- `https://nuxt.com/docs/4.x/api/composables/use-router`
  - 对照用（见 123）：`afterEach`/`beforeEach`/`beforeResolve` 与"route middleware 简化了导航守卫实现"的建议
- `https://nuxt.com/docs/4.x/getting-started/error-handling` 与 `https://nuxt.com/docs/4.x/directory-structure/app/pages`
  - 对照用（见 111、095）：`createError({ status, statusText })` 与 `validate` 返回 `false` 造成 404；`<NuxtErrorBoundary>` 只处理默认插槽内的客户端错误
