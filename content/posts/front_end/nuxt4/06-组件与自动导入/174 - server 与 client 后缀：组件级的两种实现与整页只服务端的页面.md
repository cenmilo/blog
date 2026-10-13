---
url: /posts/5358a3ed428120d38f76ac7c493c1250/
title: 174 - server 与 client 后缀：组件级的两种实现与整页只服务端的页面
date: 2026-12-09T09:00:00+08:00
updated: 2026-12-09T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：098、161、165 分别讲过几个后缀，这一篇把**后缀这一层**并成一张表，并补上前几篇没展开的**页面级**后缀。**组件侧**：`.client.vue`（"If a component is meant to be rendered only client-side, you can add the `.client` suffix to your component."）、`.server.vue`（"Register server-only components with the `.server` suffix and use them anywhere in your application automatically."）、`.global.vue`（选择性全局注册的三条入口之一）、以及**目录而非后缀**的 `~/components/islands/`（"By default, component islands are scanned from the `~/components/islands/` directory"）。**页面侧是这一篇的新内容**，官方各给一句定义：Client-Only Pages——"**You can define a page as client only by giving it a `.client.vue` suffix. None of the content of this page will be rendered on the server.**"；Server-Only Pages——"**You can define a page as server only by giving it a `.server.vue` suffix. While you will be able to navigate to the page using client-side navigation, controlled by `vue-router`, it will be rendered with a server component automatically, meaning the code required to render the page will not be in your client-side bundle.**" 页面级同样挂着那条结构要求："**Server-only pages must have a single root element. (HTML comments are considered elements as well.)**" 以及一句预取连带："**Navigating to a server-only page needs a request to render it, so when a `<NuxtLink>` to one is prefetched, Nuxt prefetches that response as well.**"。**`.server` + `.client` 成对时那个组件不是岛屿**："In that case the component is not an island: the client half hydrates normally."；而要拿到岛屿的加载与错误事件，得放弃后缀改用组件本身："Server components rendered via the auto-generated wrapper **do not expose load and error events**; use `<NuxtIsland>` directly if you need its `error` event and `refresh()` method"。

categories:
  - 前端开发

tags:
  - nuxt4
  - 后缀
  - 服务端组件
  - 页面
  - 岛屿

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


我一直以为 `.server.vue` 只能加在 `app/components/` 里。翻页面目录那一节才发现，**同样的后缀加在 `app/pages/` 上是另一回事**：整页都不进客户端包，但导航仍然由 vue-router 控制。这一篇把后缀这一层一次摆平。

## 后缀与前缀：一张表先收拢

| 写法 | 位置 | 效果 | 官方限制句 |
| --- | --- | --- | --- |
| `Lazy` **前缀** | 模板里的标签名 | 动态导入、单独 chunk | "still load eagerly unless conditionally rendered"（161） |
| `.client.vue` | `app/components/` | 只在客户端渲染 | "only works with Nuxt auto-imports and `#components` imports"（165） |
| `.server.vue` | `app/components/` | 只服务端渲染（岛屿） | "must have a single root element. (HTML comments are considered elements as well.)" |
| `.global.vue` | `app/components/` | 全局注册 | "each global component is rendered in a separate chunk"（159） |
| `islands/` | `app/components/` 下**目录** | 注册为岛屿，用 `<NuxtIsland name="…">` 渲染 | "By default, component islands are scanned from the `~/components/islands/` directory." |
| `(分组)/` | `app/components/` 下目录 | 只分组、不改名（157、158） | — |
| `.client.vue` | **`app/pages/`** | **整页内容都不在服务端渲染** | 见下一节 |
| `.server.vue` | **`app/pages/`** | **整页由服务端组件渲染，页面代码不进客户端包** | 见下一节 |

最后一行是这一篇的重点。

## 页面级的两个后缀

官方在 `app/pages` 那节各给一句定义，措辞差别很大，值得并排读。

**Client-Only Pages**：

