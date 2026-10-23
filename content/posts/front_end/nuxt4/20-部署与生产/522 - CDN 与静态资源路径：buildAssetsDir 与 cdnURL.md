---
url: /posts/3958d9d48af59ff67511e890be4c7014/
title: 522 - CDN 与静态资源路径：buildAssetsDir 与 cdnURL
date: 2027-04-04T09:00:00+08:00
updated: 2027-04-04T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：静态资源的 URL 由三个值拼出来：`app.baseURL`（默认 `/`）、`app.buildAssetsDir`（默认 `/_nuxt/`，**相对 `baseURL`，设了 `cdnURL` 就相对 `cdnURL`**，官方说明它在构建期固定、不该运行时定制）、`app.cdnURL`（默认空串，定义是"服务 public 目录的绝对 URL，**仅生产用**"）。`cdnURL` 可以在启动时用 `NUXT_APP_CDN_URL` 改。有一处官方点名的例外：设了 `cdnURL` 后 `public/` 里的资源（含 `favicon.ico`）会由 CDN 提供，但你在 `app.head` 里写死的 `href: '/favicon.ico'` 是字面路径，**不参与解析**——官方给的补法是用 `useRuntimeConfig().app` 拼出来。

categories:
  - 前端开发

tags:
  - nuxt4
  - cdn
  - 静态资源
  - 部署
  - 路径
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

把站点接到 CDN 上之后，最常见的现场是：JS 和 CSS 从 CDN 来了，但 `favicon.ico`、`og-image.png` 这些还是打原域名。不是 CDN 配错，而是这两类资源走的是**不同的拼接规则**。

## 三个值各管一段

| 配置 | 官方定义原句 | 类型 | 默认 |
| --- | --- | --- | --- |
| `app.baseURL` | The base path of your Nuxt application. | `string` | `"/"` |
| `app.buildAssetsDir` | The folder name for the built site assets, relative to `baseURL` (or `cdnURL` if set). This is set at build time and should not be customized at runtime. | `string` | `"/_nuxt/"` |
| `app.cdnURL` | An absolute URL to serve the public folder from (production-only). | `string` | `""` |

三行连读能得到一条链：**构建出来的资源**放在 `buildAssetsDir` 这个目录下，而这个目录是相对 `cdnURL`（没设时相对 `baseURL`）算的；**`cdnURL` 提供的是 public 目录**，而且官方限定它是 production-only。

`cdnURL` 的写法与运行时改法，官方都给了示例：

```ts
export default defineNuxtConfig({
  app: {
    cdnURL: 'https://mycdn.org/',
  },
})
```

> This can be set to a different value at runtime by setting the `NUXT_APP_CDN_URL` environment variable.

```bash
NUXT_APP_CDN_URL=https://mycdn.org/ node .output/server/index.mjs
```

这一点与 `buildAssetsDir` 那句"构建期固定、不该运行时定制"正好互补：**换 CDN 域名不需要重新构建，改资源目录名需要。**

`use-runtime-config` 那一节给了读取侧的对应说法：**"Nuxt uses `app` namespace in runtime-config with keys including `baseURL` and `cdnURL`. You can customize their values at runtime by setting environment variables."**，并且专门有一段讲 `app.cdnURL`：**"You can use a custom CDN for serving static assets inside `.output/public` using the `NUXT_APP_CDN_URL` environment variable."**——注意这句里的对象是 `.output/public` 里的静态资源。

## 那一处例外：写死的 head 链接不会被解析

`public/` 那一节说得很清楚，这些文件**保持原名、不被构建过程修改**："Files contained within the `public/` directory are served at the root and are not modified by the build process. This is suitable for files that have to keep their names (e.g. `robots.txt`) *or* likely won't change (e.g. `favicon.ico`)."

于是 SEO 那一节专门写了一条 `::note`，处理的就是设了 `cdnURL` 之后的边界：

> When you set `app.cdnURL`, assets in your `public/` directory (including `favicon.ico`) are served from that CDN. Nuxt resolves public assets against `cdnURL`, falling back to `app.baseURL`. However, a static `app.head` link such as `href: '/favicon.ico'` above is a literal path and is **not** resolved against `cdnURL`.

