---
url: /posts/362255a81a99fe167fc4a55033388277/
title: 521 - swr 与 isr：两种缓存策略与它们的平台前提
date: 2027-04-03T11:00:00+08:00
updated: 2027-04-03T11:00:00+08:00
author: cmdragon

summary:
  摘要：`swr` 与 `isr` 都接受 `number | boolean`，行为描述里只有一处差别：**`isr` 除了 `swr` 的一切，还能把响应放进 CDN 缓存——前提是平台支持，官方当前点名 Netlify 与 Vercel**。`swr` 那行的机制是"给服务端响应加缓存头并缓存在服务器或反向代理上，TTL 可配"，并且明写 **Nitro 的 `node-server` 预设能够缓存整份响应**；TTL 到期后会先返回缓存内容、同时在后台重新生成；传 `true` 时加的是 `stale-while-revalidate` 头而**没有 MaxAge**。`isr` 传 `true` 时官方说的是"内容在 CDN 里一直留到下次部署"。这份路由规则属性表一共九项，**里面没有 `cache` 这个键**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 缓存
  - isr
  - swr
  - 部署
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`{ swr: 3600 }` 和 `{ isr: 3600 }` 差一个字母。选错的后果不是报错，而是你以为缓存到了 CDN、其实只在服务器本机。

## 官方那份属性表：九项，逐行

Route Rules 小节给的可用属性一共九个，原文照列：

| 属性 | 官方说明（原句要点） |
| --- | --- |
| `redirect: string` | Define server-side redirects. |
| `ssr: boolean` | 关掉该区块的 HTML 服务端渲染，只在浏览器里渲染 |
| `cors: boolean` | `cors: true` 自动加 cors 头；要定制输出可以用 `headers` 覆盖 |
| `headers: object` | 给站点某些区块加特定响应头，例如静态资源 |
| `swr: number \| boolean` | 给服务端响应加缓存头，缓存在服务器或反向代理上，TTL 可配；`node-server` 预设能缓存整份响应；TTL 到期后返回缓存内容并在后台重新生成；传 `true` 时加 `stale-while-revalidate` 头且没有 MaxAge |
| `isr: number \| boolean` | 行为与 `swr` 相同，差别是**在支持的平台（当前 Netlify 或 Vercel）上可以把响应放进 CDN 缓存**；传 `true` 时内容在 CDN 里持续到下次部署 |
| `prerender: boolean` | 构建期预渲染这些路由，并把它们作为静态资源包含进构建产物 |
| `noScripts: boolean` | 禁用该区块的 Nuxt 脚本与 JS resource hint 渲染 |
| `appMiddleware: string \| string[] \| Record<string, boolean>` | 定义 Vue 应用部分里那些页面路径应当或不应当跑的中间件（**不含你的 Nitro 路由**） |

这张表本身有一条信息量很大的"缺席"：**九个属性里没有 `cache`**。缓存这件事在这份表里只有 `swr` 与 `isr` 两个入口。

## 两者的差别只在 CDN 那一层

把两行说明并排读，`isr` 那句写的是 "The behavior is the same as `swr` except that we are able to add the response to the CDN cache on platforms that support this (currently Netlify or Vercel)."

也就是说：

- **共同部分**：按需生成（generated on demand）、后台重新验证（revalidates in background）、响应带缓存头。
- **差别**：`isr` 多一层"能不能进 CDN 缓存"，而且官方立刻补了前提——**只有支持这能力的平台才行，当前是 Netlify 与 Vercel**。
- **`true` 的含义不同**：`swr: true` 是"加 `stale-while-revalidate` 头、不带 MaxAge"；`isr: true` 是"内容在 CDN 里留到下次部署"。官方在示例注释里就是这么写的（`'/blog/**': { isr: true }` 那行注释是 "generated on demand once until next deployment, cached on CDN"）。

`node-server` 那句是第三种常见误解的来源。原文只说 **"The `node-server` preset of Nitro is able to cache the full response."**——它讲的是**这个预设下整份响应能被缓存住**，并没有说缓存会出现在 CDN 上。你自己用 Node 托管时，`swr`/`isr` 的缓存在这一层；前面还挂了 CDN 的话，`isr` 才有机会落到 CDN。

