---
url: /posts/fc9214ebcf5191823391a5442c9b8502/
title: 513 - 静态托管的两条路：generate 与不带服务端的构建
date: 2027-04-01T09:00:00+08:00
updated: 2027-04-01T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：静态托管只有文件、没有进程，所以构建时必须把要的东西都渲出来。官方给两条路：`nuxt generate`（等价于 `nuxt build --prerender`，也等价于把 `nitro.static` 设为 `true`），或者 `ssr: false` 的纯客户端单页。前者的预渲染靠 **Nitro 爬虫**：从根页面出发，靠 HTML 里的 `<a href="...">` 找下一批路由，所以**没被任何可发现页面链接起来的路由不会自动预渲染**——补法有三种（`prerender.routes`、`crawlLinks` 加显式路由、`routeRules` 的 `prerender: true`）。产物在 `.output/public`，可以本地 `npx serve` 预览。代价也要知道：预渲染后 runtimeConfig 无法再设、混合渲染在 `nuxt generate` 下不可用。

categories:
  - 前端开发

tags:
  - nuxt4
  - 静态托管
  - 预渲染
  - generate
  - 部署
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

把站点扔到静态托管上，第一个反应通常是"部署成功了，可有些页面 404"。那些页面在服务器上本来是能渲染的，但静态主机只有文件——**没在构建时渲出来的路由，就不存在**。

官方围绕这件事给了两条路和一组规则。

## 第一条路：nuxt generate

命令页的定性：**"The `generate` command pre-renders every route of your application and stores the result in plain HTML files that you can deploy on any static hosting services."**，紧接着给出实现方式："The command triggers the `nuxt build` command with the `prerender` argument set to `true`"。

预渲染页那句等价关系把三种写法串起来：**"This command is similar to `nuxt build` with the `nitro.static` option set to `true`, or running `nuxt build --prerender`."**

命令形状与选项：

```bash [Terminal]
npx nuxt generate [ROOTDIR] [--cwd=<directory>] [--logLevel=<silent|info|verbose>] [--target=<target>] [--dotenv=<path>...] [--envName=<environment>] [-e, --extends=<layer-name>...] [--profile=<verbose>]
```

| 选项 | 官方说明 |
| --- | --- |
| `ROOTDIR` | 项目的根目录（默认 `.`） |
| `--cwd=<directory>` | 指定 Nuxt 项目根目录 |
| `--logLevel=<silent\|info\|verbose>` | 指定 build-time 日志级别 |
| `--target=<target>` | 配置好的 server builder 的部署目标（示例给 `node-server`、`vercel`、`netlify`） |
| `--dotenv=<path>...` | 要加载的 `.env` 路径，相对根目录，可重复，后面的文件优先 |
| `--envName=<environment>` | 解析配置覆盖时用的环境（构建时默认 `production`，dev server 是 `development`） |
| `-e, --extends=<layer-name>...` | 从某个 Nuxt layer 扩展 |
| `--profile=<verbose>` | 性能剖析，退出时写 V8 CPU profile 与 JSON 报告 |

跑法五种都给了：`npx nuxt generate`、`yarn nuxt generate`、`pnpm nuxt generate`、`bun x nuxt generate`、`deno x nuxt generate`。注意这一节用 `x` 而不是 `add --dev` 那种形式，而且**deno 在这里是有的**（测试文档那两节就没给 deno）。

## 爬虫怎么决定渲哪些页

这是整页最重要的一段机制说明，官方列了四步：

1. 加载应用根路由（`/`）的 HTML、`~/pages` 目录里任何非动态页面，以及 `prerender.routes` 数组里的其他路由
2. 把 HTML 与 `_payload.json` 存进 `.output/public/` 目录供静态服务
3. 在 HTML 里找出所有 anchor 标签（`<a href="...">`）以导航到别的路由
4. 对找到的每个 anchor 标签重复第 1-3 步，直到没有更多可爬的 anchor

后面那句结论要单独记住：**"This is important to understand since pages that are not linked to a discoverable page can't be pre-rendered automatically."**

也就是说：页面在不在产物里，取决于**有没有链接指向它**，而不是它是否存在于 `pages/`。你的文章列表页如果只渲染了 10 篇、第 11 篇没有任何 `<a>` 指向它，第 11 篇就不会被渲出来。

三种补法官方都给了示例：

```ts twoslash [nuxt.config.ts]
export default defineNuxtConfig({
  prerender: {
    routes: ['/user/1', '/user/2'],
    ignore: ['/dynamic'],
  },
})
```

```ts twoslash [nuxt.config.ts]
export default defineNuxtConfig({
  prerender: {
    crawlLinks: true,
    routes: ['/sitemap.xml', '/robots.txt'],
  },
})
```

