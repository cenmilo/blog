---
url: /posts/5b29f7f02c629718d210ccd666331214/
title: 361 - 部署在子路径与换 CDN：app.baseURL、buildAssetsDir 与 cdnURL
date: 2027-02-09T10:00:00+08:00
updated: 2027-02-09T10:00:00+08:00
author: cmdragon

summary:
  摘要：本地跑得好好的网站，一部署到 `https://站点/app/` 这种子路径下就白屏、JS 和 CSS 全 404，多半是这三项没配。`app` 节专管"应用挂在哪、资源从哪取"：`baseURL` 是整站的挂载路径前缀（默认 `/`），`buildAssetsDir` 是打包后静态资源的目录前缀（默认 `/_nuxt/`），`cdnURL` 是生产环境下 public 目录的绝对地址。前两个能用环境变量在运行时覆盖，`baseURL` 还不支持在 nuxt.config 里写相对路径。这一篇逐行讲清三项的默认值、覆盖方式，以及官方点名的两个坑。

categories:
  - 前端开发

tags:
  - nuxt4
  - baseURL
  - buildAssetsDir
  - cdnURL
  - 部署
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`nuxt dev` 下网站在根路径 `/` 一切正常。把 `.output` 丢到服务器上、让它跑在 `https://example.com/app/`，页面就空白了——打开控制台，`/_nuxt/xxx.js` 请求 404。这就是 `app` 节要解决的：应用挂在哪个路径下、静态资源去哪个目录取。

## 先说两个词

**挂载路径前缀**：整个网站前面加的那段路径。根路径部署是 `/`，子路径部署是 `/app/`。页面里所有"从网站根算起"的链接都得带上这段。

**相对路径**：像 `./` 这种"相对于当前位置"的写法，和 `/app/` 这种"相对于站点根"的绝对路径相反。静态托管（比如把生成的文件直接扔进某个目录）常用相对路径，因为它不假定自己挂在哪个域名或子路径下。

## `app` 节的三项

nuxt-config 里 `## app` 只有一句话："Nuxt App configuration."，底下挂三个键。

### `baseURL`

> The base path of your Nuxt application.

- 类型：`string`
- 默认：`"/"`

```ts
export default defineNuxtConfig({
  app: {
    baseURL: '/prefix/',
  },
})
```

它也可以不设进配置文件，改用环境变量在运行时指定：

```bash
NUXT_APP_BASE_URL=/prefix/ node .output/server/index.mjs
```

::note
`:::important` 那种块 nuxt-config 这里是 `::note`，原话：Relative paths (like `./`) are not supported directly in `nuxt.config.ts` due to Nitro limitations. For static hosting with relative asset paths, use one of these approaches:

方案一，构建时用环境变量：

```bash
NUXT_APP_BASE_URL=./ npm run generate
```

方案二，走 runtimeConfig（注意是写进 `runtimeConfig.app`）：

```ts
export default defineNuxtConfig({
  runtimeConfig: {
    app: {
      baseURL: './',
    },
  },
})
```

::

也就是说：`app.baseURL: './'` 这种直接写相对路径在 `nuxt.config.ts` 里是**不行的**（Nitro 的限制），要相对路径只能绕上面两条。

### `buildAssetsDir`

> The folder name for the built site assets, relative to `baseURL` (or `cdnURL` if set). This is set at build time and should not be customized at runtime.

- 类型：`string`
- 默认：`"/_nuxt/"`

这是打包后 JS/CSS 这些资源所在的目录名，它是**相对 `baseURL` 的**（如果设了 `cdnURL` 就相对它）。所以最终资源地址是 `baseURL` 拼上 `buildAssetsDir`：默认就是 `/_nuxt/`，配了 `baseURL: '/prefix/'` 就变成 `/prefix/_nuxt/`。

文档特别点了一句：它是构建期确定的，**运行期不该改**。

### `cdnURL`

> An absolute URL to serve the public folder from (production-only).

- 类型：`string`
- 默认：`""`

```ts
export default defineNuxtConfig({
  app: {
    cdnURL: 'https://mycdn.org/',
  },
})
```

