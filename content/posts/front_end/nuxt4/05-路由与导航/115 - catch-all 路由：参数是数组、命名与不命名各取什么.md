---
url: /posts/18d32d49d3014a04761b9683755beff0/
title: 115 - catch-all 路由：参数是数组、命名与不命名各取什么
date: 2026-11-19T10:00:00+08:00
updated: 2026-11-19T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：catch-all 的官方定义只有一句："如果你需要一个 catch-all 路由，就用一个像 `[...slug].vue` 这样的文件名来创建它。**它将匹配该路径下的所有路由。**"但取值形状必须靠官方给的输出例子才知道，而那个例子很容易被误读：模板写 `<p>{{ $route.params.slug }}</p>`，**导航到 `/hello/world` 会渲染出 `<p>["hello", "world"]</p>`**——方括号与引号是数组被序列化后的样子，也就是说 **`params.slug` 是一个数组**（每一项是一段路径，不带斜杠），不是一个字符串。服务端那一侧是同一族语法、另一套取值口径：官方说"**catch-all 路由对兜底路由处理很有用。例如，创建一个名为 `~~/server/api/foo/[...].ts` 的文件会为所有不匹配任何路由处理器的请求注册一个 catch-all 路由，例如 `/api/foo/bar/baz`**"，并在示例注释里给出两个取值点——`// event.context.path to get the route path: '/api/foo/bar/baz'` 与 **`// event.context.params._ to get the route segment: 'bar/baz'`**（注意 key 是一个下划线）。官方还补了命名形式："**你可以用一个名为 `~~/server/api/foo/[...slug].ts` 的文件为 catch-all 路由设置一个名字，并通过 `event.context.params.slug` 访问它**"，示例注释是 `// event.context.params.slug to get the route segment: 'bar/baz'`。把两侧放在一起看是这一篇的重点：**页面侧 named catch-all 的参数是数组（`["hello","world"]`），服务端侧官方注释给出的形态是 `bar/baz` 这样一段路径串，而不命名的 catch-all 的 key 是 `_`。**

categories:
  - 前端开发

tags:
  - nuxt4
  - catch-all
  - 路由参数
  - 服务端路由
  - 兜底

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


`[...slug].vue` 这一种文件，写法一眼就会，取值却最容易做错——因为**它给的不是字符串**。这一篇把页面侧与服务端侧的取值口径一次摆平。

## 官方定义与那句关键示例

> 如果你需要一个 catch-all 路由，就用一个像 `[...slug].vue` 这样的文件名来创建它。**它将匹配该路径下的\*所有\*路由。**

紧接着的示例就是取值形状的全部证据：

```vue [app/pages/[...slug].vue]
<template>
  <p>{{ $route.params.slug }}</p>
</template>
```

> 导航到 `/hello/world` 会渲染：
>
> ```html
> <p>["hello", "world"]</p>
> ```

这段输出要读对：**`["hello", "world"]` 里的方括号和引号，是数组被序列化成文本显示出来的样子。** 也就是说：

| 事实 | 结论 |
| --- | --- |
| `params.slug` 的类型 | **数组**，不是字符串 |
| 每一项 | 一个路径段，**不带斜杠**（`"hello"`、`"world"`） |
| `/hello` 这种一段的情况 | 数组只有一项 |

所以三种常见需求对应三种写法：拼回路径是 `slug.join('/')`（**注意拼出来没有前导斜杠**，官方示例的两项就是 `hello/world`）、取第一段是 `slug[0]`、判空要判的是数组长度而不是字符串真假。

## 服务端那一侧：同一个语法，另一套取值

服务端文档里的 catch-all 一节的定位不同，官方先说用途：**"catch-all 路由对兜底路由处理很有用。"**

然后是那条定义：

> 例如，创建一个名为 `~~/server/api/foo/[...].ts` 的文件，会为**所有不匹配任何路由处理器的请求**注册一个 catch-all 路由，例如 `/api/foo/bar/baz`。

```ts [server/api/foo/[...].ts]
export default defineEventHandler((event) => {
  // event.context.path to get the route path: '/api/foo/bar/baz'
  // event.context.params._ to get the route segment: 'bar/baz'
  return `Default foo handler`
})
```

