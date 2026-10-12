---
url: /posts/5ad2eae0508fe8a6ba8a8b4434ef7cbc/
title: 命名路由：name 默认按文件路径生成、以及用它导航
date: 2026-11-18T10:00:00+08:00
updated: 2026-11-18T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：每条自动生成的路由都有名字，官方对 `definePageMeta` 里 `name` 这一项的说明就把规则说清了："**你可以为这个页面的路由定义一个名字。默认情况下，名字是基于它在 `app/pages/` 目录里的路径生成的。**"（类型 `string`）官方的两处推导实例可以直接当规则用：嵌套那组文件（`pages/parent.vue` + `pages/parent/child.vue`）生成的记录是 `{ path: '/parent', component: '~/pages/parent.vue', name: 'parent', children: [{ path: 'child', component: '~/pages/parent/child.vue', name: 'parent-child' }] }`——**子路由的名字是"父名-子名"用连字符拼出来的 `parent-child`，且子记录的 `path` 是不带前导斜杠的 `child`**；另一处是 `<NuxtLink>` 的官方示例，`~/pages/posts/[id].vue` 这个文件对应的名字写作 `posts-id`——**动态段 `[id]` 在名字里变成 `-id` 这样的后缀**。用名字导航的完整写法与渲染结果官方成对给出：`<NuxtLink :to="{ name: 'posts-id', params: { id: 123 } }">Post 123</NuxtLink>` 渲染成 `<a href="/posts/123">Post 123</a>`。这与字符串写法 `to="/about"` 渲染成 `<a href="/about">About page</a>` 是同一条链路的两种输入，而 `<NuxtLink>` 自身的定位是"**它是 Vue Router 的 `<RouterLink>` 组件与 HTML 的 `<a>` 标签二者的即用替代（drop-in replacement）。它会智能判断一个链接是内部的还是外部的，并据此渲染、附带可用的优化（预取、默认属性等）。**"要改匹配形状时官方给的是另一个键：`path`（"如果你有一个比用文件名所能表达的更复杂的模式，你可以定义一个自定义正则表达式"），官方举的冲突案例是"`/test-category` 与 `/1234-post` 这两条路由会同时匹配 `[postId]-[postSlug].vue` 与 `[categorySlug].vue` 两个页面路由"，解法是写 `definePageMeta({ path: '/:postId(\\d+)-:postSlug' })` 只对 `postId` 限定数字。读取侧，`useRoute()` 返回的字段里就有 `name`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 命名路由
  - definepagemeta
  - nuxtlink
  - 路由参数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


链接一般写 `to="/posts/123"`，但还有一种写法是 `:to="{ name: 'posts-id', params: { id: 123 } }"`。这一篇讲这个名字从哪来、长什么样、什么时候值得用它。

## 名字的默认来源：文件路径

官方对 `definePageMeta` 里 `name` 这一项的完整说明只有两句：

> 你可以为这个页面的路由定义一个名字。**默认情况下，名字是基于它在 `app/pages/` 目录里的路径生成的。**

（类型是 `string`。）

所以名字不是你起的，是**推出来的**——理解推导规则，比记住每个名字更省事。

## 规则一：父子用连字符拼接

官方给的那条嵌套路由记录，就是最直接的样本：

```js
[
  {
    path: '/parent',
    component: '~/pages/parent.vue',
    name: 'parent',
    children: [
      {
        path: 'child',
        component: '~/pages/parent/child.vue',
        name: 'parent-child',
      },
    ],
  },
]
```

这一条要看三处：

| 观察 | 结论 |
| --- | --- |
| `pages/parent.vue` → `name: 'parent'` | 父级名字就是路径段本身 |
| `pages/parent/child.vue` → **`name: 'parent-child'`** | **子路由的名字是"父名-子名"，用连字符连接** |
| 子记录 `path: 'child'`（**没有前导斜杠**） | 嵌套记录的 path 是相对父级的；URL 由拼接得出 `/parent/child` |

**名字不是路径。** 路径是 `/parent/child`，名字是 `parent-child`——把路径里的斜杠换成连字符，这就是这一例的形态。

## 规则二：动态段变成后缀

官方在 `<NuxtLink>` 那页给的例子把 `posts/[id].vue` 的名字写成了 `posts-id`：

```vue [app/pages/index.vue]
<template>
  <NuxtLink :to="{ name: 'posts-id', params: { id: 123 } }">
    Post 123
  </NuxtLink>
</template>
```