三句话拆开：**public 资源按 `cdnURL` 解析、解析不到时退回 `app.baseURL`**；但**写在 `app.head` 里的字面路径不参与这套解析**。官方的补法是从运行时配置里拼：

```vue [app/app.vue]
<script setup lang="ts">
const { cdnURL, baseURL } = useRuntimeConfig().app
useHead({
  link: [
    { rel: 'icon', type: 'image/x-icon', href: `${cdnURL || baseURL}favicon.ico` },
  ],
})
</script>
```

`cdnURL || baseURL` 这个写法本身也说了件事：**两个值都以斜杠结尾**，所以拼接时直接跟文件名，不再额外加 `/`。文档没写这条规则，是从示例形状能读出来的约定——照抄比自己拼安全。

## 与 app/assets 那条链的区别

`public/` 之外还有 `app/assets/`，官方对它的定性正相反：**交给构建器处理、不自动扫描、文件名可以变**。构建出来的产物才是要长缓存的那些文件，它们走的就是 `buildAssetsDir` 这个目录（默认 `/_nuxt/`）。

两条链放在一起：

| 资源 | 位置 | 名字 | URL 由什么决定 |
| --- | --- | --- | --- |
| 构建产生的 JS/CSS 等 | `.output` 里的构建产物 | 由构建器决定 | `cdnURL` 或 `baseURL` ＋ `buildAssetsDir` |
| 保持原名的文件 | `public/` | **不改名**（官方举例 `robots.txt` 要保名、`favicon.ico` 大概不会变） | Nuxt 按 `cdnURL` 解析、退回 `baseURL`；**字面量路径例外** |
| `app/assets/` 里的资源 | 交给构建器 | **可以变**（官方明说可换名） | 走构建器解析 |

public 那一节还给了一个引用示例，说明这些文件在代码里就是按根路径写的：

```vue [app/app.vue]
<script setup lang="ts">
useSeoMeta({
  ogImage: '/og-image.png',
})
</script>
```

目录树官方列的是三行：`favicon.ico`、`og-image.png`、`robots.txt`。末尾一条历史提示：这个目录在 Nuxt 2 叫 `static/`。

`ogImage: '/og-image.png'` 这种写法正好落在上一节那条例外的边上——它也是一个以 `/` 开头的字面路径。文档对 `useSeoMeta` 里的值是否按 `cdnURL` 解析**没有单独说明**，只写了"public assets"这一句，所以设了 CDN 之后要确认社交图片到底从哪来：**在浏览器开发者工具的 Network 面板看那张图实际请求的主机名**，比按规则推断可靠。

## 顺手一条：CDN 会注入脚本

用 Cloudflare 时官方专门警告要关掉两项（Rocket Loader 与邮箱地址混淆），理由是否则**"unnecessary re-rendering or hydration errors could impact your production application"**。这两项的完整路径与后果我在讲 CDN 代理那篇单独列，这一篇只留一句：**资源路径对了但页面出现水合异常时，先看 CDN 有没有往 HTML 里注入脚本。**

## 自查

- 三个值分工：`baseURL` 是应用基路径、`buildAssetsDir` 是构建资源目录名（相对 `cdnURL` 或 `baseURL`，构建期固定）、`cdnURL` 是服务 public 目录的绝对 URL 且仅生产用。
- 换 CDN 域名可以用 `NUXT_APP_CDN_URL` 在启动时改，不必重新构建；`buildAssetsDir` 官方说不该运行时定制。
- Nuxt 按 `cdnURL` 解析 public 资源、回退 `baseURL`；**`app.head` 里写死的 `/favicon.ico` 是字面路径，不被解析**，官方补法是用 `useRuntimeConfig().app` 拼。
- `public/` 不改文件名（要保名的文件放这里），`app/assets/` 交给构建器且可以改名——两条链的缓存策略因此不同。
- `useSeoMeta` 里的绝对路径是否走 CDN，文档未单独说明，用 Network 面板实测确认。

## 下一篇

下一篇 523 讲启动脚本与健康检查：部署页给出的 PM2 配置里那五行字段、`nuxt cleanup` 清了哪些目录，以及文档在这两件事上能给到哪一步。
