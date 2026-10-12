---
url: /posts/c36fb527e0307ef3bdf64f6474af5b2a/
title: prerenderRoutes：预渲染时手动登记额外路由
date: 2027-01-27T11:00:00+08:00
updated: 2027-01-27T11:00:00+08:00
author: cmdragon

summary:
  摘要：预渲染默认靠爬虫顺 `<a href>` 链接去发现页面，没被链接到的路由就轮不到。prerenderRoutes 让你在页面里"提示"额外要预渲染的路径，即使它的 URL 没出现在已生成的 HTML 里。这一篇讲它的三条限定——只能在 Nuxt 上下文里调、必须"在预渲染过程中执行"才有效（不在预渲染的动态页里调用就啥也不做、浏览器里也无效果）、以及 Nuxt 收集这些提示转交给服务端构建器的预渲染器（默认 Nitro）。还落一个实用点：连 API 路由也能 prerenderRoutes，SSG 站就能像有后端一样 $fetch 它——但要手动写 responseType，因为预渲染的接口在生产环境可能回错 content-type。

categories:
  - 前端开发

tags:
  - nuxt4
  - prerenderRoutes
  - 预渲染
  - SSG
  - Nuxt上下文
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

预渲染时，Nitro 的爬虫只会顺着页面里的 `<a href>` 链接一页页爬（200、P11 会再讲）。问题是：有些路径（比如某个 CMS 才有的文章接口、某个没被任何链接指向的页面）爬虫发现不了，就不会被预渲染。`prerenderRoutes` 就是让你手动把这些路径"报"上去。

## 它是什么

官方定位与原理：

> prerenderRoutes hints that an additional route should be prerendered.

> When prerendering, you can hint that additional paths should be prerendered, even if their URLs do not show up in the HTML of the generated page. Nuxt collects the hints and passes them to your server builder's prerenderer (Nitro by default).

三句要点：报的是"额外要预渲染的路径"；触发条件是"在预渲染期间"；Nuxt 只是**收集这些提示转交给服务端构建器的预渲染器**（默认 Nitro，呼应 300 的 Nitro）。

## 三条限定，缺一条就无效

限定一（important）：

> `prerenderRoutes` can only be called within the [Nuxt context].

只能在 Nuxt 上下文里调（295 四类位置）。

限定二（note）：

> `prerenderRoutes` has to be executed during prerendering. If the `prerenderRoutes` is used in dynamic pages/routes which are not prerendered, then it will not be executed.

**它必须在"预渲染过程中"被真正执行才有用**。如果你把它放在一个根本不会被预渲染的动态页面里，那段代码不会跑、也就登记不上。

限定三（note）：

> In the browser, or if called outside prerendering, `prerenderRoutes` will have no effect.

浏览器里、或在预渲染之外调用，都**没有效果**。

基本用法：

```ts
const route = useRoute()

prerenderRoutes('/')
prerenderRoutes(['/', '/about'])
```

传一个路径，或一个路径数组。

## 页面里登记别的 URL

官方在预渲染那页给的例子——一个页面被预渲染时，顺手登记别的要预渲染的 URL：

```vue [app/pages/index.vue]
<script setup>
prerenderRoutes(['/some/other/url'])
prerenderRoutes('/api/content/article/my-article')
</script>
```

常见用途：首页被预渲染时，把爬虫看不见的详情页/接口路径报上去。

## 连 API 路由也能预渲染（但有个 content-type 坑）

官方点出一个对纯静态站很有用的能力：

> You can even prerender API routes which is particularly useful for full statically generated sites (SSG) because you can then `$fetch` data as if you have an available server!

```ts
prerenderRoutes('/api/content/article/name-of-article')

// Somewhere later in App
const articleContent = await $fetch('/api/content/article/name-of-article', {
  responseType: 'json',
})
```

含义：静态站本来没有活的后端，但如果你把这个接口的返回预渲染成一个静态文件，SSG 站点就能像有服务器一样 `$fetch` 它——预渲染阶段把结果落成了一个文件。

但紧跟一条 warning，务必照做：

> Prerendered API routes in production may not return the expected response headers, depending on the provider you deploy to. For example, a JSON response might be served with an `application/octet-stream` content type. Always manually set `responseType` when fetching prerendered API routes.

预渲染的接口在生产环境、按不同部署商，可能回**不预期的响应头**——比如一个本该是 JSON 的响应，可能带 `application/octet-stream` 的 content-type。所以官方要求：**取预渲染接口时，永远手动写 `responseType`（如示例的 `responseType: 'json'`）**，别指望靠响应头自动判断格式。

## 和别的"登记预渲染路由"的方式对齐

登记要预渲染的路径，官方在预渲染那页一共给了几条路，`prerenderRoutes` 是"运行时报"的这一条，其余记一下坐标：

| 方式 | 时机 | 在哪写 |
| --- | --- | --- |
| `prerender.routes` 数组 | 配置期 | `nuxt.config` 的 `prerender` |
| `routeRules` 里 `prerender: true` | 配置期 | `nuxt.config` |
| `prerenderRoutes(...)` | 预渲染运行期（上下文内） | 页面 setup / 插件 |
| `prerender:routes` Nuxt 钩子 | 预渲染前 | `nuxt.config` 的 `hooks` |
| `prerender:generate` Nitro 钩子 | 每个路由预渲染时 | `nitro.hooks`，可 `route.skip = true` |

`prerenderRoutes` 独有的是"在页面渲染那一刻、根据这次渲染算出的路径动态报上去"。

## 自查

- `prerenderRoutes` 只在 Nuxt 上下文、且真的在预渲染执行时才起作用；浏览器或预渲染之外调用没效果。
- 用它登记"爬虫看不见链接"的路径；动态未预渲染的页面里调它也不会跑。
- 预渲染 API 路由能让 SSG 像有后端一样取数，但生产可能回错 content-type → 永远手动写 `responseType`。
- 静态可枚举的路径更该用 `prerender.routes`/`routeRules`/`prerender:routes` 钩子；`prerenderRoutes` 专管"渲染时才知道的"路径。

## 下一篇

下一篇 324 讲流式 SSR（`experimental.ssrStreaming`）：先吐 HTML 外壳再逐步流式渲染 body，以及命中缓存/机器人等会退回缓冲渲染、状态头必须在 shell 刷出前设好这些边界。

## 本文依据

- prerenderRoutes（`api/utils/prerender-routes`）：定位句与"even if their URLs do not show up in the HTML ... Nuxt collects the hints and passes them to your server builder's prerenderer (Nitro by default)"；important "can only be called within the Nuxt context"；note "has to be executed during prerendering ... dynamic pages/routes which are not prerendered, then it will not be executed"；`prerenderRoutes('/')`/`(['/', '/about'])` 用法；note "In the browser, or if called outside prerendering ... will have no effect"；预渲染 API 路由段（"full statically generated sites (SSG) ... $fetch data as if you have an available server"、`prerenderRoutes('/api/content/article/name-of-article')` + `$fetch(..., { responseType: 'json' })`）；warning（"may not return the expected response headers ... application/octet-stream content type. Always manually set responseType"）。
- Prerendering（`getting-started/prerendering`）：`prerenderRoutes` 页面示例与 prerender:routes / prerender:generate 钩子（见 200）。
