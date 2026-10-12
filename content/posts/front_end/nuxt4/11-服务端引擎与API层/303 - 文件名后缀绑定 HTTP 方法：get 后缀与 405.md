---
url: /posts/7e27407329fe4430712c5cff66ef0f3a/
title: 文件名后缀绑定 HTTP 方法：get 后缀与 405
date: 2027-01-21T09:00:00+08:00
updated: 2027-01-21T09:00:00+08:00
author: cmdragon

summary:
  摘要：上一篇（302）说过服务端路由的路径来自文件名，这一篇讲文件名另一层含义：后缀可以绑定 HTTP 方法。`test.get.ts` 只处理 GET、`test.post.ts` 只处理 POST，同一个 `/test` 端点按方法分发到不同文件；用 GET 之外或不匹配的方法访问，返回 405。还有一条容易踩的 note：`readBody` 在 GET 请求里会直接抛 405，所以"要读请求体"的 handler 后缀得写成 `.post` 之类。末尾补目录形态 `index.get.ts`。

categories:
  - 前端开发

tags:
  - nuxt4
  - HTTP方法
  - 服务端路由
  - 405
  - server
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`server/api` 里的文件名，最后一段可以不是 `.ts` 而是一层"方法限定"。这是 Nitro 服务端的约定，官方在 server 那页 "Matching HTTP Method" 讲清了。

## 后缀绑方法

官方原话：

> Handle file names can be suffixed with `.get`, `.post`, `.put`, `.delete`, ... to match request's HTTP Method.

两个文件示例：

```ts [server/api/test.get.ts]
export default defineEventHandler(() => 'Test get handler')
```

```ts [server/api/test.post.ts]
export default defineEventHandler(() => 'Test post handler')
```

同一个 `/test` 端点，官方给的匹配规则：

- **GET** 方法：返回 `Test get handler`
- **POST** 方法：返回 `Test post handler`
- **其他任何方法**：返回 **405 error**

也就是说后缀决定这个文件只接哪种方法的请求。不写方法后缀（只 `.ts`），就是所有方法都进这一个 handler。

## 一条必须知道的坑：GET 里读 body 会抛 405

方法后缀不只是"分类"，它和你"要不要读请求体"直接挂钩。官方 note：

> We are using `submit.post.ts` in the filename only to match requests with `POST` method that can accept the request body. When using `readBody` within a GET request, `readBody` will throw a `405 Method Not Allowed` HTTP error.

读法：GET 请求按约定不该带请求体，所以在 GET handler 里调 `readBody` 会被判成"方法不对"、抛 405。要读 body 的接口（提交表单、创建资源），文件名后缀就写 `.post`，别写成 GET 或裸 `.ts`。这条把"方法后缀"和"body 处理"绑在一起，是新手最常混的一处。

配套的读 body 示例官方也给在 `submit.post.ts` 里：

```ts [server/api/submit.post.ts]
export default defineEventHandler(async (event) => {
  const body = await readBody(event)
  return { body }
})
```

## 目录形态：index.方法.ts

不想一个文件堆太多，或要建 API 命名空间，可以用"目录 + `index.[method].ts`"。官方给的例子：

```ts [server/api/foo/index.get.ts]
export default defineEventHandler((event) => {
  // 处理 GET /api/foo
})
```

```ts [server/api/foo/index.post.ts]
export default defineEventHandler((event) => {
  // 处理 POST /api/foo
})
```

```ts [server/api/foo/bar.get.ts]
export default defineEventHandler((event) => {
  // 处理 GET /api/foo/bar
})
```

`index.get.ts` 对应目录本身的路径（`/api/foo`），`bar.get.ts` 对应子路径（`/api/foo/bar`）。官方说"这种组织方式 useful to create API namespaces"——把一个资源的多方法、多子路由拆进目录。

## 自查

- 一个 handler 只该接某方法 → 文件名带 `.get`/`.post`/`.put`/`.delete`；不带后缀则全方法都进它。
- 要 `readBody` 的接口 → 后缀写 `.post` 之类，别在 GET 里读 body（官方明说会抛 405）。
- 一个资源多方法/多子路径 → 用 `目录/index.[method].ts` + `子文件.[method].ts` 组织。
- 405 是"方法不匹配"，和 404"路径不存在"是两回事，报错时先看是不是后缀写错。

## 下一篇

下一篇 304 讲 `defineEventHandler` 本身：handler 的三种合法返回（可 JSON 序列化的值、Response、ReadableStream），以及"return 值"和"`event.node.res.end()` 自己写响应"两种方式的区别。

## 本文依据

- server 目录（`directory-structure/server`）：Matching HTTP Method 一节（"Handle file names can be suffixed with `.get`, `.post`, `.put`, `.delete`, ... to match request's HTTP Method."、`test.get.ts`/`test.post.ts` 示例、GET/POST/其他→405 三条、目录形态 `index.get.ts`/`index.post.ts`/`bar.get.ts` 示例与"create API namespaces"）；Body Handling 一节（`submit.post.ts` 与 `readBody`、note "When using `readBody` within a GET request, `readBody` will throw a 405 Method Not Allowed HTTP error."）。
