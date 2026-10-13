---
url: /posts/6c0a3e6daff40cbeba83e2853449e491/
title: 204 - canonical 链接：文档只提过一次，可用的工具是 useRequestURL
date: 2026-12-19T09:00:00+08:00
updated: 2026-12-19T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：`rel="canonical"` 是 `<link>` 上的一个属性，作用是告诉搜索引擎"这篇内容的正式地址是哪一个"，用于同一个页面有多个入口地址（带筛选参数的列表页、分页、站内转载）的情况。我把整套 4.x 文档检索过，**`canonical` 作为 SEO 概念只出现一次**，在流式渲染那一节的警告里："Keep other SEO-critical tags (canonical links, social meta) synchronous, or opt the route out of streaming with `routeRules: { '/path': { streaming: false } }`."——它给了一个要求（要同步注册），没给写法。能拼出写法的素材有三块：`useHead` 参数表里 `link` 那行 "Each element is mapped to a `<link>` tag, where object properties correspond to HTML attributes."（`rel`、`href` 都是属性，直接当键写）；同表 `base` 行 "Sets the `<base>` tag for the document."（文档里唯一被明写能设 `<base>` 的地方）；以及 `useRequestURL`——"a helper function that returns an URL object working on both server-side and client-side"，开发环境示例给出 `http://localhost:3000/about` 这种**绝对形态**。最要紧的坑在 `useRequestURL` 页那条 important：混合渲染命中缓存时"all incoming request headers are dropped"，此时 **host 会读成 `localhost`**，得用 `cache.varies` 把 `host`、`x-forwarded-host` 放进参与缓存键的头里。

categories:
  - 前端开发

tags:
  - nuxt4
  - canonical
  - useRequestURL
  - 缓存
  - SEO

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先说清一件事：这一篇的写法部分是我按文档给的映射规则拼出来的，官方没有 canonical 示例。规则我有出处，拼出来的结果你得自己验一次。

## 文档里那唯一一次提到

在讲流式渲染的那一节（193 也引过同一段），警告框最后一句是：

> Keep other SEO-critical tags (canonical links, social meta) synchronous, or opt the route out of streaming with `routeRules: { '/path': { streaming: false } }`.

这句给的是**时机要求**，不是写法。它的前半段解释了为什么要同步：

> The renderer waits for the first render pass before flushing the shell, so `useHead`/`useSeoMeta` called synchronously in `<script setup>` (before any `await`), in plugins, or in middleware render into the shell `<head>` as usual. Tags registered later (after an `await` in setup, or inside a nested `<Suspense>` boundary) cannot reach the already-sent `<head>`; they are delivered as inline scripts that patch the DOM on the client.

对 canonical 来说这条尤其致命：它是给搜索引擎读的，而"不跑 JS 的一方"（原话点名了 crawler、link unfurler、`curl`）读不到那些客户端补丁。所以我从这两句能确认的只有一件——**如果你的站点开了流式渲染，canonical 必须在任何 `await` 之前注册**，不能跟在取数后面。

## 写法：从 link 的映射规则拼

`useHead` 参数表里跟 `<link>` 有关的两行，原话如下：

| 字段 | 类型 | 文档说明 |
| --- | --- | --- |
| `base` | `Base` | "Sets the `<base>` tag for the document." |
| `link` | `Link[]` | "Array of link objects. Each element is mapped to a `<link>` tag, where object properties correspond to HTML attributes." |

同页有一个"外部 CSS"的示例，它示范了 `rel` 与 `href` 就是当键直接写的：

```vue [app/pages/external.vue]
<script setup lang="ts">
useHead({
  link: [
    {
      rel: 'stylesheet',
      href: 'https://cdn.example.com/styles.css',
    },
  ],
})
</script>
```

按同一条映射规则，把 `rel` 换成 `'canonical'`、`href` 换成正式地址即可。**这一段是我的套用，文档没有 canonical 示例**——所以落地前用 `curl` 看一眼服务端 HTML 里那一行的实际形状，不要凭这段代码就上线。

## 绝对 URL 从哪来：useRequestURL

canonical 需要带域名的完整地址。文档给的工具是 `useRequestURL`：

> `useRequestURL` is a helper function that returns an [URL object] working on both server-side and client-side.

```vue [app/pages/about.vue]
<script setup lang="ts">
const url = useRequestURL()
</script>

<template>
  <p>URL is: {{ url }}</p>
  <p>Path is: {{ url.pathname }}</p>
</template>
```

