---
url: /posts/cd37f255017a90e206a01867e9a65c6a/
title: 未匹配路由与 404：early404 和 validate 各管一段
date: 2027-03-16T10:00:00+08:00
updated: 2027-03-16T10:00:00+08:00
author: cmdragon

summary:
  摘要：一个"404"在 Nuxt 里可能来自三条完全不同的路。第一条是**页面自己拒绝**：`definePageMeta` 里的 `validate` 返回 `false` 就产生 404，官方还写明可以直接返回带 `status`/`statusText` 的对象来自定义错误。第二条是**客户端导航匹配不到页面**：`experimental.serverPathFallback`（默认开）会把它当整份文档加载，让服务端去回应 `public/` 里的文件、某个服务端路由或它自己的 404 页，`<NuxtLink to="/brochure.pdf">` 因此和硬加载表现一致。第三条是**服务端提前拒绝**：`experimental.early404`（默认关）把页面路由编译成构建期静态匹配器，匹配不上的请求**不加载 Vue 应用、插件与中间件**直接进错误页。状态码字段新旧两套并存：`status` 是当前字段，`statusCode` 是官方标注的 legacy 等价物。

categories:
  - 前端开发

tags:
  - nuxt4
  - 404
  - validate
  - early404
  - 路由
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

用户点了首页上一份 PDF 的链接，页面跳去一个 404；你在浏览器地址栏直接粘贴同一个地址，却正常下载了。

这不是缓存问题——**这两条路径走的是不同机制**。404 在 Nuxt 文档里其实分散在三处，先把它们分清再谈怎么改。

## 三条路一览

| 场景 | 由谁负责 | 默认状态 | 官方定义位置 |
| --- | --- | --- | --- |
| 路由匹配到了，但参数不合法 | `definePageMeta` 的 `validate` | 需自己在页面里写 | 路由那一页 Route Validation |
| 客户端导航匹配不到任何页面 | `experimental.serverPathFallback` | **默认开** | 实验特性那一节 |
| 服务端收到压根匹配不上的路径 | `experimental.early404` | **需显式开** | 实验特性那一节 |

三者互不覆盖：`validate` 是**页面层面的拒绝**，后两者是**路由解析层面的处理策略**。

## 路一：validate 返回 false 就是 404

官方给的定义段：

> Nuxt offers route validation via the `validate` property in `definePageMeta()` in each page you wish to validate.
>
> The `validate` property accepts the `route` as an argument. You can return a boolean value to determine whether or not this is a valid route to be rendered with this page. If you return `false`, this will cause a 404 error. You can also directly return an object with `status`/`statusText` to customize the error returned.

四个要点：**逐页声明**（每个想校验的页面自己写）、入参是 `route`、返回布尔、**返回 `false` 就产生 404**。第二句里那条更实用的能力常被忽略：**直接返回带 `status`/`statusText` 的对象，就能自定义返回的错误**——想给不同参数错返回 400 或 410 的，走这里而不是抛 `createError`。

官方示例（判断 id 是不是纯数字）：

```vue twoslash [pages/posts/[id\\].vue]
<script setup lang="ts">
definePageMeta({
  validate (route) {
    // Check if the id is made up of digits
    return typeof route.params.id === 'string' && /^\d+$/.test(route.params.id)
  },
})
</script>
```

复杂情况的官方指路是："If you have a more complex use case, then you can use anonymous route middleware instead."

这里还接得上 441 那条编译期事实：`definePageMeta` 里被构建提取的属性名单中就包含 `validate`（`objectDefinitions` 的默认值是 `["middleware", "validate"]`），所以**这个函数的值必须是静态可提取的**——不能是变量或动态生成的函数。

## 路二：serverPathFallback 让点链接和硬加载一致

这一节官方写的正是开头那个现象：

> Loads a client-side navigation that matches no page route as a full document, so the server can respond with a file in `public/`, a server route, or its 404 page. With this enabled, `<NuxtLink to="/brochure.pdf">` and `<NuxtLink to="/rss.xml">` work on click as they do on a hard load.

关键句是最后那句：**点击链接和硬加载表现一致**，因为匹配不到页面路由的导航被**当作整份文档重新加载**，交给服务端去答。三种回应都可能：`public/` 里的文件、某个服务端路由、或者服务端的 404 页。

再一句解释为什么不无限套娃：

> A path is never reloaded if the current document was already served for it, so hosts that respond to unknown paths with an SPA fallback render the client-side 404 page.

**当前文档已经为这个路径服务过，就不再重载**——所以那种"任何未知路径都返回 SPA 壳"的托管环境，渲染出来的是**客户端侧的 404 页**。

默认开，关掉是 `serverPathFallback: false`；另外一句限定：**启用 `hashMode` 时这一项无效**。

## 路三：early404 在加载应用之前就拒掉

官方定义：

> Responds with an early 404 error for requests whose path cannot match any page route, without loading the Vue app, its plugins or middleware on the server.

机制句：

> Page routes (including aliases) are compiled into a static route matcher at build time, and incoming requests are checked against it before server-side rendering begins. Requests that cannot match any page skip loading the app entirely and go straight to the error page.

