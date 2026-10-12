---
url: /posts/a7b2062e8c038c63aa0800fd20413848/
title: Moving to nuxt-server：一张 17 行的对照表与一个 500 报错码
date: 2027-05-09T09:00:00+08:00
updated: 2027-05-09T09:00:00+08:00
author: cmdragon

summary:
  摘要：升级页的 "Moving to `nuxt/server`" 一节解决的是**同一份服务端代码要能同时在 Nuxt 4（nitropack v2 ＋ h3 v1）与 Nuxt 5（Nitro v3 ＋ h3 v2）上跑**这件事：**从 Nuxt 4.6 起**可以把 handler 与助手从 `nuxt/server` 显式导入，而不是从 h3；**4.6 上必须显式 import**，因为同名自动导入来的仍是 h3 v1 的那批，而 h3 的 `defineEventHandler` 传给你的 event 与 `nuxt/server` 助手读取的不是同一个。随后那张 **17 行对照表**逐条列出差异，最容易中招的四条是：**`sendRedirect` 从"发送响应"变成返回响应体（要 `return`）**、**`createError({ statusCode, statusMessage })` 变成 `createError({ status, statusText })`**、**`isError` 变成 `isNuxtError`**、**`useRuntimeConfig(event)` 变成没有按请求形式的 `useRuntimeConfig()`**。还有一条 500 的成因：**`NUXT_E8012`——`nuxt/server` 的助手收到了 h3 的 event**，说明 handler 是用 h3 的 `defineEventHandler`（通常就是自动导入那个）定义的。

categories:
  - 前端开发

tags:
  - nuxt4
  - 服务端导入
  - h3
  - 错误码
  - 升级迁移
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

同一段 `server/api/*.ts`，在 4 上跑得好，升到 5 就 500，报的还是一句没见过的 `[NUXT_E8012]`。这一节讲的就是它：把导入源从 h3 换成 `nuxt/server`，然后按那张表逐条对。

## 为什么要换导入源

> From Nuxt 4.6, server code can import its handler and helpers from `nuxt/server` rather than from h3, so the same file runs on Nuxt 4 (`nitropack` v2 and h3 v1) and on Nuxt 5 (Nitro v3 and h3 v2).

关键是"一份代码跑两代"。**但 4.6 上要显式写 import**，官方给的理由很具体：

> On Nuxt 4.6, import `defineEventHandler` and the helpers from `nuxt/server` explicitly. The auto-imported names of the same spelling are h3 v1's, and **h3's `defineEventHandler` passes your handler a different event from the one the `nuxt/server` helpers read.**

也就是说：**自动导入的那批同名函数是 h3 v1 的**，用它们定义 handler、再用 `nuxt/server` 的助手读参数，两边拿到的 event 不是同一个——这正是 `NUXT_E8012` 的成因。到了 Nuxt 5，**这些名字会从 `nuxt/server` 自动导入**，所以 4.6 上写的显式 import 届时无需改动。

官方示例就是加两行 import：

```diff [server/api/hello.ts]
+ import { defineEventHandler, getQuery } from 'nuxt/server'
+
 export default defineEventHandler((event) => {
   const { name } = getQuery<{ name?: string }>(event)
   return { message: `Hello, ${name ?? 'world'}!` }
 })
```

## 17 行差异对照

这张表就是"迁移时到底要改哪几处"的全部依据：

| h3 v1 | `nuxt/server` |
| --- | --- |
| `sendRedirect(event, location)` **会发送响应** | `sendRedirect(event, location)` **返回响应体**：`return sendRedirect(event, '/login')` |
| `getRouterParam(event, name)` 与 `getRouterParams(event)` 返回**已解码**的值 | 返回 URL 里的原样值；**要解码得传 `{ decode: true }`** |
| `handleCors(event, options)` 在应答预检请求时返回 `true` | 返回要由 handler 返回的预检 `Response`，或 `false`：`const preflight = handleCors(event, options); if (preflight) return preflight` |
| `handleCors` 运行时可接受单个 origin 字符串 | `origin` 是 `'*'`、`'null'`、origin/模式数组，或函数：**单个 origin 要包成数组**；`maxAge` 是秒数字符串（如 `'86400'`）或 `false`；`CorsOptions` 类型从 `nuxt/server` 导入 |
| `readValidatedBody` 与 `getValidatedQuery` 接受校验函数 | 接受 **Standard Schema** 或函数 |
| `createError({ statusCode, statusMessage })` | `createError({ status, statusText })` |
| `isError(error)` | `isNuxtError(error)` |
| h3 的 `useSession`／`getSession`／`updateSession`／`clearSession` 用你传的 `password` 封一个 `h3` cookie | 它们封一个 `nuxt-session` cookie，默认密钥由 `appSecret` 推导；**两者不可互换**：一边签发的 session 另一边解不开 |
| `useRuntimeConfig(event)` | `useRuntimeConfig()`，**没有按请求的形式** |
| 全局 `$fetch` | **不属于 `nuxt/server`**：调应用内路由用 `serverFetch(event, path)`（返回 `Response`）；其它 URL 用原生 `fetch` 时**非 2xx 不抛错、也不解析 JSON**，要自己看 `response.ok` 与 `response.json()` |
| `event.path` | `getRequestURL(event).pathname`，或 `event.url.pathname` |
| `event.node` | 不属于 `nuxt/server`：改用 `event.req` 与 `event.res` |
| `getResponseHeader`／`setResponseHeader`／`appendResponseHeader`／`removeResponseHeader` | `event.res.headers.get()`／`.set()`／`.append()`／`.delete()` |
| `getMethod(event)`、`isMethod(event, method)`、`assertMethod(event, method)` | 直接比较 `event.req.method` |
| `readMultipartFormData(event)` | `await event.req.formData()` |
| `readRawBody(event)`、`getRequestWebStream(event)` | `await event.req.arrayBuffer()`（或 `.text()`）、`event.req.body` |
| `sanitizeStatusCode` | **不提供** |