**这里有两个和页面侧不同的点**，官方是用注释直接给出的：

1. **key 是 `_`**：`event.context.params._`。因为文件名 `[...]` 里**没有名字**，服务端就用一个下划线来放这一段。
2. **拿到的形态是 `bar/baz`**：官方注释写的是 "to get the route segment: 'bar/baz'"——一段路径串。整条路径另有其键：`event.context.path`（`'/api/foo/bar/baz'`）。

想拿到命名参数，官方给了对应的写法：**"你可以用一个名为 `~~/server/api/foo/[...slug].ts` 的文件为 catch-all 路由设置一个名字，并通过 `event.context.params.slug` 访问它。"**

```ts [server/api/foo/[...slug].ts]
export default defineEventHandler((event) => {
  // event.context.params.slug to get the route segment: 'bar/baz'
  return `Default foo handler`
})
```

## 两侧对照表

这张表是这一篇要留下的东西：

| | 页面（`app/pages/`） | 服务端（`server/`） |
| --- | --- | --- |
| 不命名形式 | 官方示例只给了 `[...slug].vue` | **`[...].ts`** |
| 取值入口 | `$route.params.slug` / `useRoute().params.slug` | `event.context.params._` 或 `.slug` |
| 官方展示的形态 | `["hello", "world"]`（**数组**） | `bar/baz`（**路径串**） |
| 整路径 | `useRoute().path`（另见 121） | `event.context.path` = `/api/foo/bar/baz` |
| 官方给的用途 | "匹配该路径下的所有路由" | "对兜底路由处理很有用"，处理不匹配任何处理器的请求 |
| 层级前缀 | 放在 `pages/` 根目录时匹配全站各深度 | 示例挂在 `/api/foo` 下，兜的是这个前缀下的未匹配 |

**结论一句话：同一族方括号语法，页面侧按"段数组"给你，服务端侧按"剩余路径串"给你。** 两侧切换时不要假设对方能用。

## 三个容易做错的地方

| 现场 | 原因 |
| --- | --- |
| 把 `params.slug` 当字符串去 `split('/')` | 它本来就是数组，官方示例的输出形态就是证据 |
| 拼 URL 时多写了斜杠或忘了前导斜杠 | 数组项不带斜杠，`join('/')` 的结果是 `hello/world` |
| 服务端里写 `params.slug` 但文件名是 `[...].ts` | 不命名的 catch-all 在官方示例里 key 是 `_`；要用 `slug` 得把文件名写成 `[...slug].ts` |

## 什么时候用 catch-all，什么时候用别的

三种手段各管一类需求：

| 你要的匹配 | 用 |
| --- | --- |
| 一段、必填 | `[id].vue`（114） |
| 一段、可有可无（**只到这一层**） | `[[slug]].vue` 或 `[[slug]]/index.vue`（114：官方说它"会同时匹配 `/` 和 `/test`"） |
| **任意深度全部接住** | `[...slug].vue`（本篇） |
| 段的内容要受约束（比如必须是数字） | `definePageMeta({ path })` + 自定义正则（113） |

另外别忘了 111 讲过的分工：catch-all **只是把 URL 交给你这个页面**，它本身不等于 404——要不要报 404 得由页面自己决定（`throw createError({ status: 404, ... })`），静态托管那条路另有 `404.html` 兜底。

## 自查

1. 官方给 catch-all 的定义是哪一句？它的匹配范围怎么描述？
2. `/hello/world` 命中 `pages/[...slug].vue` 时，`params.slug` 是什么类型、内容是什么？官方用什么形式展示它？
3. 拼回路径要注意哪一点？为什么？
4. 服务端 `[...].ts` 与 `[...slug].ts` 在取值上差在哪？各自的 key 是什么？
5. 服务端这一侧想拿整条请求路径，官方给的键是什么？
6. "可选的一段"和"任意深度"分别用哪种命名约定？

## 下一篇

116 讲路由的三种结构形态：嵌套路由（父页面里必须再放一个出口）、命名视图（v4.5，一个路由填多个出口、meta 只从默认文件读）与路由组（用 `(marketing)` 这类括号目录组织文件、组名不进 URL，v4.3 起还会自动出现在 `route.meta.groups`）。
