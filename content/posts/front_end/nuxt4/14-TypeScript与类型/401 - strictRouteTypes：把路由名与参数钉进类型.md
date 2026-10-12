---
url: /posts/303ff7db95f80742b18b165cf39f4bc7/
title: strictRouteTypes：把路由名与参数钉进类型
date: 2027-02-22T11:00:00+08:00
updated: 2027-02-22T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`routeTypedFetch`（402）让内部接口按服务端路由定型，但认不出的路径怎么处理，是另一个开关决定的。`experimental.strictRouteTypes` 就干这件事：官方定义 "Reject a request to a path no server route answers."，且**它依赖 `routeTypedFetch`**。三档：`false`（默认）未识别的路径照样通过、类型是 `unknown`、方法也不拒绝——官方给的理由是"Nuxt cannot see every way a request may be answered: nitro middleware、`routeRules` proxy 或 catch-all handler 都可能答一个没有路由描述的路径"；`true` 只接受服务端构建器报告的路由、且只按它们应答的方法；`'isomorphic'` 在 true 之上把页面也算成返回 `string` 的 `GET` 路由。开了之后拼错路径和方法不匹配都是错误。另有几种情况官方明确说仍放行：绝对 URL、运行时拼出的路径、显式写了响应泛型的调用。

categories:
  - 前端开发

tags:
  - nuxt4
  - strictRouteTypes
  - 类型检查
  - $fetch
  - 实验特性
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

开了 `routeTypedFetch` 之后，`$fetch('/api/todos')` 的返回类型来自真正应答它的那个 handler。但接下来有个必须回答的问题：请求一个 Nuxt 根本没记录的路径，算错误，还是将就？`experimental.strictRouteTypes` 就是这枚档位开关。

## 它的前提是 routeTypedFetch

官方定义：

> Reject a request to a path no server route answers. Requires [`routeTypedFetch`], which types `$fetch` and `useFetch` from the routes your server will serve.

两件事：作用域是"没有服务端路由应答的路径要不要拒绝"；**必须先开 `routeTypedFetch`**，两个一起配：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    routeTypedFetch: true,
    strictRouteTypes: true,
  },
})
```

## 先分清谁决定什么

官方把职责切得很干净，别把两件事记到同一个开关头上：

> With `routeTypedFetch` enabled a response is typed by the handler that answers it whatever this option is set to. What the option decides is whether a path Nuxt does not recognise is an error or is simply typed as `unknown`.

- **返回类型**由 `routeTypedFetch` 决定，跟 `strictRouteTypes` 取值无关。
- **未识别路径算不算错**，才由 `strictRouteTypes` 决定。

## 三档逐条

| 取值 | 行为（依文档） |
| --- | --- |
| `false`（默认） | Nuxt 认不出的路径**被接受**、解析为 `unknown`，且**不拒绝任何方法** |
| `true` | 只接受你的服务端构建器报告的那些路由，并且只按它们应答的方法 |
| `'isomorphic'` | 同 `true`，另外把页面也算作返回 `string` 的 `GET` 路由，于是 `$fetch('/about')` 由 Vue 路由服务的那条路由定型，而不是被拒绝 |

`false` 这一档官方给了完整理由句——不是"还没做完"，而是 Nuxt 天生看不见所有应答方式：

> Nuxt cannot see every way a request may be answered: nitro middleware, a [`routeRules`] proxy or a catch-all handler can answer a path no route describes.

这就是为什么默认宽松。反过来，开了严格档之后，拼错和方法不匹配都成错误：

```ts
// server/api/todos.post.ts 只应答 POST
await $fetch('/api/todos', { method: 'POST' })
await $fetch('/api/todo', { method: 'POST' })
//           ^ 没有 POST 路由匹配 '/api/todo'（拼错）
await $fetch('/api/todos')
//           ^ 没有 GET 路由匹配 '/api/todos'（方法不对）
```

`'isomorphic'` 解决的是一个真实矛盾：前端有时确实会去 `$fetch` 一个页面路径，`true` 会把它判成错误，`'isomorphic'` 把页面纳进路由表当 `GET` 返回 `string`，就正常了。

## 三种情况即使开了也放行

官方点名了两类"本来就不归 Nuxt 描述"的，以及一类"你主动关掉推断"的：

1. **绝对 URL**：`$fetch('https://example.com')` 仍被接受。
2. **运行时拼出来的路径**：解析为 `unknown`（和关掉这个选项时一样）。
3. **显式写了响应泛型**：`$fetch<{ id: string }>(\`/api/${resource}\`) —— 因为你亲自命名了响应类型，就等于这次请求不要推断了。

另外 `::note` 补了一条容易踩的：**`public/` 目录里的文件不是服务端构建器报告的路由**，所以请求一个 public 文件（比如 `/robots.txt`）落不进那张表。

还有一条关于 `baseURL` 的机制值得单独记住：

> A `baseURL` is resolved into the path before it is matched, so a request made through one is checked against the route it actually asks for.

也就是 `baseURL` 会先解析进路径再去比对——用 `baseURL` 发的请求，是按它真正请求的那条路由来检查的，不会因为 baseURL 前缀而误判。

## 怎么选

- 内部接口都由 `server/` 里的 handler 应答、没有依赖 Nitro 中间件或 routeRules proxy 去答特殊路径 → `true`，让拼错和方法错误在编辑器就暴露。
- 页面也要被 `$fetch`（比如某些取 HTML 的场景）→ `'isomorphic'`。
- 项目里有 catch-all、代理、中间件这些"Nuxt 看不见"的应答方式 → 保持默认 `false`，否则会被误报。

## 自查

- `strictRouteTypes` 管"未识别路径算错误还是算 `unknown`"；返回类型由 `routeTypedFetch` 决定，与它取值无关；**必须先开 `routeTypedFetch`**。
- 三档：`false`(默认宽松，理由句点名 nitro 中间件/routeRules proxy/catch-all)、`true`(只接受构建器报告的路由与其方法)、`'isomorphic'`(再把页面当返回 `string` 的 GET 路由)。
- 开了之后拼错路径与方法不匹配都是错误。
- 仍放行三种：绝对 URL、运行时拼出的路径、显式写了响应泛型的调用；`public/` 下的文件不算路由。
- `baseURL` 先解析进路径再比对，所以按真正请求的那条路由检查。

## 下一篇

下一篇 402 讲这一组的地基 `routeTypedFetch`：`$fetch`/`useFetch` 改由服务端构建器报告的路由集定型，五条后果逐条讲，以及 `Endpoint` 与手写 `ServerRoutes` 的迁移差异。

## 本文依据

- `## strictRouteTypes`：定义句与 Requires routeTypedFetch、配置示例、职责切分句（"a response is typed by the handler that answers it whatever this option is set to"）、三档 `false`/`true`/`'isomorphic'` 逐条描述与 `false` 的理由句、拼错与方法不匹配的典型代码、三类仍放行（绝对 URL、运行时路径、显式响应泛型）、`baseURL` 先解析再匹配、`::note`（`public/` 不是构建器报告的路由）（`guide/going-further/experimental-features`）。