同一组里官方还给出了渲染结果：

```html [(Renders as) index.html]
<a href="/posts/123">Post 123</a>
```

连起来读：

- `~/pages/posts/[id].vue` 的名字是 **`posts-id`**——**方括号那一段在名字里变成 `-id` 这样的后缀**，而不是留在路径里的 `:id`。
- 用 `{ name, params }` 导航时，`params: { id: 123 }` 会被填进那条 `/posts/:id` 记录，最终 **`href="/posts/123"`**。
- 官方特意用"Renders as"这块展示成品 `<a>`：**渲染出来的仍然是带真 `href` 的链接**（`to="/about"` 那组也一样，渲染成 `<a href="/about">About page</a>`）。

## `name` 与 `params`：对象写法的价值

对比两种输入的差别，就能看出什么时候该用哪一个：

| 写法 | 例子 | 出错方式 |
| --- | --- | --- |
| 字符串 `to` | `to="/posts/123"` | 路径拼错要靠肉眼发现；URL 改了要全站搜替换 |
| 对象 `{ name, params }` | `:to="{ name: 'posts-id', params: { id: 123 } }"` | **参数名与路由段绑定**：`params` 写错 key 时链接不会静默指到别处；URL 形状变了只要记录还在，链接不用改 |

而 `<NuxtLink>` 本身的行为决定了两种写法都不牺牲可爬性。官方的定位原文：

> `<NuxtLink>` 是 Vue Router 的 `<RouterLink>` 组件与 HTML 的 `<a>` 标签**二者的即用替代（drop-in replacement）**。它会智能判断一个链接是内部的还是外部的，并据此渲染、附带可用的优化（预取、默认属性等）。

"预取"那半句在 110 引过：链接进入视口时 Nuxt 会自动提前预取被链接页面的组件与 payload。

## 想改匹配形状：那是 `path` 的活，不是 `name`

这一点要分清，否则会把两个键当一个用。官方给 `path` 的说明是：

> 如果你有一个**比用文件名所能表达的更复杂的模式**，你可以定义一个自定义正则表达式。

官方举的冲突场景非常具体：

> 自定义正则表达式是解决重叠路由之间冲突的一个好办法。例如，`/test-category` 与 `/1234-post` 这两条路由会**同时匹配** `[postId]-[postSlug].vue` 与 `[categorySlug].vue` 两个页面路由。
>
> 为了确保 `[postId]-[postSlug]` 路由里的 `postId` 只匹配数字（`\d+`），可以在 `[postId]-[postSlug].vue` 页面模板里加上下面这段：

```vue [app/pages/[postId]-[postSlug].vue]
<script setup lang="ts">
definePageMeta({
  path: '/:postId(\\d+)-:postSlug',
})
</script>
```

官方接着指向 Vue Router 的 Matching Syntax 要更多例子。

这一节的要点：**`path` 处理的是"哪个 URL 归这个文件"，`name` 处理的是"这条记录叫什么"。** 用 `path` 收窄匹配之后，链接侧可以照旧用字符串，也可以用名字——但此时名字的来源仍然是那条"基于 `app/pages/` 里的路径生成"的默认规则，官方在这个例子里并没有同时改 `name`。`path` 那一套规则（同文件多 URL、`alias` 与 params 的硬约束）下一篇单独展开。

## 覆盖 `name` 之后会发生什么

官方给了定义 `name` 的能力，但没列副作用。可以从默认规则直接推出后果：**名字本来是路径推出来的，一旦你显式写了 `name`，用 `{ name }` 引用这条路由的地方就必须跟着改**——原来写 `posts-id` 的链接，改完名字之后就不该再写 `posts-id`。

所以一条实用纪律：

| 情况 | 建议 |
| --- | --- |
| 没有特殊需要 | **不要动 `name`**，让它按路径生成，链接用字符串 `to` 最直观 |
| 一处 URL 会被很多组件链接引用，且 URL 常变 | 显式 `name` + `{ name, params }` 写法，把变化收在页面文件里 |
| 要解决路由重叠 | 用 `path`（正则），不要试图靠改 `name` 解决——名字不参与匹配 |

## 读回来：`route.name`

`useRoute()` 返回的字段里就包含 `name`，所以判断"当前是哪条路由"有两种口径：