四行最容易出事，因为它们**不报错但语义变了**：`sendRedirect`（忘了 `return` 就等于没生效）、`getRouterParam` 的解码、`createError` 的字段名、以及 **`useRuntimeConfig()` 不再有按请求形式**。另外两行是"能力被拿走"：`$fetch` 全局不在、`event.node` 不在。

## 需要 h3 独有助手时

> For a handler that needs h3 or Nitro helpers that `nuxt/server` does not provide, such as `readMultipartFormData`, `proxyRequest`, `useStorage` or `defineCachedEventHandler`, keep importing `defineEventHandler` and those helpers from `h3` (or `nitro/
> pack/runtime`) for that handler. **Code that does this is tied to the server runtime and needs revisiting on Nuxt 5.**

官方示例是上传处理：

```ts [server/api/upload.post.ts]
import { defineEventHandler, readMultipartFormData } from 'h3'

export default defineEventHandler(async (event) => {
  const parts = await readMultipartFormData(event)
  return { received: parts?.length ?? 0 }
})
```

要点是那句代价：**这么写就把这个 handler 绑在当前服务端运行时上了，Nuxt 5 时要回头处理**。点名的四个助手是 `readMultipartFormData`、`proxyRequest`、`useStorage`、`defineCachedEventHandler`——**注意第一个在上表里其实有 Web 标准对应写法（`event.req.formData()`）**，也就是官方给了两条路：临时留在 h3，或直接换成 Web 标准形态。

## NUXT_E8012：一种 500

> A request fails with a `500` whose message starts with `[NUXT_E8012]` when a `nuxt/server` helper such as `readValidatedBody`, `getValidatedQuery`, `handleCors` or `useSession` receives an h3 event.

> This means the handler was defined with h3's `defineEventHandler` (usually the auto-imported one). **Import `defineEventHandler` from `nuxt/server` in the same file as the helpers.**

诊断路径很短：看到 `[NUXT_E8012]` 开头的 500，**先看同一个文件里的 `defineEventHandler` 是从哪儿 import 的**——点名的助手是 `readValidatedBody`、`getValidatedQuery`、`handleCors`、`useSession`。修法就是那句：助手与 handler 必须同源。

## 应用密钥的一处 4 侧坑

这一节最后一段是 `### Application Secret`：**`nuxt/server` 的 session 助手与 `deriveSecret` 读的是 `runtimeConfig.appSecret`，用 `NUXT_APP_SECRET` 设**。而在 Nuxt 4 上 **`nitropack` v2 会解析环境变量覆盖值，所以纯数字组成的密钥会以数字形式到达**——官方把那段的细节推给运行时配置页的 application secret 一节。

## 自查

- 目的：**一份服务端代码同时跑 4 与 5**；4.6 起**必须显式从 `nuxt/server` 导入 handler 与助手**（同名自动导入仍是 h3 v1 的）。
- 语义变化的四行：`sendRedirect` 要 `return`、路由参数不再自动解码（`{ decode: true }`）、`createError` 字段改 `status`/`statusText`、`useRuntimeConfig()` 无按请求形式；`isError`→`isNuxtError`。
- 被拿走的能力：全局 `$fetch`（改 `serverFetch(event, path)` 或原生 `fetch` ＋ 自己判 `ok`）、`event.node`、`sanitizeStatusCode`。
- 需要 h3/Nitro 独有助手时**这个文件就绑定当前运行时**，Nuxt 5 要回头改。
- `[NUXT_E8012]` 的 500 = **助手与 handler 不同源**；`appSecret` 纯数字会被解析成数字。

## 下一篇

下一篇 628 读 nightly 渠道那一页：**合进 `main` 且通过全部测试后由 GitHub Actions 自动发布**；进出都要改 `package.json` 里的 `nuxt` 值（`npm:nuxt-nightly@latest`）并**删掉锁文件**——官方列了 5 个锁文件名；那条 `::warning` 现在特别要读：**`latest` 渠道正追 Nuxt v4 分支，因此此刻很可能有破坏性变更**，想留在 3.x 要用 `"nuxt": "npm:nuxt-nightly@3x"`；CLI 另有 `npx @nuxt/cli-nightly@latest [command]`。

## 本文依据

- https://nuxt.com/docs/4.x/getting-started/upgrade（"Moving to `nuxt/server`" 整节：4.6 起可改导入源的理由句、"the auto-imported names of the same spelling are h3 v1's…" 句、`server/api/hello.ts` diff 示例、Nuxt 5 上改为自动导入那句；`### Differences From h3 v1` 的 17 行对照表全文；`### Helpers Outside nuxt/server` 段与四个被点名助手、`server/api/upload.post.ts` 示例；`### NUXT_E8012` 整段；`### Application Secret` 整段含 `NUXT_APP_SECRET` 与数字密钥被解析的说明）
- https://nuxt.com/docs/4.x/guide/going-further/server-imports（`nuxt/server` 导入面本身，本页只按升级视角引用）
- https://nuxt.com/docs/4.x/guide/going-further/runtime-config#application-secret（应用密钥一节，被本节链接指向，用于查数字密钥的具体避法）
