---
url: /posts/5b09d6d3d7584335e43e5a21bc471e6e/
title: 741 - 目录边界（4）：server／api 的地址前缀与方法后缀
date: 2026-10-20T12:00:00+08:00
updated: 2026-10-20T12:00:00+08:00
author: cmdragon

summary:
  摘要：`server/api/console/metrics.get.ts` 这个文件里没有任何一行写着它的地址和允许的方法，但 `GET /api/console/metrics` 就是能用、别的都不给用——地址由目录和文件名拼出来，方法由后缀认。这一站把这张注册表逐行读出来，再用真实请求打一圈：文档承诺的 405 在产物上并没有出现，掉的是 404；而 `$fetch` 这一侧认方法、不认地址。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 服务端路由
  - HTTP 方法
  - Nitro
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 屏幕上看得到的一条接口

这个仓库的 `server/api/` 下有十七个文件，`server/routes/` 下有三个。随便挑一个：

```bash [真实文件名，不是造的]
server/api/count.get.ts
server/api/docs/[slug].get.ts
server/api/auth/logout.post.ts
server/api/admin/users/[userId]/sessions.delete.ts
server/routes/robots.txt
```

文件里只写了一个默认导出的处理函数：

```ts [server/api/count.get.ts 的形状]
export default defineEventHandler(() => {
  // ...
})
```

没有配置文件声明"`GET /api/count` 归这个文件管"。这一站就是把这条隐式的映射一层层读出来：先看注册表里它长什么样，再看真实请求打上去得到什么。

## 三个词先说清

**注册表**：Nitro 把扫到的处理函数汇成的那张表。Nuxt 还会把它同时写成类型（`.nuxt/types/nitro-routes.d.ts` 里的 `InternalApi`），所以这张表既是路由的真相，也是 `$fetch` 能拿到返回类型的来源。

**方法后缀**：文件名结尾那一段 `.get`、`.post`、`.put`、`.delete`。文档还写了别的（`...` 那行是省略写法），本轮量了认与不认两种情况。

**兜底渲染**：如果请求在路由表里没被接住，它会一路掉到渲染页面的那一位——服务端会试着把那个地址当一个页面来渲，渲不出来就给你一张 404。这一位是后面那个 405 疑问的关键。

## 谁带 /api，谁不带

文档那句很直白：`server/api` 里的文件**自动带上 `/api` 前缀**；要加不带前缀的地址，放 `server/routes`。本轮数出来的注册表分三堆：

```text
/api/**          17 条    ← server/api 下十七个文件，一个文件一条
/llms.txt        1 条      ┐
/robots.txt      1 条      ├ server/routes 下三个文件，不带前缀
/sitemap.xml     1 条      ┘
/__nuxt_error    1 条      ┐ 框架自己的内部路由，不是你写的
/__nuxt_island/** 1 条     ┘
合计             22 条路径键、22 个方法位
```

同一条节里还有一句容易被跳过的注：服务端路由对动态路由的支持不如 pages 完整。所以别拿页面那套目录规则去想 `server/routes`。

## 注册表里那一格长什么样

`.nuxt/types/nitro-routes.d.ts` 里真实的四行（左边是地址，右边是方法）：

```ts [真实注册，逐字符来自生成物]
'/api/admin/users/:userId/sessions': {
  'delete': Simplify<Serialize<Awaited<ReturnType<typeof import('../../server/api/admin/users/[userId]/sessions.delete').default>>>>
}
'/api/count': {
  'get': Simplify<Serialize<Awaited<ReturnType<typeof import('../../server/api/count.get').default>>>>
}
```

对上三条规矩：目录层级原样进地址（`admin/users/` 一位不落）；方括号变成 `:userId` 这种参数；后缀变成方法名那一格。

再补三条本轮用探针量出来的（探针文件读完就删）：

```ts [探针的注册结果]
'/api/probe-method': {
  'get':  ...
  'post': ...
}
'/api/probe-ns': {
  'get':  ...
  'post': ...
}
'/api/probe-ns/bar': {
  'get': ...
}
```

`probe-method.get.ts` 与 `probe-method.post.ts` 两个文件汇成**一个路径键、两个方法位**；`index.get.ts`＋`index.post.ts` 这种目录写法注册地址是 `'/api/probe-ns'`（不带 `index`），同目录的 `bar.get.ts` 是 `'/api/probe-ns/bar'`——文档那段"用 `index.[method].ts` 组 API 命名空间"就是这个样子。

## 未知后缀不会报错，只会改地址

把一个不认识的后缀放进去：`server/api/probe-teapot.foo.ts`。注册结果是：

```ts [未知后缀的注册]
'/api/probe-teapot.foo': {
  'default': ...
}
```

**`.foo` 不是方法匹配器，它留在了地址里**，而方法那一格变成 `default`——任何方法都应答。这不是推测：请求打上去，`GET`、`PUT`、`HEAD` 三种都是 `200`，body 都是同一句 `{"handler":"unknown-suffix"}`。

