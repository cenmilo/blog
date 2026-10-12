---
url: /posts/250e5647bbb0cfca22882c1fee518b02/
title: 预渲染页面给抓取者看什么：抓取规则、载荷文件与 botRegex
date: 2026-12-17T11:00:00+08:00
updated: 2026-12-17T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇把"预渲染对抓取者意味着什么"落到文档写过的三件具体事上。第一件是**抓取器的发现规则**：`nuxt generate` "build your site, stand up a nuxt instance, and, by default, prerender the root page `/` along with any of your site's pages it links to, any of your site's pages they link to, and so on."，四步机制里第 3 步是 "Find all anchor tags (`<a href=\"...\">`) in the HTML to navigate to other routes."，紧接着那句要紧的结论："pages that are not linked to a discoverable page can't be pre-rendered automatically."——**没被链接指向的页面，抓取者也就拿不到**。第二件是载荷文件：预渲染路由在构建期生成 `_payload.json`，而 "On a fully static site, client-side navigation reuses the data captured at build time, so data can be stale until the next rebuild."。第三件是 `botRegex`（属于流式渲染那一节，不是预渲染页的内容）："A request whose `user-agent` header matches the pattern is **not** streamed: it receives the fully-rendered, buffered HTML."，且 "Setting `botRegex` replaces the default pattern rather than adding to it, so list every crawler you want to keep out of streaming"。顺带把预渲染这一页**没有**提 head、没提 meta、也没提 `robots.txt` 内容规则这件事说清楚——它只在目录结构那一页出现过一次文件名示例。

categories:
  - 前端开发

tags:
  - nuxt4
  - 预渲染
  - SEO
  - payload
  - botRegex

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


我原来把"预渲染"当成性能优化词，直到有次一个页面在站点里存在但没被任何页面链接，抓取工具死活找不到它，才回去读这一页。这一页的定位句其实就说明了它服务两个目标：

> Nuxt allows pages to be statically rendered at build time to improve certain performance or SEO metrics

注意是 "certain"（某些），不是全部——这也解释了我下面的困惑。

## 第一件：能预渲染哪些页，由链接决定

`nuxt generate` 与 `nuxt build --prerender` 的行为（原文）：

> Use the [`nuxt generate` command] to build and pre-render your application using the [Nitro] crawler. This command is similar to `nuxt build` with the `nitro.static` option set to `true`, or running `nuxt build --prerender`.

> This will build your site, stand up a nuxt instance, and, by default, prerender the root page `/` along with any of your site's pages it links to, any of your site's pages they link to, and so on.

四步机制照抄：

> 1. Load the HTML of your application's root route (`/`), any non-dynamic pages in your `~/pages` directory, and any other routes in the `prerender.routes` array.
> 2. Save the HTML and `_payload.json` to the `~/.output/public/` directory to be served statically.
> 3. Find all anchor tags (`<a href="...">`) in the HTML to navigate to other routes.
> 4. Repeat steps 1-3 for each anchor tag found until there are no more anchor tags to crawl.

> This is important to understand since pages that are not linked to a discoverable page can't be pre-rendered automatically.

第 3 步那句是我这次最想要的信息： crawler 找路靠的是 **HTML 里的 `<a href>`**。这有三个直接推论，我按文档能支撑的程度写：

| 推论 | 支撑 |
| --- | --- |
| 只在 JS 里动态拼出来的链接，抓取器在构建期看不见 | 机制第 3 步只说 "Find all anchor tags in the HTML" |
| 需要预渲染但没人链接的页面，得手动进 `prerender.routes` | 第 1 步把 `prerender.routes` 列为独立来源；示例正是 `/sitemap.xml`、`/robots.txt` 这两个"爬不到但必须存在"的路径 |
| `<a>` 之外的导航方式不产生预渲染 | 文档只写了 anchor tag 一种发现途径，没写别的 |

三种手动指定入口，文档都给了例子：

```ts [选择性预渲染]
export default defineNuxtConfig({
  prerender: {
    routes: ['/user/1', '/user/2'],
    ignore: ['/dynamic'],
  },
})
```

```ts [和 crawlLinks 组合]
export default defineNuxtConfig({
  prerender: {
    crawlLinks: true,
    routes: ['/sitemap.xml', '/robots.txt'],
  },
})
```

```ts [routeRules 形态]
export default defineNuxtConfig({
  routeRules: {
    '/rss.xml': { prerender: true },
    '/this-DOES-NOT-get-prerendered': { prerender: false },
    '/blog/**': { prerender: true },
  },
})
```

`'/blog/**'` 那条注释值得读："Everything under /blog gets prerendered as long as it is linked to from another page"——**即使规则命中了，仍然要求它被链接到**。规则开权限，链接负责发现，两件事都要满足。

还有一句容易漏的配置边界：

> The top-level `prerender` option applies whichever `server.builder` you use. Builder-specific options, such as Nitro's `concurrency` or `failOnError`, can be set in `nitro.prerender`.

运行时也能补登记路由，`prerenderRoutes(['/some/other/url'])` 在 Nuxt 上下文里可用；构建期还有两个钩子：`prerender:routes`（`ctx.routes.add(...)`，示例从远端 CMS 拉列表）和 Nitro 侧的 `prerender:generate`（`route.skip = true` 可以按路径名跳过，示例是包含 `private` 就跳过）。**后者是"别把私密页预渲染出去"的官方抓手**，我觉得这一条比任何 head 技巧都实在。

## 第二件：静态站的抓取者看到的时间点被冻结在构建期

Payload Extraction 一节原文（节选）：