```html [Result in development]
<p>URL is: http://localhost:3000/about</p>
<p>Path is: /about</p>
```

它返回的是 URL 对象（`pathname` 之外还有 MDN 那份实例属性清单，官方在这一节直接指向 MDN），示例给出的形态正是我要的绝对地址。要注意这是**开发环境**的结果：值里带 `localhost:3000`，说明它读的是进来的那个请求，而不是某个配置项。

## 真正的坑：命中缓存时 host 会变成 localhost

`useRequestURL` 页的 important 是这个话题里最该背下来的一条：

> When utilizing Hybrid Rendering with cache strategies, all incoming request headers are dropped when handling the cached responses via the Nitro caching layer (meaning `useRequestURL` will return `localhost` for the `host`).
>
> You can define the `cache.varies` option to specify headers that will be considered when caching and serving the responses, such as `host` and `x-forwarded-host` for multi-tenant environments.

我把它翻译成症状：**开了 `swr` / `isr` 这类缓存的路由，从缓存命中的那次响应里拿到的 host 是 `localhost`**，于是你生成的 canonical 也带上 `localhost`——而且只在缓存命中时出现，本地怎么测都像没问题。解法是文档给的那一条：把这些头列入 `cache.varies`（`host`、`x-forwarded-host`）。

顺带一提，多域名/多租户环境里 `x-forwarded-host` 也要一起考虑，官方原话点名的正是这两个头。

## 三条自检

1. **同步性**：如果 `experimental.ssrStreaming` 是开的（或者你不确定），先看你的 `useHead` 调用有没有落在某个 `await` 之后。判据来自上面那句"before any `await`"。
2. **域名**：在部署环境而不是本机，走一条会被缓存的路由两次（第一次生成缓存、第二次命中），比较两次 `curl` 出来的 canonical。两条不一样就说明命中了那条 important。
3. **参数页**：同一篇文章带 `?page=2` 与不带参数是两个 URL。文档没写该怎么选正式地址——那是搜索引擎侧的规则，不在 Nuxt 文档范围里，我只能提醒你这个场景一定会遇到。

## 自查

1. `canonical` 在整套文档里作为 SEO 概念出现几次，在哪一节？（答：一次，流式渲染那节的警告框里，给的是"要同步注册"的要求。）
2. `useRequestURL` 在缓存命中时为什么会给出 `localhost`？（答：混合渲染命中 Nitro 缓存时会丢弃全部入站请求头，host 因此读成 `localhost`；用 `cache.varies` 把 `host`/`x-forwarded-host` 纳入缓存键。）
3. 我的 canonical 写法有多少是文档原话？（答：`link` 的映射规则与 `base` 行是原话；`rel: 'canonical'` 这个具体用法是我的套用，需实测。）

## 下一篇

205 - robots 的两处记载：meta 里那个字段与一个要保名的文件

## 本文依据

- `guide/going-further/experimental-features`（`ssrStreaming` 一节）："Keep other SEO-critical tags (canonical links, social meta) synchronous, or opt the route out of streaming with `routeRules: { '/path': { streaming: false } }`."；"`useHead`/`useSeoMeta` called synchronously in `<script setup>` (before any `await`), in plugins, or in middleware render into the shell `<head>` as usual."；"Anything else (a crawler your `botRegex` does not list, a link unfurler, `curl`) will not see those tags at all."；开启键是 `experimental.ssrStreaming`（同节示例），路由级退出写 `'/no-stream/**': { streaming: false }`。
- `api/composables/use-head`：参数表 `base` 行与 `link` 行原文；"Adding External Scripts and Styles" 里 `{ rel: 'stylesheet', href: '...' }` 的示例形态。
- `api/composables/use-request-url`：定位句 "a helper function that returns an URL object working on both server-side and client-side"；`const url = useRequestURL()` 与模板输出、开发环境结果 `http://localhost:3000/about`；important 两段（headers dropped ⇒ host 为 `localhost`、`cache.varies` 与 `host`/`x-forwarded-host`）；指向 MDN 的 URL 实例属性 tip。
- `api/nuxt-config`（routeRules 相关，141 已细讲）：`swr` / `isr` 落缓存层的说法，用来定位"哪条路由会命中缓存"。
- 检索结论（否定性事实）：`canonical` 在 4.x 语料的正文里只命中 experimental-features 一处；SEO 页与 `useHead` 页都没有 canonical 示例。
