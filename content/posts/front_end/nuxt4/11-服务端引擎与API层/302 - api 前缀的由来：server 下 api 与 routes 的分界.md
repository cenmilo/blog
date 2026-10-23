---
url: /posts/02e7ca954e096ce0eeab2d1075c2dc18/
title: 302 - api 前缀的由来：server 下 api 与 routes 的分界
date: 2027-01-20T11:00:00+08:00
updated: 2027-01-20T11:00:00+08:00
author: cmdragon

summary:
  摘要：为什么 server/api/hello.ts 的访问路径是 /api/hello、而 server/routes/bonjour.ts 是 /bonjour？这一篇讲这个前缀到底怎么来的：api/ 目录里的文件被自动加一层 /api 前缀（约定把"给应用用的接口"归拢），routes/ 目录则按文件名直接成路径、不加前缀。再把路径怎么从文件名推出来讲全：动态段 [name] 用 getRouterParam 读、catch-all [...] 用 event.context.params、方法后缀 .get/.post 会按 HTTP 方法匹配且其他方法返回 405。最后给一条选择判据：什么时候用 api、什么时候用 routes。

categories:
  - 前端开发

tags:
  - nuxt4
  - api前缀
  - 服务端路由
  - 动态路由
  - http方法
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


上一篇把 `server/` 六个子目录铺开，这一篇把其中最容易被问的一个具体讲透：`/api` 这个前缀是从哪冒出来的。

## 前缀怎么来的：api 目录 vs routes 目录

官方对两个目录各给了规则：

> Files inside the `~~/server/api` are automatically prefixed with `/api` in their route.

> To add server routes without `/api` prefix, put them into `~~/server/routes` directory.

对照结构图最直观：

```bash
-| server/
---| api/
-----| hello.ts      # /api/hello
---| routes/
-----| bonjour.ts    # /bonjour
```

- 放 `server/api/hello.ts` → 路径自动是 **`/api/hello`**（多出来的 `/api` 是目录名带的约定前缀）。
- 放 `server/routes/bonjour.ts` → 路径是 **`/bonjour`**（文件名直接成路径，不加前缀）。

所以"api 和 routes 的区别"这一层就是**要不要那层 `/api`**。除此之外两者写法、handler 模型、动态段能力完全一样。官方示例给了 routes 的：

```ts [server/routes/hello.ts]
export default defineEventHandler(() => 'Hello World!')
```

> 访问 `/hello` 就在 `http://localhost:3000/hello`。

## 什么时候用 api、什么时候用 routes

判据很实用：

- 这是**给你自己前端应用调用的数据接口** → 放 `server/api/`，`/api` 前缀天然把"接口"和"页面 URL"分开，前端 `useFetch('/api/...')`、`$fetch('/api/...')` 一眼看出是打后端。
- 这条路径**本身要公开、或不想要 /api 这层** → 放 `server/routes/`。典型：要响应一个固定外部路径（比如 webhook 得挂在 `/github-webhook` 这种约定地址），或做不带 `/api` 的端点。

一句提醒（note 原话）：server routes 的动态路由能力**不如 pages 完整**（"server routes do not support the full functionality of dynamic routes as pages do"）。服务端路由做复杂路径匹配时别期望它和页面路由一样全。

## 路径怎么从文件名推：动态段与 catch-all

前缀之外，文件名本身也参与路径。官方 Recipes 给了两档：

**动态参数 `[name]`**：文件名里方括号是动态段，用 `getRouterParam` 读：

```ts [server/api/hello/[name].ts]
export default defineEventHandler((event) => {
  const name = getRouterParam(event, 'name')
  return `Hello, ${name}!`
})
```

访问 `/api/hello/nuxt` → `Hello, nuxt!`。官方 tip 也建议配合 `getValidatedRouterParams` 加 schema 校验。

**catch-all `[...]`**：兜底所有未匹配路径。方括号里点三种形态：

```ts [server/api/foo/[...].ts]
export default defineEventHandler((event) => {
  // event.context.path 取整条路径：'/api/foo/bar/baz'
  // event.context.params._ 取匹配到的片段：'bar/baz'
  return `Default foo handler`
})
```

给 catch-all 起名字用 `[...slug].ts`，就用 `event.context.params.slug` 取片段。注意服务端 catch-all 的 key 是 `_`（无名时）或你给的名字——这和 pages 的 `[...slug]` 数组形态不同，别照页面那套记。

## 方法后缀：.get / .post 怎么影响路由

文件名还能用 HTTP 方法后缀限定：

> Handle file names can be suffixed with `.get`, `.post`, `.put`, `.delete`, ... to match request's HTTP Method.

官方给的匹配表很清楚（`/test` 这个端点，`test.get.ts` 与 `test.post.ts` 两个文件）：

- GET → `Test get handler`
- POST → `Test post handler`
- 其他方法 → **返回 405 error**

一条容易忽略的 note：`readBody` 只在能接收请求体的方法（如 POST）下用；在 GET 请求里用 `readBody` 会抛 `405 Method Not Allowed`。所以"要读 body 的 handler"文件名要带 `.post` 之类后缀，别写成 GET。

另外目录形态也支持：`server/api/foo/index.get.ts`、`index.post.ts` 这样把命名空间拆进目录。

## 前端怎么调

放好之后，前端"通用地"调它——api 或 routes 都一样：

```vue [app/pages/index.vue]
<script setup lang="ts">
const { data } = await useFetch('/api/hello')
</script>
```

300 讲的 `$fetch` 服务端直调在这里也适用：SSR 阶段 `useFetch('/api/hello')` 命中的是那个 handler 函数本身，不多一次网络往返。

## 自查

- 想要 `/api` 前缀、当作应用数据接口 → `server/api/`；想要干净路径（webhook 等）→ `server/routes/`。
- 动态段 `[name]` 用 `getRouterParam(event,'name')`；catch-all `[...]` 用 `event.context.params._`（无名）或 `event.context.params.slug`（`[...slug]`），整路径 `event.context.path`。
- 方法后缀 `.get/.post` 会把 handler 限到对应方法、其他方法返回 405；读 body 的 handler 别挂成 GET。
- 别把服务端路由当成页面路由——它动态段能力更弱。

## 下一篇

下一篇 303 继续服务端这一层：讲 Nitro 的服务端路由怎么组织、以及 API 与页面路由的分界。