现成的对照就在这三个文件上：`server/routes/robots.txt`、`llms.txt`、`sitemap.xml` 都没有方法后缀，注册表里它们的方法位同样是 `default`。想限方法，后缀必须写准；写错了没人拦你，只是地址跟着变了。

## 产物上打一圈：405 没出现

文档那句是：GET 命中 get 处理函数、POST 命中 post 处理函数、**其它方法返回 405 错误**。用真请求打在产物服务上（e2e 那一层自己拉起 `npm run build` 的产物），十条读数是：

```text
GET   /api/probe-method      → 200 | allow=- | application/json | {"handler":"get"}
POST  /api/probe-method      → 200 | allow=- | application/json | {"handler":"post"}
PUT   /api/probe-method      → 404 | allow=- | application/json | { "error": true, ... "statusCode": 404
PATCH /api/probe-method      → 404 | allow=- | application/json | 同上
HEAD  /api/probe-method      → 404 | allow=- | application/json |（没有 body）
DEL   /api/probe-ns          → 404 | allow=- | 同上
PUT   /api/count             → 404 | allow=- | 同上
GET   /api/probe-teapot.foo  → 200 | allow=- | application/json | {"handler":"unknown-suffix"}
GET   /api/probe-teapot      → 404 | allow=- |（这个地址压根没注册）
GET   /api/logout-no-such    → 404 | allow=- |
```

三件事值得停下来。第一，未匹配的方法和未注册的地址给的是同一个 404，`allow` 头一次都没出现，405 在这套产物里没读到。第二，`PUT /api/count` 也是 404——这个文件是站里真实接口，不是探针，所以那个结果与探针无关。第三，`HEAD` 不算 `GET`：一个 `.get.ts` 处理函数不接 `HEAD` 请求。健康检查、探活脚本用 `HEAD` 打接口时要记住这点。

顺带一条读数：`GET /api/count` 是 `200`，但它的 `content-type` 是 `text/html`、body 是 `10`；返回对象的几个是 `application/json`。文档只说处理函数可以直接返回 JSON 数据、返回 Promise，或者用 `event.node.res.end()` 自己发响应——**返回一个标量时按什么类型发出去，文档没写**，上面这行是这一轮读到的。

## 为什么会掉到 404

同一趟请求，服务端日志里出现的是这一行：

```text
[WebServer] [VUE_ROUTER_R0004] No match found for location with path "/api/probe-method"
```

方法没被接住的请求没有停在服务端路由表里，它继续往下走，走到"按地址渲一张页面"那一位；那一位在这个地址上找不到任何页面，于是给出的是它自己的错误体（`statusMessage` 是 `Page not found: /api/probe-method`）。404 与 `Page not found` 是页面渲染通道的话，不是路由表的话。

这条读数是产物服务上的。开发模式（`nuxt dev`）会不会给 405，**这一轮没读**——起开发服务要你点头，而我不想拿一个没量的档位写进结论。要判这个问题，最直接的办法就是在你自己的机器上拿 `PUT` 打一个 `.get.ts` 的接口，两边各打一次对比。

## `$fetch` 这一侧：认方法，不认地址

`InternalApi` 那张表不只给运行时用，它同时进类型。三个探针写法，跑 `npx nuxt typecheck`：

```ts [app/utils/：三种写法]
await $fetch('/api/coun')                                // 地址打错
await $fetch('/api/auth/logout', { method: 'GET' })      // 方法与该地址不符
const wrong: number = await $fetch('/api/docs')          // 给返回值加个标注
```

读数（`rc=2`，六行）：

```text
app/utils/probe-typed-fetch.ts(10,45): error TS2322: Type '"GET"' is not assignable to
  type '"POST" | "post" | undefined'.
app/utils/probe-typed-fetch.ts(14,31): error TS2321: Excessive stack depth comparing types ...
（另外四行同样是 TS2321，内容全是路由匹配的模板字面量分支）
```

第一行是这站最有用的一条护栏：`/api/auth/logout` 只有 `.post.ts`，类型里那个地址允许的方法就是 `"POST" | "post"`，写 `GET` 当场红。也就是说**方法名进类型检查了，地址字符串没有**——`'/api/coun'` 那个拼错一个字母的地址零报错，运行时才 404。

第三行是个坑：给 `$fetch` 的结果加类型标注，会撞上 `Excessive stack depth comparing types`——那是 Nitro 生成的一长串"这个地址匹配哪条路由"的模板字面量类型在比来比去，编译器直接放弃深度。同一句不加标注（`const rows = await $fetch('/api/docs')`）不报。仓库里现成的做法是走**泛型参数**：

```ts [app/pages/docs/index.vue 的真实写法]
const { data: docs } = await useAsyncData('docs:list', () => $fetch<DocSummary[]>('/api/docs'))
```

形状写进泛型那一位，今天整仓 `npx nuxt typecheck` 是 `rc=0`。

## 取参数与状态码

文档给的其余几条，一并列出（这几条本轮没造探针验证，别当已量）：