| 判据 | 用途 |
| --- | --- |
| `route.name` | 与记录绑定，URL 形状改了仍然稳定 |
| `route.path` / `route.fullPath` | 与地址栏绑定，含 query 与片段（注意 `fullPath` 渲染进模板会造成水合不匹配） |

## 自查

1. 官方对 `name` 的默认生成规则是哪一句？
2. `pages/parent/child.vue` 生成的记录里 `name` 和 `path` 分别是什么？为什么 `path` 没有前导斜杠？
3. `posts/[id].vue` 的路由名写作什么？`[id]` 这一段在名字里变成了什么？
4. 用 `{ name, params }` 写链接的渲染结果是什么？官方用哪块内容展示它？
5. 想解决 `[postId]-[postSlug].vue` 与 `[categorySlug].vue` 的抢匹配，应该改哪个键、写成什么？改 `name` 有用吗？
6. 什么情况下建议不要显式设 `name`？

## 下一篇

113 讲 `path` 与 `alias`：同一个文件挂多个 URL 的规则，以及官方那条硬约束——`alias` 与 `path` **必须共享同样的 params**。

## 本文依据

- `https://nuxt.com/docs/4.x/api/utils/define-page-meta`
  - 原文（Type 签名）：`interface PageMeta { ... name?: string; path?: string; props?: RouteRecordRaw['props']; alias?: string | string[]; groups?: string[]; ... }`
  - 原文（`name`）："You may define a name for this page's route. By default, name is generated based on path inside the `app/pages/` directory."；Type `string`
  - 原文（`path`）："You may define a custom regular expression if you have a more complex pattern than can be expressed with the file name."
  - 原文（Using a Custom Regular Expression）："A custom regular expression is a good way to resolve conflicts between overlapping routes, for instance: The two routes '/test-category' and '/1234-post' match both `[postId]-[postSlug].vue` and `[categorySlug].vue` page routes."；"To make sure that we are only matching digits (`\d+`) for `postId` in the `[postId]-[postSlug]` route, we can add the following to the `[postId]-[postSlug].vue` page template"；示例 `definePageMeta({ path: '/:postId(\\d+)-:postSlug' })`（文件标注 `app/pages/[postId]-[postSlug].vue`）；"For more examples see Vue Router's Matching Syntax"
- `https://nuxt.com/docs/4.x/api/components/nuxt-link`
  - 原文（note）："`<NuxtLink>` is a drop-in replacement for both Vue Router's `<RouterLink>` component and HTML's `<a>` tag. It intelligently determines whether the link is *internal* or *external* and renders it accordingly with available optimizations (prefetching, default attributes, etc.)"
  - 原文（Internal Routing）：`<NuxtLink to="/about">About page</NuxtLink>` 与"Renders as"块 `<!-- (Vue Router & Smart Prefetching) --> <a href="/about">About page</a>`
  - 原文（Passing Params to Dynamic Routes）："In this example, we pass the `id` param to link to the route `~/pages/posts/[id].vue`."；`<NuxtLink :to="{ name: 'posts-id', params: { id: 123 } }">Post 123</NuxtLink>`；渲染结果 `<a href="/posts/123">Post 123</a>`
- `https://nuxt.com/docs/4.x/directory-structure/app/pages`
  - 原文（Nested Routes 生成的记录）：`{ path: '/parent', component: '~/pages/parent.vue', name: 'parent', children: [{ path: 'child', component: '~/pages/parent/child.vue', name: 'parent-child' }] }`
  - 原文（Dynamic Routes）："If you place anything within square brackets, it will be turned into a dynamic route parameter. You can mix and match multiple parameters and even non-dynamic text within a file name or directory."
- `https://nuxt.com/docs/4.x/getting-started/routing`
  - 原文（Navigation）："It renders an `<a>` tag with the `href` attribute set to the route of the page."；"When a `<NuxtLink>` enters the viewport on the client side, Nuxt will automatically prefetch components and payload (generated pages) of the linked pages ahead of time, resulting in faster navigation."
  - 原文（生成的路由记录）：`{"path": "/posts/:id", "component": "pages/posts/[id].vue"}`
- `https://nuxt.com/docs/4.x/api/composables/use-route`
  - 对照用（见 120）：`useRoute()` 返回字段含 `name`、`path`、`fullPath`（`fullPath` 渲染进模板会造成水合不匹配）
