---
url: /posts/3069492d9bde6340327d53fbdc013d7f/
title: 205 - robots 的两处记载：meta 里那个字段与一个要保名的文件
date: 2026-12-19T10:00:00+08:00
updated: 2026-12-19T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：控制爬虫有两条完全不同的路：页面里那条 meta，和站点根目录那个文本文件。Nuxt 文档对两条都只写了很少的话，我把每一句都摆出来。**meta 那条**：`robots` 这个键在语料里只以值形态出现两次，两处都是 `useSeoMeta({ robots: 'index, follow', ... })`，而且**都在 `if (import.meta.server)` 块里**——紧挨着的理由句是 "In most instances, SEO meta tags don't need to be reactive as search engine robots primarily scan the initial page load."（190 已经讲过这一招，这里只讲它为什么特别适合 robots）。文档没有 robots 的字段说明、没有可选值清单。**文件那条**：`robots.txt` 出现三处——`public` 页把它当作"必须保名的文件"举的例子（"Files contained within the `public/` directory are served at the root and are not modified by the build process."）、Kit `addServerHandler` 用整节演示怎么注册 `route: '/robots.txt'` 并给出响应 `User-agent: *` / `Disallow: /`、预渲染页告诉你它属于"爬不到所以要手动进 `prerender.routes`"那一类。还有一条与爬虫直接相关：`experimental.ssrStreaming` 那节写着 "Streaming is automatically disabled for bot and crawler user agents (such as Googlebot, Bingbot, etc.) to ensure search engines receive fully-rendered HTML for SEO safety."，而且默认模式只匹配索引爬虫，**Lighthouse 这类审计工具是刻意排除在外的**。**另有一处必须说清的空白：`X-Robots-Tag`、`noindex`、`nofollow` 三个词在整套 4.x 语料里零命中。**

categories:
  - 前端开发

tags:
  - nuxt4
  - robots
  - SEO
  - Nitro
  - 爬虫

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先解释这两个东西分别管什么，因为它们的生效范围不一样：

- **meta 里的 `robots`**：只影响爬虫读**当前这一页**——这一页要不要被收录、页上的链接要不要被跟。
- **站点根目录的 robots 文件**：是爬虫进来时读的第一份文本，管的是**哪些路径不许抓**，跟单页无关。

两者互不替代，所以我按两条线各列一次文档记载。

## 第一条线：meta 里的 robots

官方那一节给的就是这两个块。`useSeoMeta` 那一节里性能小节的原文示例：

```vue [app/app.vue]
<script setup lang="ts">
if (import.meta.server) {
  // These meta tags will only be added during server-side rendering
  useSeoMeta({
    robots: 'index, follow',
    description: 'Static description that does not need reactivity',
    ogImage: 'https://example.com/image.png',
    // other static meta tags...
  })
}
</script>
```

`useServerSeoMeta` 那一节（已废弃的那个组件）给的替代写法是同一块代码。也就是说，**文档只在这一种上下文里写过 `robots`，而那种上下文是"给静态 meta 省掉客户端开销"**，不是在讲索引控制。它的配套理由句：

> In most instances, SEO meta tags don't need to be reactive as search engine robots primarily scan the initial page load.

这句话对 robots 特别贴切：搜索引擎读的是首屏 HTML 里那一行，你在水合之后把它改掉，对爬虫基本没意义。所以 190 那条建议（静态 meta 包进 `import.meta.server`）用在 robots 上是最合适的场景之一。

文档没给的东西我不补：`robots` 的合法值清单（`index`/`noindex`/`follow`/`nofollow`/`none`/`max-image-preview` 这一整套）**不在 Nuxt 文档里**，示例只出现了 `'index, follow'` 一个值；`useSeoMeta` 的 100 多个参数说明官方是甩给源码链接的（"There are over 100 parameters. See the full list of parameters in the source code."）。要按取值语义决定，得去搜索引擎的文档。

如果你担心这行值来自不可信来源，`useHeadSafe` 的白名单里 `meta` 一组是 `['name', 'property', 'charset', 'content', 'media']`——也就是说 `name="robots"` 加 `content` 这种形态是允许的，被剥掉的是 `http-equiv` 那类（189 抄过完整六组）。

## 第二条线：那个要保名的文件

`public` 页给 robots 的定位很准：

> Files contained within the `public/` directory are served at the root and are not modified by the build process. This is suitable for files that have to keep their names (e.g. `robots.txt`) *or* likely won't change (e.g. `favicon.ico`).

它列的目录树也是 `-| public/ ---| favicon.ico ---| og-image.png ---| robots.txt` 三个文件。这条路径适合**内容是固定的**场景：直接把文本放进去，文件名不许被构建改（因为爬虫就找这个地址）。

内容要按请求生成时，文档给的入口是 Kit 的 `addServerHandler`，整节示例就是拿 robots 做的：

```ts twoslash
import { addServerHandler, createResolver, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  setup (options) {
    const { resolve } = createResolver(import.meta.url)

    addServerHandler({
      route: '/robots.txt',
      handler: resolve('./runtime/robots.get'),
    })
  },
})
```

对应处理函数与官方声明的响应（两处都照抄，包括那个我觉得别扭的地方）：