以及按路由规则：

```ts twoslash [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    // Set prerender to true to configure it to be prerendered
    '/rss.xml': { prerender: true },
    // Set it to false to configure it to be skipped for prerendering
    '/this-DOES-NOT-get-prerendered': { prerender: false },
    // Everything under /blog gets prerendered as long as it
    // is linked to from another page
    '/blog/**': { prerender: true },
  },
})
```

三段各有分工：`routes` 是**手工点名**，`ignore` 是**手工排除**，`crawlLinks` 是让爬虫工作并把爬虫发现不了的（`/sitemap.xml`、`/robots.txt`）加进来。第三段最后那行注释是官方原话里带的条件——`/blog/**` 下的东西要渲，前提**是它被别的页面链接到了**。

两条边界写在同一段里：顶层的 `prerender` 选项对你用的**任何一种 `server.builder` 都生效**；而 Nitro 特有的选项（官方举了 `concurrency` 与 `failOnError`）要写在 `nitro.prerender` 里。

还有一种更靠后的入口，是给"路由要跑起来才知道"的场景：运行时在 Nuxt 上下文里调 `prerenderRoutes([...])` 追加路由；或者用 `prerender:routes` 这个 Nuxt 钩子在建前从外部 CMS 拉一批名字 `ctx.routes.add(...)`；或者用 `prerender:generate` 这个 Nitro 钩子按路由逐个处理，官方示例就是把路径里含 `private` 的 `route.skip = true`。

## 产物与本地预览

一句可执行的话：**"You can now deploy the `.output/public` directory to any static hosting service or preview it locally with `npx serve .output/public`."**

另外静态与预渲染构建会同时产出 `200.html` 与 `404.html` 两个 SPA 兜底页。这两个文件谁生成、宿主怎么指过去，我在讲兜底页那篇单独说。

## 第二条路：ssr 设为 false

预渲染之外，官方还给了"干脆不要服务端"的路径。部署页的措辞是：可以 `ssr: false` 预渲染站点（静态单页应用），这会产出**在本来要渲染 Vue 应用的位置放一个空 `<div id="__nuxt"></div>`** 的 HTML 页；并且直接说明 **"You will lose many SEO benefits of prerendering your site, so it is suggested instead to use `<ClientOnly>` to wrap the portions of your site that cannot be server rendered (if any)."**

也就是说：**这条路官方自己评价为丢 SEO，并推荐只在必须只在浏览器渲染的部分套 `<ClientOnly>`**，而不是整站走这条路。

命令侧的对应行为写在同页另一段：`ssr: false` 时 `nuxt generate` 输出 `.output/public/index.html` 入口加 JS bundle，"like a classic client-side Vue.js applications"。

## 静态主机拿走的两样东西

选这条路要知道的代价，文档有两处明确写了：

- **预渲染之后没法再设运行时配置。** 环境变量那一节有一句：**"Note that for a purely static site, it is not possible to set runtime configuration config after your project is prerendered."** 你的 `runtimeConfig` 值在构建时就固定进产物了，改它要重新构建。
- **混合渲染在 `nuxt generate` 下不可用。** 渲染概念页那条 `::important` 写着 "Note that Hybrid Rendering is not available when using `nuxt generate`." 想用 `swr`/`isr` 这类按路由的缓存策略，就得有一个跑着的服务器。

顺带一条 payload 的：预渲染的路由在构建期就产出 `_payload.json`，客户端导航复用它——**"so data can be stale until the next rebuild"**，官方这句直译就是"数据会一直旧到下次构建"。

## 自查

- `nuxt generate` ＝ 带 prerender 的 build（三种等价写法：命令本身、`nuxt build --prerender`、`nitro.static: true`）。
- 渲哪些路由由 Nitro 爬虫决定：**没有链接指向它就不会被自动预渲染**；补法是 `prerender.routes`、`crawlLinks`、`routeRules` 的 `prerender: true`，动态场景用 `prerenderRoutes` 或两个钩子。
- 顶层 `prerender` 对任何 server builder 都生效，`concurrency`/`failOnError` 这类要写进 `nitro.prerender`。
- 产物目录是 `.output/public`，本地用 `npx serve .output/public` 预览；同时产出 `200.html`/`404.html`。
- `ssr: false` 那条路官方评价为丢 SEO、建议只对不能服务端渲染的部分用 `<ClientOnly>`。
- 静态站两个硬限制：预渲染后 runtimeConfig 改不了、`nuxt generate` 下没有混合渲染。

## 下一篇

下一篇 514 讲另一侧：Node 服务器方式部署——入口文件怎么起、`node-server` 预设的三条定性、以及 Nitro 那份"独立于 node_modules 的 dist"意味着你能只搬一个目录。
