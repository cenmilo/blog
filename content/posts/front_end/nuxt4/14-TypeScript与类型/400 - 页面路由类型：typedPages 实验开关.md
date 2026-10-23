---
url: /posts/20e075373da46aad5b8a5bd27acd6799/
title: 400 - 页面路由类型：typedPages 实验开关
date: 2027-02-22T10:00:00+08:00
updated: 2027-02-22T10:00:00+08:00
author: cmdragon

summary:
  摘要：`navigateTo('/usr/1')` 少打一个字母，运行时才知道 404——这类错误在 Nuxt 里可以用 `experimental.typedPages` 提前到编辑器阶段拦住。官方定义是"Enable the new experimental typed router."，打开后一句话说明覆盖面："Out of the box, this will enable typed usage of `navigateTo`, `<NuxtLink>`, `router.push()` and more."——跳转的三个常用入口同时受类型约束。额外一条能力：在页面里写 `const route = useRoute('route-name')` 能拿到**按该路由参数定型的 route**，`route.params.id` 不再是 `string | string[]` 那种含糊类型。它是实验开关、需要显式开启；`navigateTo` 本身的行为（必须 await/return、在 Nitro 路由里不能用）不受这个开关影响。

categories:
  - 前端开发

tags:
  - nuxt4
  - typedPages
  - 类型安全路由
  - navigateTo
  - 实验特性
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

路由是字符串这件事很方便，也很危险：`/user/:id` 你写成 `/usr/1`，编译不报错、运行就 404。Nuxt 把路由表喂给类型系统来解决这类问题，开关是 `experimental.typedPages`。

## 它开的是什么

官方定义一句："Enable the new experimental typed router."

开启方式：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    typedPages: true,
  },
})
```

覆盖面官方一句话讲完：

> Out of the box, this will enable typed usage of [`navigateTo`], [`<NuxtLink>`], [`router.push()`] and more.

也就是**跳转的三个常用入口同时受约束**：`navigateTo()` 的 `to`、`<NuxtLink to="...">`、以及 `useRouter()` 拿到的 `router.push()`。写错路径名或漏了必填参数，编辑器和 `nuxt typecheck` 会在你还没运行时标出来。

这是实验特性，需要显式打开——不是默认状态。

## 额外一条：页面内的参数类型

同一页还点了一条常被忽略的能力：

> You can even get typed params within a page by using `const route = useRoute('route-name')`.

差别在 `useRoute()` 传不传**路由名**。`pages/user/[id].vue` 声明了名字之后，用 `useRoute('user-id')` 这种带名字的形式拿到的 route，其 `params` 是按那条路由定型的，而不是默认的模糊类型。官方在这一节挂了 Daniel Roe 讲 "type-safe routing in Nuxt" 的视频。

## 它不改 navigateTo 的运行时行为

要划清边界：`typedPages` 只管**类型**，`navigateTo` 的运行时规则一条都没变，那几条都得照常遵守（`api/utils/navigate-to`）：

- 它是 server 与 client 都可用的，要在 Nuxt 上下文里用，也可以直接调。
- `::warning`：**调用时永远对结果 `await` 或 `return`。**

  ```ts
  await navigateTo('/search')
  await navigateTo({ path: '/search', query: { page: 1, sort: 'asc' } })
  ```

- `::note`：**`navigateTo` 不能在 Nitro 路由里用**；要在 Nitro 路由做服务端重定向，改用 h3 的 `sendRedirect`。
- 在路由中间件里必须**返回**它的结果，否则中间件执行流程不对。
- 调用 `navigateTo` **不会停下** `<script setup>` 剩下的代码；想让导航成功后提前从 `setup()` 返回，另有实验开关 `experimental.navigateToEarlyReturn`（这条独立于 typedPages）。

## 和另两个类型化开关的层次

395-414 里连着三个和"路由/接口类型化"有关的开关，别混：

| 开关 | 管什么 |
| --- | --- |
| `experimental.typedPages` | 页面路由：`navigateTo`/`<NuxtLink>`/`router.push` 的目标路径与参数 |
| `experimental.routeTypedFetch` | 内部 API 路径：`$fetch`/`useFetch` 按服务端路由定型（402） |
| `experimental.strictRouteTypes` | 配套上一条：Nuxt 认不出的路径要不要直接报错（401） |

第一个管"跳去哪"，后两个管"请求哪个接口"。

## 自查

- `experimental.typedPages: true` 打开类型化路由，让 `navigateTo`、`<NuxtLink>`、`router.push()` 的目标受类型约束；实验特性、需显式开。
- 页面内 `useRoute('route-name')` 能拿到按该路由参数定型的 route。
- 只改类型不改运行时：`await`/`return` 必带、Nitro 路由里不能用（换 `sendRedirect`）、中间件里要 return。
- 想让导航成功即提前返回 setup，是另一个开关 `experimental.navigateToEarlyReturn`。
- 与 `routeTypedFetch`/`strictRouteTypes` 分层：前者管页面路径，后两个管内部接口。

## 下一篇

下一篇 401 讲 `strictRouteTypes`：把"只允许已注册的路由名与参数"这件事推到接口请求侧，`false`/`true`/`'isomorphic'` 三档各自的取舍与官方给的理由句。