```ts [runtime/robots.get.ts]
export default defineEventHandler(() => {
  return {
    body: `User-agent: *\nDisallow: /`,
  }
})
```

> When you access `/robots.txt`, it will return the following response:

```txt
User-agent: *
Disallow: /
```

**这个示例的形状我不敢替你美化**：处理函数返回的是带 `body` 键的对象，而官方紧接着说访问该地址得到的就是那两行纯文本。这两件事是不是在所有 Nitro 版本上都一致，文档没解释，所以我在自己项目里做这一步时会直接读返回的响应体确认，而不是照抄就上线。

第三条记载在预渲染页（200 已讲机制）：`prerender` 那一节特意把 robots 与 sitemap 列为"爬虫发现不了、要手动写进 `routes`"的例子——

```ts twoslash [nuxt.config.ts]
export default defineNuxtConfig({
  prerender: {
    crawlLinks: true,
    routes: ['/sitemap.xml', '/robots.txt'],
  },
})
```

原因是发现机制只认 HTML 里的 `<a href>`（"Find all anchor tags (`<a href=\"...\">`) in the HTML to navigate to other routes."）。**如果整站预渲染、又不手动登记这个地址，静态产物里就没有它**——`public/` 里的那份是原样拷贝，跟这条不同路径；用服务端 handler 动态生成的那份才需要登记。

## 爬虫拿到的 HTML 是完整的那一份

`experimental.ssrStreaming` 那节有一段专门讲爬虫，值得整句留着：

> Streaming is automatically disabled for bot and crawler user agents (such as Googlebot, Bingbot, etc.) to ensure search engines receive fully-rendered HTML for SEO safety. The default pattern matches indexing crawlers only; Lighthouse and other audit tools deliberately fall outside it so synthetic measurements reflect the same streamed response real users get.

这段话替我解释了一个很容易误判的现象：**同一个 URL，用爬虫身份访问和用浏览器访问，拿到的渲染方式可能不同**（一个走缓冲、一个走流式）。所以拿 Lighthouse 的读数去推断"搜索引擎看到的 head 全不全"是不成立的——官方写明了审计工具**刻意不在**匹配名单里，目的是让它测到真实用户那份流式响应。要换匹配名单，文档给的口子是 `botRegex`，而它的坑在 200 记过：设置它是**替换**默认模式而不是追加。

## 三个词的空白

我把 `X-Robots-Tag`、`noindex`、`nofollow` 在官方文档里检索过，**零命中**。含义说清楚：

- 文档没有提过用 HTTP 响应头来控制索引（`routeRules` 的 `headers` 能设任意头，但 Nuxt 文档没有把它和 robots 连起来写过一句）。
- 文档没有给过 robots 的具体取值语义。

这两条我不会编成"可以这样做"，只标成需要外部依据的空白。

## 自查

1. meta robots 与 robots 文件各自的生效范围有什么不同？（答：前者管单页是否被索引/跟链，后者管哪些路径不许抓，与单页无关。）
2. 文档里 `robots` 这个字段出现在什么样的代码块里，配套理由句是什么？（答：两处都在 `if (import.meta.server)` 包着的 `useSeoMeta` 示例里；理由是爬虫主要读首屏那一次加载。）
3. 全站预渲染时，用服务端 handler 生成的 robots 地址为什么会缺？（答：爬虫发现机制只认 HTML 里的 anchor tag，要手动进 `prerender.routes`。）

## 下一篇

206 - 301 还是 302：redirectCode 只管服务端那一次

## 本文依据

- `api/composables/use-seo-meta`：Performance 一节 "In most instances, SEO meta tags don't need to be reactive as search engine robots primarily scan the initial page load." 与含 `robots: 'index, follow'` 的 server-only 示例；Parameters 一节 "There are over 100 parameters. See the full list of parameters in the source code."
- `api/composables/use-server-seo-meta`：deprecated warning 与 `if (import.meta.server) { useSeoMeta({ robots: 'index, follow' }) }` 替代写法。
- `api/composables/use-head-safe`：`meta: ['name', 'property', 'charset', 'content', 'media']` 与 "http-equiv in meta tags are automatically stripped out"。
- `directory-structure/public`：定位句与目录树（`favicon.ico` / `og-image.png` / `robots.txt`）。
- `api/kit/nitro`：`addServerHandler` 定位句 "Adds a Nitro server handler. Use this if you want to create server middleware or a custom route."、`route: '/robots.txt'` + `handler: resolve('./runtime/robots.get')` 示例、`runtime/robots.get.ts` 返回 `{ body: \`User-agent: *\nDisallow: /\` }`、"When you access `/robots.txt`, it will return the following response:" 与其下 `User-agent: *` / `Disallow: /` 两块。
- `getting-started/prerendering`：`prerender: { crawlLinks: true, routes: ['/sitemap.xml', '/robots.txt'] }` 示例与四步机制第 3 步。
- `guide/going-further/experimental-features`（`ssrStreaming`）：爬虫自动退出流式那句、默认模式只匹配索引爬虫、Lighthouse 等审计工具刻意排除、`botRegex` 可自定义。
- 检索结论（否定性事实）：`X-Robots-Tag`、`noindex`、`nofollow` 在 4.x 语料中零命中。
