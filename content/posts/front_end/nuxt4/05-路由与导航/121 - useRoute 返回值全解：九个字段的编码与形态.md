---
url: /posts/9454cff20037f9948246e1e76e47740b/
title: 121 - useRoute 返回值全解：九个字段的编码与形态
date: 2026-11-21T10:00:00+08:00
updated: 2026-11-21T10:00:00+08:00
author: cmdragon

summary:
  摘要：官方在 API 一节先给了身份界定："除了动态参数与查询参数之外，`useRoute()` **还提供以下与当前路由相关的计算引用（computed references）**"，随后列出八项：`fullPath`（**与当前路由关联的编码 URL，它包含 path、query 和 hash**）、`hash`（**解码后的、以 # 开头的 URL 片段部分**）、`query`（访问路由查询参数）、`matched`（**规范化匹配路由的数组，带有当前路由位置**）、`meta`（**附着在该记录上的自定义数据**）、`name`（**该路由记录的唯一名称**）、`path`（**URL 的编码路径段**）、`redirectedFrom`（**在最终落到当前路由位置之前所尝试访问的那个路由位置**）；再加上开头就讲过的 `params`（动态参数）就是完整的一组。这一组里有三对最容易搞错。第一对是 **`params` 与 `query`**：官方举例说得很直白——"如果你需要访问路由查询参数（例如路径 `/test?example=true` 里的 `example`），那么你可以使用 `useRoute().query` 而不是 `useRoute().params`"，**`?` 后面的东西永远不在 `params` 里**。第二对是 **编码与解码**：`path` 与 `fullPath` 官方都写成 "encoded"，而 `hash` 写成 "decoded"。第三对是 **`matched` 与 `meta`**：`meta` 是"附着在该记录上的自定义数据"，而 117 引过的合并规则是"如果你使用嵌套路由，所有这些路由的页面元数据会被合并进一个单一的对象"——`route.meta` 是合并后的结果，`matched` 才是那条规范化匹配记录数组。另外 `redirectedFrom` 的官方措辞是"路由位置"（route location），与 `name`/`path` 这类字符串字段不同；而模板侧官方另给了一个入口："在一个 Vue 组件的模板内，你可以使用 `$route` 来访问路由。"

categories:
  - 前端开发

tags:
  - nuxt4
  - useroute
  - 路由参数
  - query
  - meta

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


`route` 上能取到的字段有九个，官方每个只给一句话。但正是这几句话里藏着三对容易用错的东西——参数与查询、编码与解码、合并后的 meta 与匹配链。

## 官方怎么定义这一组字段

> 除了动态参数与查询参数之外，`useRoute()` 还提供以下**与当前路由相关的计算引用（computed references）**。

先记这一句的定性：**它们是计算引用**，也就是说读它们是响应式的（122 的响应式取数就建立在这点上）。

完整清单（官方原文逐条）：

| 字段 | 官方说明 |
| --- | --- |
| `params` | 动态参数（来自 `[id]`、`[[slug]]`、`[...slug]` 这些段） |
| `query` | 访问路由查询参数 |
| `fullPath` | 与当前路由关联的**编码** URL，它包含 path、query 和 hash |
| `hash` | **解码**后的、以 `#` 开头的 URL 片段部分 |
| `path` | URL 的**编码**路径段 |
| `matched` | 规范化匹配路由的数组，带有当前路由位置 |
| `meta` | 附着在该记录上的自定义数据 |
| `name` | 该路由记录的唯一名称 |
| `redirectedFrom` | 在最终落到当前路由位置之前**所尝试访问**的那个路由位置 |

## 第一对：`params` 与 `query`

官方给了一个具体例子来划界：

> 如果你需要访问路由查询参数（例如路径 `/test?example=true` 里的 `example`），那么你可以使用 `useRoute().query` 而不是 `useRoute().params`。

对同一个 URL 拆一遍就清楚了：

```
/users-admins/123?q=new%20search#top
└── params ──┘  └── query ──┘  └hash┘
group=admins    q='new search'  '#top'
id=123
```