## TTL 的单位与两处佐证

数字的语义在示例注释里给了换算：`{ swr: 3600 }` 与 `{ isr: 3600 }` 的注释都写着 **"cached for 1 hour (3600 seconds)"**。所以这两项传的数字单位是**秒**，不是毫秒。

对照另一处会更清楚：`experimental.checkOutdatedBuildInterval` 那项官方明写单位是 **毫秒**（示例 `3600000 // 1 hour`）。两个"一小时"在文档里写成 `3600` 与 `3600000`，**别把秒和毫秒混着填**——填错的表现是缓存时间差一千倍，不会报错。

## 动态路由怎么开：官方给的 glob 写法

预渲染那一节讲载荷提取时，专门给了动态路由的开启方式：**"Dynamic routes like `pages/[...slug].vue` can opt in with glob patterns such as `'/**': { isr: true }`."**

这行的重点是**glob 匹配的是 URL，不是文件路径**：`'/**'` 覆盖全站所有路由，包括 catch-all 那种动态段。而示例里 `'/products/**': { swr: 3600 }`、`'/blog/**': { isr: true }` 是按 URL 前缀分组——这与混合渲染那节"给一组路由定义规则"的说法一致。

## 这两种路由也会产出 `_payload.json`

那条 `::note` 把缓存策略和载荷联系起来了：

> Routes using `isr` or `swr` also generate `_payload.json` files alongside HTML. Client-side navigation loads these cached payloads instead of re-fetching data.

预渲染那一节补得更细：**首次渲染时就生成**（"Routes using ISR or SWR caching generate their payload file when the route is first rendered, even on a hybrid (non-static) site"），并且**CDN 可以把载荷文件和 HTML 一起缓存**，从而提升已缓存路由的客户端导航性能。

对部署的含义：**你缓存的不止 HTML，还有一份数据**。所以"改了内容但页面没变"这类问题，要看两层缓存；另外载荷是用 devalue 序列化的，自定义类型（比如类实例）需要载荷插件的 reducer 与 reviver 才能往返——这条在讲预渲染那篇有原文。

## 一个反面提醒：`noScripts` 也在同一张表里

`noScripts` 与缓存同在 routeRules，容易被当成"给静态页省 JS"的顺手配置。但它在特性那一节还有一句默认值说明（官方给的默认是 `'production'`），而错误码 `e8007` 讲的正是这个默认造成的开发期差异：**dev 里有 JS、生产剥掉 JS**，两类会静默坏掉的能力是延迟水合与服务端组件里的 `nuxt-client`。给某个区块设 `noScripts` 之前，先确认那一节没有依赖水合的交互。

## 自查

- `swr` 与 `isr` 都是 `number | boolean`；`isr` 的全部差别是"能把响应放进 CDN 缓存"，且官方限定当前只有 Netlify 与 Vercel 支持。
- 传 `true` 时两者含义不同：`swr: true` 加 `stale-while-revalidate` 且不带 MaxAge；`isr: true` 是内容在 CDN 留到下次部署。
- 数字单位是**秒**（示例注释 "3600 seconds" = 1 hour），与 `checkOutdatedBuildInterval` 的毫秒不要混。
- "`node-server` 预设能缓存整份响应"只说明本机这层缓存存在，不等于进了 CDN。
- 动态路由用 glob 开（`'/**': { isr: true }`），匹配的是 URL；这两种路由首次渲染时就生成 `_payload.json`，且 CDN 可连载荷一起缓存。
- 属性表九项里**没有 `cache`**；`noScripts` 也在同一张表里，设之前要确认该区块不依赖水合交互。

## 下一篇

下一篇 522 讲静态资源的路径链条：`app.baseURL`、`buildAssetsDir`（默认 `/_nuxt/`）与 `cdnURL`（生产专用）三者怎么拼出资源 URL，以及 `public/` 目录为什么是唯一保持原名的那部分。