给 `public` 目录提供一个绝对地址，只在生产环境生效。同样能用环境变量在运行时改成别的值：

```bash
NUXT_APP_CDN_URL=https://mycdn.org/ node .output/server/index.mjs
```

设了它，资源就从 `cdnURL` 拼 `buildAssetsDir` 去取，而不是 `baseURL`。

## 三项对照

| 键 | 默认值 | 管什么 | 运行时覆盖变量 | 覆盖时机 |
| --- | --- | --- | --- | --- |
| `app.baseURL` | `"/"` | 整站挂载路径前缀 | `NUXT_APP_BASE_URL` | 可以运行时设 |
| `app.buildAssetsDir` | `"/_nuxt/"` | 静态资源目录名 | 文档未给 | 构建期确定，运行期不该改 |
| `app.cdnURL` | `""` | public 目录绝对地址（生产） | `NUXT_APP_CDN_URL` | 可以运行时设 |

`resolve` 之后的配置里这三项就是 `"baseURL": "/"`、`"buildAssetsDir": "/_nuxt/"`、`"cdnURL": ""`——和默认值一致，能看到它们确实是 `app` 下的键。

## 部署侧怎么用

deployment 那页「Serving the Same Build at Multiple Paths」讲子路径部署就一句话：

> For a normal subpath deployment, set `app.baseURL` or the `NUXT_APP_BASE_URL` environment variable.

所以回到开头那个白屏：站到 `/app/` 下，要么 `app.baseURL: '/app/'`，要么启动时 `NUXT_APP_BASE_URL=/app/ node .output/server/index.mjs`，资源路径跟着就对了。

它还补了一个反向代理的场景：如果代理故意让同一个渲染结果对多个公开路径可见，水合时 Nuxt 可能把浏览器地址换成服务端渲染用的那个路径。想保住浏览器地址，就在服务端插件里把渲染路径从 payload 删掉：

```ts [app/plugins/preserve-proxy-url.server.ts]
export default defineNuxtPlugin((nuxtApp) => {
  delete nuxtApp.payload.path
})
```

文档提醒：只有在代理已经把每个公开路径的资源和路由都接管了时才这么干，否则 Nuxt 就没法纠正"请求 URL 和服务端路由真正不匹配"的情况了。

## 自查

- 三项都在 `app` 节：`baseURL`（挂载前缀）、`buildAssetsDir`（资源目录）、`cdnURL`（public 绝对地址，仅生产）。
- 资源最终地址 = `baseURL`（或 `cdnURL`）拼 `buildAssetsDir`。
- `baseURL` 在 `nuxt.config.ts` 里**不能**写相对路径 `./`；要么 `NUXT_APP_BASE_URL=./` 构建时设，要么塞进 `runtimeConfig.app.baseURL`。
- `buildAssetsDir` 是构建期定的，运行期别改；`baseURL`/`cdnURL` 有对应的 `NUXT_APP_*` 环境变量能运行时覆盖。
- 子路径部署：设 `app.baseURL` 或 `NUXT_APP_BASE_URL` 即可。

## 下一篇

下一篇 362 讲配置生效时机：改哪些文件 Nuxt 会整个重启（`nuxt.config`、`.env`、`.nuxtrc`、`.nuxtignore`），哪些值能热更（`app.config`），哪些不能（`runtimeConfig`）。

## 本文依据

- `app` / `baseURL` / `buildAssetsDir` / `cdnURL` 的类型、默认值、示例与覆盖变量，及相对路径限制的 `::note`（`api/nuxt-config`）。原文："baseURL — The base path of your Nuxt application." "This can also be set at runtime by setting the NUXT_APP_BASE_URL environment variable." "buildAssetsDir — The folder name for the built site assets, relative to baseURL (or cdnURL if set). This is set at build time and should not be customized at runtime." "cdnURL — An absolute URL to serve the public folder from (production-only)." "Relative paths (like `./`) are not supported directly in nuxt.config.ts due to Nitro limitations."
- 子路径部署与多路径代理的 payload 处理（`getting-started/deployment`「Serving the Same Build at Multiple Paths」）。原文："For a normal subpath deployment, set app.baseURL or the NUXT_APP_BASE_URL environment variable."