| 来源 | 落在 | 依据 |
| --- | --- | --- |
| `[group]` / `[id]` 这类方括号段 | `params` | 官方说"如果你把任何东西放进方括号里，它就会被转换成一个动态路由参数"（114） |
| `?` 之后的键值对 | `query` | 上面那句官话 |
| `#` 之后的片段 | `hash` | "`hash`：解码后的、以 # 开头的 URL 片段部分" |

还要接上 115 那条：`params` 里 catch-all 那一项**是数组**，不是字符串。

## 第二对：编码与解码

官方用词在这三个字段上是不同的，这是排查"我拿到的值怎么带 `%20`"的唯一线索：

| 字段 | 官方用词 | 含义 |
| --- | --- | --- |
| `path` | **encoded** pathname section | 路径段是编码形态 |
| `fullPath` | **encoded** URL … contains path, query and hash | 整体也是编码形态 |
| `hash` | **decoded** hash section | 片段是**解码后**的 |

结论：**别拿 `path` 与浏览器地址栏做字符串相等比较之前，先想清楚编码差异**；而 `hash` 给的是解码后的值。

## 第三对：`matched` 与 `meta`

官方给 `meta` 的定义只有一句——"**附着在该记录上的自定义数据**"；给 `matched` 的定义是——"**规范化匹配路由的数组，带有当前路由位置**"。

它们的关系要接上 117 那条合并规则：

> 如果你使用嵌套路由，所有这些路由的页面元数据会被合并进一个单一的对象。

| 你要问的问题 | 该看 |
| --- | --- |
| 这次导航最终生效的元数据是什么？ | `route.meta`（合并后的单对象） |
| 这个字段是**哪一层**贡献的？ | `route.matched` 这条匹配记录数组 |

同一个键在父层和子层都写了值时，`route.meta` 里只有一个值，但 `matched` 里能看到两条记录各自的 `meta`。

## `name`、`redirectedFrom`：两个"不是字符串"和"是名字"

| 字段 | 形态 | 用法 |
| --- | --- | --- |
| `name` | 字符串，且官方定义为"**该路由记录的唯一名称**" | 112 的推导规则：默认基于它在 `app/pages/` 里的路径生成（`parent-child`、`posts-id`）；判断"是不是这一条路由"比 `path` 稳 |
| `redirectedFrom` | 官方写的是"**路由位置**"（route location），不是字符串 | 想知道"这次最终落点之前尝试过哪里" |

`redirectedFrom` 的存在与 113 讲的 `redirect` 是同一件事的两端：`redirect` 的官方说明是"**重定向发生在任何导航守卫之前，并会用新的目标位置触发一次新的导航**"——那次"新的导航"之前，就是 `redirectedFrom` 记下的那个位置。中间件里 `return navigateTo('/login')` 这类改写落点的写法，同样属于"曾经尝试过的位置"。

判据：**要区分"用户本来想去哪"和"现在在哪"，用 `redirectedFrom` 与当前路由对照。**

## 模板里用 `$route`

官方的口径写在最前面的 note：**"在一个 Vue 组件的模板内，你可以使用 `$route` 来访问路由。"**

两种写法官方都亲自演示过：

```vue [app/pages/[...slug].vue]
<template>
  <p>{{ $route.params.slug }}</p>
</template>
```

```html [~/pages/[slug].vue]
<script setup lang="ts">
const route = useRoute()
const { data: mountain } = await useFetch(`/api/mountains/${route.params.slug}`)
</script>
```

规则很简单：**模板里用 `$route`，脚本里用 `useRoute()`**，指的是同一个东西；脚本里要参与 `computed`、`watch`、取数选项，就必须先拿到 `route` 这个引用。

## 一张"该用哪个字段"的表

| 需求 | 字段 | 注意 |
| --- | --- | --- |
| 路径里的动态段 | `params` | catch-all 那项是**数组**（115）；值来自 URL，判空与转类型都要自己做（114） |
| `?` 后的筛选条件 | `query` | 不在 `params` 里 |
| 锚点 | `hash` | 解码后的值 |
| 参与显示的整路径 | `path` | 是编码形态；**`fullPath` 进模板会水合不匹配**（120：浏览器请求不带片段） |
| 做 key、做比较、做日志 | `fullPath` | 官方示例用法（`key: route => route.fullPath`） |
| 判断是哪条路由 | `name` | URL 形状改了仍然稳定 |
| 元数据（含自定义字段） | `meta` | 是**合并**结果（117）；要类型安全就增强 `#app` 里的 `PageMeta` |
| 元数据来自哪一层 | `matched` | 规范化匹配记录数组 |
| 本次导航的原始目标 | `redirectedFrom` | 一个路由位置，不是字符串 |