- 参数用方括号文件名（`server/api/hello/[name].ts`），取值走 `getRouterParam(event, 'name')` 或 `event.context.params`；要带校验就用 `getValidatedRouterParams` 配一个 schema。
- catch-all 用 `server/api/foo/[...].ts`（段落读 `event.context.params._`）或具名形式 `[...slug].ts`（读 `params.slug`）。为什么本轮没试：文档写它会注册一条匹配一切未命中地址的路由，我不敢在跑着真实接口的仓库里放这样一位。
- 不抛错就是 `200`，未捕获的错误是 `500`；要回别的错误码用 `createError({ status, statusText })` 抛出。
- 要回非错误的其它状态码用 `setResponseStatus(event, 202)`。
- 每个文件都默认导出 `defineEventHandler()`（`eventHandler()` 是别名）。
- `server/` 里不要引 Vue 应用侧的代码，反之也一样——这条与 738 那条边界是同一件事。

## 出问题对照

| 症状 | 原因 | 怎么办 |
| --- | --- | --- |
| 接口死活是 404，但文件就在 `server/api/` 里 | 方法没对上，请求掉进了页面渲染的兜底 | 先确认客户端用的方法；再看后缀拼写 |
| 想给某地址两个方法，注册表里却只有 `default` | 后缀拼错（比如 `.GET`、`.quert`） | 后缀是识别出来的关键字；写错它就变成地址的一部分 |
| 探活脚本 `HEAD` 打 `.get.ts` 得到 404 | `HEAD` 不自动等于 `GET` | 换 `GET`，或专门给 `HEAD` 一个处理函数 |
| `$fetch` 的地址拼错，类型检查全绿 | 地址字符串没进护栏 | 靠运行时用例，或者自己把那串地址收成常量 |
| 给 `$fetch` 结果加类型标注后 TS2321 爆一大片 | 路由匹配的模板字面量类型深度超限 | 形状写进泛型参数，别写外部标注 |
| 深层文件里 `../../..` 数到晕 | v4.3 起有 `#server` 别名，只能在 `server/` 内用 | `import { x } from '#server/utils/x'`；客户端里引 `#server` 会报错 |

## 自己跑一遍

1. 建 `server/api/probe-x.get.ts` 与 `server/api/probe-x.post.ts`，跑 `npx nuxt prepare`，打开 `.nuxt/types/nitro-routes.d.ts` 搜 `probe-x`：你会看到一个键下面挂着两行方法。
2. 再起一次 `nuxt dev`，`curl` 那个地址的 GET 与 POST，然后换 PUT——把读回来的状态码记下来。
3. 把后缀改成 `.foo`，重复第 1 步：键变成 `/api/probe-x.foo`、方法是 `default`；这时候原来的 `/api/probe-x` 就不存在了。
4. 在页面里写 `$fetch('/api/probe-x', { method: 'DELETE' })`，跑 `npx nuxt typecheck`：它会红，而报错里那个联合列出的就是这个地址注册过的方法。
5. 删掉探针，重跑 `npx nuxt prepare`，确认那张表回到你开工前的条数。

本轮六个探针处理函数、一个类型扩展测试文件、一条 e2e spec 与一个 `$fetch` 探针文件都已删净（空目录回收，注册表里搜 `probe` 命中 0，条数回到 22 键／22 方法位）；复跑 `npx nuxt typecheck` **rc=0、输出 0 字节**，`npm run lint` **rc=0**（还挂着那条 `vue/no-v-html` 的 warning）。

下一步 [742《目录边界（5）：server／routes、动态参数与 catch-all》](/posts/4da4e73a736020732f24a8defec04d7b/)：`[[id]]` 在服务端目录里到底等不等于"可省"、两种 catch-all 各注册成什么地址、以及那张注册表会不会被它们接管——都用真请求的读数说话。

## 本文依据

- `server/api` 自动前缀 `/api`、`server/routes` 不带前缀、服务端路由的动态支持不如 pages、默认导出 `defineEventHandler()`/`eventHandler()`、返回 JSON／Promise／`res.end()`、不要混引 Vue 与 Nitro 代码：`https://nuxt.com/docs/4.x/directory-structure/server`
- 方法后缀 `.get`/`.post`/`.put`/`.delete`、其它方法回 405、`index.[method].ts` 命名空间写法、`foo/bar.get.ts`：`https://nuxt.com/docs/4.x/directory-structure/server`
- 路由参数与 `getRouterParam`、`getValidatedRouterParams` 的校验建议、catch-all `[...].ts` 与 `[...slug].ts` 两种形式、`createError` 与 `setResponseStatus` 的状态码：`https://nuxt.com/docs/4.x/directory-structure/server`
- `#server` 别名（v4.3）只能在 `server/` 内使用、客户端引用它会报错：`https://nuxt.com/docs/4.x/directory-structure/server`
- Nitro 会为返回值的 API 路由生成类型，`$fetch()` 与 `useFetch()` 可取用：`https://nuxt.com/docs/4.x/guide/concepts/server-engine`