> When Nuxt renders a page on the server, it serializes the results of your data fetching (`useAsyncData` and `useFetch`) and app state (`useState`) into a payload so the client can hydrate without re-fetching. With payload extraction enabled, Nuxt also writes this payload to a `_payload.json` file alongside the route's HTML:
> - Prerendered routes generate their payload file at build time.
> - Routes using ISR or SWR caching generate their payload file when the route is first rendered, even on a hybrid (non-static) site.

三条实践后果里，第一条直接关系到"抓取者读到什么"：

> On a fully static site, client-side navigation reuses the data captured at build time, so data can be stale until the next rebuild.

我把这句读成：**静态站的 head 与内容，快照时间都是构建那一刻**。文档这里说的是数据（payload），但同一段第 2 步写的是 "Save the HTML and `_payload.json`"——HTML 和 payload 一起落盘。所以对纯静态站来说，"内容更新了但抓取者还没看到"通常是"还没重新构建"，不是 head 写错了。这一句是我这次最想留给自己的诊断顺序。

## 第三件：botRegex 只属于流式那条路

`botRegex` 不在预渲染页里，它出现在流式渲染那节，原话是：

> You can customise which user agents are opted out with `botRegex`. A request whose `user-agent` header matches the pattern is **not** streamed: it receives the fully-rendered, buffered HTML. Requests that do not match are streamed. Setting `botRegex` replaces the default pattern rather than adding to it, so list every crawler you want to keep out of streaming:

```ts
botRegex: /googlebot|bingbot|my-internal-crawler/i,
```

另外那句把匹配到的请求列进了"不流式"的清单：

> Bot/crawler user agents, that is, requests whose `user-agent` matches `botRegex`

这一段和 193/198 有关，因为那句 "Anything else (a crawler your `botRegex` does not list, a link unfurler, `curl`) will not see those tags at all." 说的就是**没被列进去的抓取者会拿到流式 HTML，因而看不见晚注册的 head 标签**。三个容易混的点我一并理清：

| 情形 | head 是否完整 | 依据 |
| --- | --- | --- |
| 预渲染（`nuxt generate` / `routeRules.prerender`） | 是，HTML 与 payload 一起在构建期写盘 | 机制第 2 步 |
| 在线 SSR 且流式开启，`user-agent` 命中 `botRegex` | 是，走缓冲渲染 | botRegex 原话 "receives the fully-rendered, buffered HTML" |
| 在线 SSR 且流式开启，未命中 | 晚注册的标签看不见（JSON-LD、noscript、body 位置除外） | 193 引的那段 warning |

所以"我这个路由要不要退出流式"在文档里给的是 `routeRules: { '/path': { streaming: false } }`（193 末尾也抄过）。

## 这一页没写的几件事

我不替它补：`getting-started/prerendering` 整页**没有**出现 head、meta、SEO 标签相关的操作建议；`robots.txt` 只在两处出现——目录结构页把它当作"必须保名的文件"举例（`public/robots.txt`），本页把它当作"爬不到但要预渲染"的路径举例。**内容规则该怎么写不在 Nuxt 文档范围里**。同理，预渲染页也没提 sitemap 怎么生成：整套语料里 `sitemap` 作为功能出现的只有三处——本页的 `routes: ['/sitemap.xml', '/robots.txt']`、`directory-structure/server` 把 `/sitemap.xml` 当作动态服务端路由的例子、以及 Kit 那页一个名为 `nuxt-sitemap` 的模块示例（`name: 'nuxt-sitemap'`）。**没有 sitemap 模块的使用文档页**，要写得去模块自己的站点查。

## 自查

1. crawler 在构建期靠什么发现下一页？（答：机制第 3 步——HTML 里的 anchor tag `<a href="...">`。）
2. 规则命中但页面没被任何页面链接，会预渲染吗？（答：不会，"as long as it is linked to from another page" 与 "can't be pre-rendered automatically"。）
3. 静态站内容改了但抓取者还看到旧数据，先怀疑什么？（答：先怀疑"还没重新构建"——"data can be stale until the next rebuild"。）

## 下一篇

201 - OG 图片到底写相对还是绝对：两处示例给的是不同写法

## 本文依据

- `getting-started/prerendering`：页首定位句 "…to improve certain performance or SEO metrics"；"Nuxt will serve the prebuilt pages when requested instead of generating them on the fly."；`nuxt generate` 与 `nitro.static` / `nuxt build --prerender` 等价句；默认预渲染范围句；四步机制原文与 "pages that are not linked to a discoverable page can't be pre-rendered automatically."；`.output/public` 与 `npx serve .output/public`；`200.html` / `404.html` 兜底页一句；`prerender.routes` / `ignore` / `crawlLinks` 与 `routeRules.prerender` 三块配置示例（含 `'/blog/**'` 的注释）；"The top-level `prerender` option applies whichever `server.builder` you use. Builder-specific options, such as Nitro's `concurrency` or `failOnError`, can be set in `nitro.prerender`."；Payload Extraction 全段（含 "Prerendered routes generate their payload file at build time."、三条 `experimental.payloadExtraction` 取值、默认 `true` 与 cv5 下 `'client'`、`ssr: false` 强制 `false`、"data can be stale until the next rebuild"、devalue reducer/reviver 一句）；`prerenderRoutes`、`prerender:routes`、`prerender:generate`（`route.skip = true`）三个小节。
- `guide/going-further/experimental-features`（streaming）：`botRegex` 两句原文与 `/googlebot|bingbot|my-internal-crawler/i` 示例；"Bot/crawler user agents, that is, requests whose `user-agent` matches `botRegex`"；"Anything else (a crawler your `botRegex` does not list, a link unfurler, `curl`) will not see those tags at all."；退出流式的 `routeRules: { '/path': { streaming: false } }`。
- `directory-structure/public`：`robots.txt` 作为"必须保名文件"的例子（199/201 也引过这页）。