## 自查

1. 官方把这组字段定性成什么？这一定性与 122 有什么关系？
2. `/test?example=true` 里的 `example` 从哪个字段取？官方用什么例子说明？
3. 三个字段的编码状态分别是什么？哪一个是解码给出的？
4. `route.meta` 与 `route.matched` 的差别是什么？什么时候必须看后者？
5. `redirectedFrom` 是什么形态？它与 `redirect` 这条记录属性有什么关系？
6. 什么情况下想区分"用户本来想去哪"和"现在在哪"？

## 下一篇

122 讲监听路由变化：官方给的正解是**响应式取数**——`watch` 选项、把 ref 直接塞进 `query`、用 `route.params` 派生的 computed key，以及 `watch: false` 与 `dedupe` 这两处控制。

## 本文依据

- `https://nuxt.com/docs/4.x/api/composables/use-route`
  - 原文（note）："Within the template of a Vue component, you can access the route using `$route`."
  - 原文（API 引导句）："Apart from dynamic parameters and query parameters, `useRoute()` also provides the following computed references related to the current route:"
  - 原文（字段逐条）："`fullPath`: encoded URL associated with the current route that contains path, query and hash"；"`hash`: decoded hash section of the URL that starts with a #"；"`query`: access route query parameters"；"`matched`: array of normalized matched routes with current route location"；"`meta`: custom data attached to the record"；"`name`: unique name for the route record"；"`path`: encoded pathname section of the URL"；"`redirectedFrom`: route location that was attempted to access before ending up on the current route location"
  - 原文（query 与 params 的划界）："If you need to access the route query parameters (for example `example` in the path `/test?example=true`), then you can use `useRoute().query` instead of `useRoute().params`."
  - 原文（Example）：`const route = useRoute()` 与 `await useFetch(\`/api/mountains/${route.params.slug}\`)`（文件标注 `~/pages/[slug].vue`）
  - 原文（Hydration Issues with `route.fullPath`）："Browsers don't send URL fragments (for example `#foo`) when making requests. So using `route.fullPath` to affect the template can trigger hydration issues because this will include the fragment on client but not the server."
- `https://nuxt.com/docs/4.x/directory-structure/app/pages`
  - 原文（合并）："If you are using nested routes, the page metadata from all these routes will be merged into a single object."
  - 原文（Page Metadata）："This data can then be accessed throughout the rest of your app from the `route.meta` object."；示例 `console.log(route.meta.title) // My home page`
  - 原文（类型增强）：`declare module '#app' { interface PageMeta { pageType?: string } }` + `export {}`
  - 原文（Catch-all）："Navigating to `/hello/world` would render:" + `<p>["hello", "world"]</p>`；（Dynamic Routes）："If you place anything within square brackets, it will be turned into a dynamic route parameter."
  - 原文（模板侧 `$route`）：`<template><p>{{ $route.params.slug }}</p></template>`
- `https://nuxt.com/docs/4.x/api/utils/define-page-meta`
  - 原文（`redirect`）："Where to redirect if the route is directly matched. The redirection happens before any navigation guard and triggers a new navigation with the new target location."
- `https://nuxt.com/docs/4.x/api/utils/define-page-meta`（`name`）与 `https://nuxt.com/docs/4.x/directory-structure/app/pages`
  - 对照用（见 112、116）：name 默认基于 `app/pages/` 里的路径生成；`parent-child`、`posts-id` 两个官方样本
- `https://nuxt.com/docs/4.x/api/components/nuxt-page`
  - 对照用（见 092）：`definePageMeta({ key: route => route.fullPath })` 与 `<NuxtPage :page-key="route => route.fullPath" />`