**页面路由（含 alias）在构建期被编译成静态匹配器**，请求在 SSR 开始前先查它。这条省的是服务端整棵应用的加载。

三种请求的回应差异官方写全了：

- 没有 `Accept: text/html` 头的请求（API 客户端、多数扫描器）**收到 JSON 错误且不加载应用**，除非 `future.compatibilityVersion` 是 `5`。
- HTML 请求**仍然服务端渲染错误页**。
- 覆盖该路径的 `cache` 路由规则，其 `maxAge` 在 GET/HEAD 的 miss 上**也会作为 `Cache-Control` 广播出去**，让 CDN 吸收对未知路径的重复请求。

因为是 opt-in，官方列了它会打断什么（`::warning`）：

- 插件里用 `router.addRoute()` 动态加的页面——为了让这种情况提前暴露，**开启时 `addRoute` 会从 `useRouter` 返回值的类型里被移除**
- 把未知路径重定向到已有页面的路由中间件

以及四种自动失效的情形：**开发模式下、启用 `hashMode` 时、存在匹配所有路径的根级 catch-all 页面（如 `pages/[...slug].vue`）、以及自定义 `app/router.options` 文件可能修改 `routes` 时**（官方补了一句：默认导出是不含 `routes` 键的普通对象的那些文件仍然兼容）。

## 状态码那个字段：新旧两套

404 的"码"存在错误对象的哪个字段上？464 记的那份清单在这里正好用上：

```ts
interface NuxtError {
  status: number
  statusText?: string
  // legacy/deprecated equivalent of `status`
  statusCode: number
  // legacy/deprecated equivalent of `statusText`
  statusMessage?: string
}
```

**`statusCode`/`statusMessage` 是官方标注的旧等价物**，当前字段是 `status`/`statusText`。旧教程里读 `error.value.statusCode` 的写法就是这个来源。

再补一条与响应形态有关的规则：460 记过 fatal 错误会按 `Accept: application/json` 出 JSON 或全屏错误页；而升级到 `compatibilityVersion: 5` 时，官方在升级说明里写的是 `experimental.inlineErrorRendering` 启用后"页面错误返回错误页、服务端路由错误返回 JSON"，**不再取决于请求头**。

## 自查

- 分清三层：`validate` 是页面拒绝（返回 `false` 出 404，也可直接返回 `{ status, statusText }`）；`serverPathFallback` 管点链接匹配不到路由时按整份文档重载；`early404` 管服务端在加载应用前拒掉不可匹配的路径。
- `validate` 属于 `definePageMeta` 被静态提取的键之一，不能写成动态函数。
- `serverPathFallback` 默认开、`hashMode` 下无效；已服务过的路径不会重复重载。
- `early404` 默认关，会打断 `router.addRoute()` 与"中间件重定向未知路径"，并在 dev、`hashMode`、根级 catch-all、自定义 `router.options` 可能改 `routes` 时自动关闭。
- 读状态码用 `status`，`statusCode` 是旧等价物。

## 下一篇

下一篇 467 讲 `parseErrorData`：它决定**服务端渲染错误页时要不要解析 `error.data`**，默认开启。它接的是 462 那条链路的最后一环——你用 `createError({ data: {...} })` 送回来的结构化信息，在错误页上能不能读回来，就归这一项管。

## 本文依据

- 路由那一页 Route Validation 全节：`validate` 属于 `definePageMeta`、接受 `route` 作参数、返回布尔、返回 `false` 产生 404、可直接返回带 `status`/`statusText` 的对象自定义错误、复杂场景改用匿名路由中间件、`pages/posts/[id].vue` 示例原文（`getting-started/routing`）
- `## serverPathFallback` 全节：定义句（匹配不到页面路由的客户端导航按整份文档加载，服务端可回应 `public/` 文件、服务端路由或其 404 页；`/brochure.pdf` 与 `/rss.xml` 点击与硬加载一致）、"A path is never reloaded if the current document was already served for it" 与 SPA fallback 主机渲客户端 404、默认开启与关闭写法、`hashMode` 下无效（`guide/going-further/experimental-features`）
- `## early404` 全节：定义句（不加载 Vue 应用、插件与中间件）、构建期静态路由匹配器与 SSR 前检查、无 `Accept: text/html` 返回 JSON 且不加载应用（cv5 除外）、HTML 请求仍服务端渲错误页、`cache` 规则的 `maxAge` 在 GET/HEAD miss 上广播为 `Cache-Control` 供 CDN 吸收重复请求、`::warning` 两条打断项（`router.addRoute()` 与 `addRoute` 从 `useRouter` 类型移除；中间件重定向未知路径）、四种自动关闭情形与 router.options 兼容例外（`guide/going-further/experimental-features`）
- 错误对象的 `status`/`statusText` 与 `statusCode`/`statusMessage` 的 legacy 标注（`directory-structure/app/error`）；`definePageMeta` 被静态提取的键含 `validate`（`api/nuxt-config` 的 `optimization.asyncTransforms`）
- fatal 响应按 `Accept` 分岔（`getting-started/error-handling`）；cv5 下 `experimental.inlineErrorRendering` 使页面错误出错误页、服务端路由错误出 JSON 且与请求头无关（`getting-started/upgrade`）