> You can define a page as [client only](directory-structure/app/components#client-components) by giving it a `.client.vue` suffix. **None of the content of this page will be rendered on the server.**

**Server-Only Pages**：

> You can define a page as [server only](directory-structure/app/components#server-components) by giving it a `.server.vue` suffix. **While you will be able to navigate to the page using client-side navigation, controlled by `vue-router`, it will be rendered with a server component automatically, meaning the code required to render the page will not be in your client-side bundle.**

第二句里三个信息一次给完：

| 信息 | 含义 |
| --- | --- |
| **仍然能客户端导航** | 后缀不改变路由表，`<NuxtLink>` 照点、中间件照跑 |
| **自动用服务端组件渲染** | 不需要你写 `<NuxtIsland>`，页面被自动包起来 |
| **渲染这页所需的代码不进客户端 bundle** | 这才是省下来的东西 |

**这三条合起来解释了一个我一开始想错的问题**：既然整页只在服务端渲染，那还叫"客户端导航"吗？答案是叫——**导航与水合是两件事**，服务端页面省的是渲染代码，不是路由能力。

两个连带事实官方也写在这一节：

> **Server-only pages must have a single root element. (HTML comments are considered elements as well.)**

> Navigating to a server-only page needs a request to render it, so **when a `<NuxtLink>` to one is prefetched, Nuxt prefetches that response as well.**

后者正是 169 那句"预取只救整页 `.server.vue` 这种情况"的出处——那里还跟了一条 note："Islands used within a page are still fetched when they mount."**所以页面级后缀是预取唯一能覆盖的情况。**

## 页面级与 `ssr: false` 的区别

这两条都让"服务端不渲染内容"，但方向相反，很容易选错：

| | `.client.vue` 页面 | `.server.vue` 页面 | `ssr: false`（165） |
| --- | --- | --- | --- |
| 服务端 HTML | 没有 | **有**（就是页面本身） | 没有 |
| 渲染代码在客户端包里 | 在（要水合才出内容） | **不在** | 在 |
| 交互 | 有 | **静态**（除非 `nuxt-client`，171） | 全部 |
| 每次进入要不要请求渲染 | 首次请求 HTML，之后导航不请求渲染 | **每次导航都要一次渲染请求** | 否 |

选哪条其实是问："我不想下发的是渲染代码，还是不想在服务端跑渲染？"**`.server.vue` 页面省前者、`.client.vue` 页面与 `ssr: false` 省后者。**

## 成对写法不是岛屿：三种实现方式的边界

`.server` 与 `.client` 同名成对时，官方明确否掉了"岛屿"这个理解：

> In this case, the `.server` + `.client` components are two 'halves' of a component and can be used in advanced use cases for separate implementations of a component on server and client side.

而在服务端组件那一页补的是关键半句：

> You can also pair a `.server.vue` component with a `.client.vue` component of the same name for separate server and client implementations. **In that case the component is not an island: the client half hydrates normally.**

把三种"只要服务端渲染"的做法分开：

| 做法 | 是不是岛屿 | 客户端能做什么 |
| --- | --- | --- |
| `.server.vue` 单独 | ✅ 是（`<NuxtIsland>` 包装） | 静态 HTML；`nuxt-client` 需开 `selectiveClient`（171） |
| `.server.vue` + `.client.vue` 成对 | ❌ 不是 | 挂载后由客户端那份接管、正常水合 |
| `~/components/islands/` + `<NuxtIsland>` 手动渲染 | ✅ 是 | 可加 `lazy`、`#fallback`、`error`、`refresh()` |

**第三行是唯一能拿到事件的写法**，官方在限制清单里点名：

> Server components rendered via the auto-generated wrapper **do not expose load and error events**; use `<NuxtIsland>` directly if you need its `error` event and `refresh()` method.

（这两个入口在 107 逐项列过：`refresh()` 是"force refetch the server component by refetching it"，`error` 在"`NuxtIsland` fails to fetch the new island"时发出。）

## 回顾一句：后缀组件在服务端经历了什么

106 讲透了三件事，这里只留结论式的一句对照，方便判断成对写法与单 `.server` 的差别从哪来：

> - creates a **new, isolated Vue app** on the server to render just that component
> - creates an 'island context' that you can access via `nuxtApp.ssrContext.islandContext` inside the island
> - runs your plugins again, unless they set `env: { islands: false }` (object-syntax plugins)

**成对写法不走这套**——它客户端那份是普通水合，所以共享状态、路由、插件都按常规生效；代价是那些库的代码也回到客户端包里了。

## 三个我会踩的意外

| 意外 | 依据 |
| --- | --- |
| 服务端页面在根前写了个 HTML 注释 → 违反单根要求 | 括号那句 "HTML comments are considered elements as well."（组件与页面两处同句） |
| 以为页面级 `.server.vue` 会省掉导航请求 | 官方："Navigating to a server-only page **needs a request to render it**"，只能靠预取提前 |
| 给 `.server` 组件写 `@error` 或调 `refresh()` 拿不到 | 自动生成的包装不暴露 load/error，改直接用 `<NuxtIsland>` |

## 自查

1. 组件侧四个后缀与一个目录分别让组件变成什么？哪些限制句必须同时记住？
2. 页面级 `.client.vue` 与 `.server.vue` 的官方定义句分别是什么？后者那句里的三个信息是什么？
3. "能客户端导航"与"只在服务端渲染"为什么不矛盾？
4. 页面级后缀与 `ssr: false` 各自省下什么？用表格的四个维度回答。
5. 成对写法为什么不是岛屿？它换来的代价是什么？
6. 想要岛屿的加载与错误事件，必须改用哪个组件？官方哪句话说明自动包装没有这些事件？
7. 服务端页面进入时的网络请求能被优化吗？靠什么？覆盖范围有什么限制？
8. 单根要求那句括号说明针对什么常见写法？

## 下一篇

175 讲 `<DevOnly>`：它的内容为什么在生产构建里不存在、`#fallback` 那版替代要注意什么（官方提醒要用 `nuxt preview` 测），以及性能指南列出的几个"包里多了东西"的常见原因。

## 本文依据

- `https://nuxt.com/docs/4.x/directory-structure/app/pages`
  - 原文（Client-Only Pages）："You can define a page as [client only](https://nuxt.com/docs/4.x/directory-structure/app/components#client-components) by giving it a `.client.vue` suffix. None of the content of this page will be rendered on the server."
  - 原文（Server-Only Pages）："You can define a page as [server only](https://nuxt.com/docs/4.x/directory-structure/app/components#server-components) by giving it a `.server.vue` suffix. While you will be able to navigate to the page using client-side navigation, controlled by `vue-router`, it will be rendered with a server component automatically, meaning the code required to render the page will not be in your client-side bundle."；warning "Server-only pages must have a single root element. (HTML comments are considered elements as well.)"；"Navigating to a server-only page needs a request to render it, so when a `<NuxtLink>` to one is prefetched, Nuxt prefetches that response as well." + read-more 指向 `guide/concepts/server-components#the-client-navigation-round-trip`
- `https://nuxt.com/docs/4.x/directory-structure/app/components`
  - 原文（后缀四处）："If a component is meant to be rendered only client-side, you can add the `.client` suffix to your component."；note "This feature only works with Nuxt auto-imports and `#components` imports. Explicitly importing these components from their real paths does not convert them into client-only components."；important "`.client` components are rendered only after being mounted. To access the rendered template using `onMounted()`, add `await nextTick()` in the callback of the `onMounted()` hook."；"Register server-only components with the `.server` suffix and use them anywhere in your application automatically."；"Standalone server components will always be rendered on the server, also known as Islands components."；"Server-only components use `<NuxtIsland>` under the hood, meaning that `lazy` prop and `#fallback` slot are both passed down to it."；"In this case, the `.server` + `.client` components are two 'halves' of a component and can be used in advanced use cases for separate implementations of a component on server and client side." + `Comments.client.vue`/`Comments.server.vue`；"You can also selectively register some components globally by placing them in a `~/components/global` directory, or by using a `.global.vue` suffix in the filename."
- `https://nuxt.com/docs/4.x/guide/concepts/server-components`
  - 原文（页面成对与限制）："You can also pair a `.server.vue` component with a `.client.vue` component of the same name for separate server and client implementations. In that case the component is not an island: the client half hydrates normally."；warning "Server components (and islands) must have a single root element. (HTML comments are considered elements as well.)"；"Server components rendered via the auto-generated wrapper do not expose load and error events; use `<NuxtIsland>` directly if you need its `error` event and `refresh()` method."；"Each nested island adds extra overhead, so be careful when nesting islands within other islands."；机制三句（isolated Vue app / island context / runs your plugins again, unless they set `env: { islands: false }`）
- `https://nuxt.com/docs/4.x/api/components/nuxt-island`
  - 对照用（见 107、170）："By default, component islands are scanned from the `~/components/islands/` directory. So the `~/components/islands/MyIsland.vue` component could be rendered with `<NuxtIsland name=\"MyIsland\" />`."；Ref `refresh()` — "force refetch the server component by refetching it."；Event `error` — "emitted when `NuxtIsland` fails to fetch the new island."
- `https://nuxt.com/docs/4.x/api/components/client-only` 与 `https://nuxt.com/docs/4.x/guide/concepts/rendering`
  - 对照用（见 165）：`<ClientOnly>` 与 `ssr: false` 两条"不在服务端渲染"的路径及其代价
