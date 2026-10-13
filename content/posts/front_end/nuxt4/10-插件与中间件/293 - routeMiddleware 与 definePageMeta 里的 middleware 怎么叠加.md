---
url: /posts/68cde99f07d158e7f52403119b9c11d9/
title: 293 - routeMiddleware 与 definePageMeta 里的 middleware 怎么叠加
date: 2027-01-17T11:00:00+08:00
updated: 2027-01-17T11:00:00+08:00
author: cmdragon

summary:
  摘要：一个页面要跑哪几个中间件、按什么顺序跑，有两个地方在声明：目录里带 .global 的全局中间件，和页面 definePageMeta 里的 middleware 字段。这一篇讲 definePageMeta.middleware 的三种写法（内联函数、单个字符串名、字符串数组），官方给的执行顺序（全局先按文件名字母序、再页面声明的顺序、页面里内联函数排在它前面），以及两处覆盖关系——构建期 pages:extend 里的 page.meta.middleware 会盖掉 definePageMeta、同名中间件在插件注册那份赢。

categories:
  - 前端开发

tags:
  - nuxt4
  - 中间件
  - definePageMeta
  - 执行顺序
  - 路由
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上一篇（292）用 `definePageMeta({ middleware: ['authenticated'] })` 给页面点名了一个中间件。这一篇把"一个页面的中间件到底怎么声明、多个的先后怎么定"讲全。

## definePageMeta 里的 middleware：三种写法

`definePageMeta` 的类型定义里，这个字段是这样的：

```ts
middleware?: MiddlewareKey | NavigationGuard | Array<MiddlewareKey | NavigationGuard>
```

官方对它的说明只有一句，但点出了两种来源：

> Define anonymous or named middleware directly within `definePageMeta`.

anonymous = 内联匿名函数，named = 目录里的命名中间件。合起来对应三种写法，官方给的示例一次全列了：

```vue [app/pages/some-page.vue]
<script setup lang="ts">
definePageMeta({
  // 写成函数
  middleware: [
    function (to, from) {
      const auth = useState('auth')
      if (!auth.value.authenticated) {
        return navigateTo('/login')
      }
      if (to.path !== '/checkout') {
        return navigateTo('/checkout')
      }
    },
  ],

  // ... 或一个字符串（命名中间件）
  middleware: 'auth',

  // ... 或多个字符串
  middleware: ['auth', 'another-named-middleware'],
})
</script>
```

要点：

- **字符串**（`'auth'`）指向 `app/middleware/auth.ts` 里的命名中间件；名字规范化成 kebab-case（`myMiddleware` → `my-middleware`，290 提过）。
- **数组**能把命名中间件和内联函数混在一个数组里（示例就同时放了 function 和字符串）。数组的顺序就是"页面声明的顺序"，见下面。
- 内联函数里能 `useState('auth')`、`navigateTo(...)`——因为内联中间件跑在 Nuxt 上下文里；但同样**别在里面用 `useRoute()`**（290 那条"中间件里没有当前路由概念"对匿名中间件也成立，参数只用 `to`/`from`）。

## 多个中间件的执行顺序

一个页面可能同时有：全局中间件（`.global`）、页面 definePageMeta 声明的。官方 "Middleware Order" 给的规则是两步：

> Middleware runs in the following order:
> 1. Global Middleware
> 2. Page defined middleware order (if there are multiple middleware declared with the array syntax)

官方还给了一个完整对照——目录有 `analytics.global.ts`、`setup.global.ts`、`auth.ts`，页面里 `middleware: [function(){...}, 'auth']`，那么实际执行顺序是：

1. `analytics.global.ts`
2. `setup.global.ts`
3. 内联的 custom middleware（函数）
4. `auth.ts`

读这张顺序表抓三点：

- **全局永远先跑**，且全局之间按文件名字母序——所以 `setup` 想排在 `analytics` 前，得用编号前缀补零（290 讲过 `10.` 会排在 `2.` 前）。
- 页面声明里，**内联函数在数组中的位置**决定它和命名中间件的相对先后（示例里函数写在 `'auth'` 前面，就先跑）。
- 全局那批先走完，才轮到页面这批。

## 两处覆盖：definePageMeta 不是终点

中间件"声明在哪"有三处能写，官方给了明确的覆盖关系：

**一、构建期 pages:extend 盖掉 definePageMeta。** app/middleware 那页 "Setting Middleware at Build Time" 的示例里，注释直接写死：

```ts
// Note that this will override any middleware set in `definePageMeta` in the page
page.meta.middleware = ['named']
```

也就是说在 `nuxt.config` 的 `pages:extend` 钩子里给某条路由设 `page.meta.middleware`，会**覆盖**页面里 `definePageMeta` 写的中间件。适合"按路径批量给一批页面加同一个中间件"，不用每页手点。

**二、插件里 addRouteMiddleware 盖同名。** 290 讲过：插件里用 `addRouteMiddleware` 注册一个和目录同名的中间件，"would override any existing middleware of the same name"——插件那份赢。这是"同一个中间件名，谁的定义生效"的覆盖，不是"页面用哪些"的覆盖，别和上面那条混。

一张归属表：

| 决定什么 | 在哪声明 | 覆盖关系 |
| --- | --- | --- |
| 某页跑哪些命名/内联中间件 | `definePageMeta({ middleware })` | 会被 pages:extend 的 `page.meta.middleware` 覆盖 |
| 全站每页都跑的中间件 | 目录 `.global.ts` / `addRouteMiddleware(...,{global:true})` | 先于页面声明执行 |
| 同名中间件用哪份定义 | `app/middleware/xxx.ts` vs 插件注册 | 插件注册那份覆盖目录的 |

## 自查

- 一个页面用哪些中间件、什么顺序，看 `definePageMeta.middleware` 数组顺序；但**全局中间件永远先跑**。
- 内联函数和命名中间件能混在一个数组里，数组位置就是它们的相对顺序。
- 想按路径批量套中间件、又不想逐页写 → `pages:extend` 里设 `page.meta.middleware`（会盖过 definePageMeta）。
- 内联中间件里同样别用 `useRoute()`，只用 `to`/`from`。

## 下一篇

下一篇 294 回到钩子总览：先分清"构建期钩子"和"运行时钩子"这两组，是理解后面所有钩子话题的地基。

## 本文依据

- definePageMeta（`api/utils/define-page-meta`）：`PageMeta` 里 `middleware?: MiddlewareKey | NavigationGuard | Array<MiddlewareKey | NavigationGuard>`；"Defining Middleware" 一节（"Define anonymous or named middleware directly within definePageMeta"、内联 function（`useState('auth')`+`navigateTo`）/单字符串 `'auth'`/多字符串数组三形态示例）。
- middleware 目录（`directory-structure/app/middleware`）：Middleware Order（"1. Global Middleware 2. Page defined middleware order"、analytics.global→setup.global→inline→auth 的示例顺序）、Ordering Global Middleware（字母序、补零）、"Setting Middleware at Build Time"（pages:extend 里 `page.meta.middleware = ['named']` 及 "will override any middleware set in definePageMeta" 注释）、"Adding Middleware Dynamically"（addRouteMiddleware 同名覆盖，见 290）、kebab-case 规范化。
