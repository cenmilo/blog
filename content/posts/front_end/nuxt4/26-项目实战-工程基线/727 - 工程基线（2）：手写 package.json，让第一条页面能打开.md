---
url: /posts/68db802def36940278cb75034a559d61/
title: 727 - 工程基线（2）：手写 package.json，让第一条页面能打开
date: 2026-10-18T10:00:00+08:00
updated: 2026-10-18T10:00:00+08:00
author: cmdragon

summary:
  摘要：不用脚手架，把一个 Nuxt 4 项目的最小形状手写出来：一份 package.json、一份 nuxt.config.ts、一个 app/app.vue。这一篇给的不是"能跑"两个字，而是装完之后目录里到底多了什么、构建产物有多大、生成出来的 HTML 里那句话在不在——所有数字都来自同一次实跑。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - package.json
  - 构建产物
  - 最小工程
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 这一步只造三个文件

终端里现在是一个空目录。把 Nuxt 4 项目跑起来需要的东西比想象中少：三个文件，一条安装命令。之所以要手写而不是先敲 `npm create nuxt`，是因为脚手架会顺手生成十几个文件，你不会知道哪几个是必需的、哪一个是为了触发某件事才存在的；写完之后再加东西时，每一步都能对应到"是谁让它出现的"。

前置只有一条硬要求：官方讲安装那一节写的是 **Node.js 22.x 或更新**，并且建议用偶数版本（22、24），因为奇数版本不进长期支持线。先看自己机器：

```bash
node -v
npm -v
```

## 官方给的最小 package.json

讲 `package.json` 那一节直接给了这份形状，逐字照抄：

```json [package.json]
{
  "name": "min-scratch",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "nuxt build",
    "dev": "nuxt dev",
    "generate": "nuxt generate",
    "preview": "nuxt preview",
    "postinstall": "nuxt prepare"
  },
  "dependencies": {
    "nuxt": "latest",
    "vue": "latest",
    "vue-router": "latest"
  }
}
```

四个字段各有各的责任，缺一个都会在后面某个时刻冒出来：

| 字段 | 为什么必须有 |
| --- | --- |
| `"private": true` | 这是应用不是库。npm 对"未标 private 又没写 publishConfig"的包会拒绝发布，这条是防误发 |
| `"type": "module"` | 整个项目按 ESM 解析。Nuxt 4 的配置与产物都是 ESM，缺了它 `.ts` 之外的 `.js` 会被按 CommonJS 处理 |
| `"postinstall": "nuxt prepare"` | 每次装完依赖就生成 `.nuxt/`（类型、自动导入清单、别名）。少了它，第一次 `npm run typecheck` 会在一片"找不到名称"里查半天 |
| `"dependencies"` 里那三项 | `nuxt` 是本体；`vue` 与 `vue-router` 写在这里，是为了让版本由你锁定，而不是各自被 `nuxt` 的依赖树拽进来一个 |

## 装完之后目录里有什么

```bash
npm install nuxt vue vue-router
```

这台机器上跑完的数（`npm` 有脚本白名单策略，见下面排查表第 3 行）：

```text
audited 588 packages
node_modules：172,376,447 字节 / 14,080 个文件，顶层目录 429 个
```

**五百八十八个包、一百七十多兆**，只为了一个页面。这不是配置错了，是构建工具链的真实重量：`nuxt` 底下挂着 Vite、Rollup、Nitro、unhead、Vue 编译器、TypeScript 类型包等一整套。它提醒你两件事：把 `node_modules` 排除在版本控制与备份之外；以及第三篇讲生产体积时，要区分"开发装了多少"与"产物带了多少"——后者小两个数量级。

安装过程顺手执行了 `postinstall`，于是多了 `.nuxt/`：

```text
.nuxt 条目数：16   （含 imports.d.ts、components.d.ts、tsconfig 相关文件、dist/ 缓存目录等）
```

这个目录是**生成物**，不要提交、不要手改，删了下次 `nuxt prepare` 会重建。它这一次列出来是 16 项：

```text
.nuxt/  app.config.mjs  components.d.ts  fetch.d.ts  imports.d.ts  manifest
        nuxt.d.ts  nuxt.node.d.ts  nuxt.shared.d.ts  schema  types
        tsconfig.app.json  tsconfig.json  tsconfig.node.json
        tsconfig.server.json  tsconfig.shared.json
```

四个 tsconfig 对应四类代码（app 侧、node 侧、server 侧、shared），自动导入的清单在 `imports.d.ts` 与 `components.d.ts` 里——编辑器能提示 `useRuntimeConfig` 这种"没 import 就能用"的名字，靠的就是这两份。

同一趟还落下了 `package-lock.json`：**323,587 字节**。它记录的是这 588 个包各自的精确版本与来源，`^` 范围下次解析可能拿到不同的补丁号，而按 lockfile 装不会。

`vue-router` 这一项为什么要自己写？装完直接问它：

```bash
npm ls vue-router
```

```text
min-scratch@ C:\...\_scratch-min
├─┬ nuxt@4.5.2
│ └── vue-router@5.4.0 deduped
└── vue-router@5.4.0
```

`deduped` 那个词是答案：因为你在 `dependencies` 里写了它，npm 把这一份提到顶层，`nuxt` 内部依赖的同一个版本**复用同一份**而不是再装一遍。自己写它的收益是版本由你在这份文件里看得见、能锁；不写它会跟着 `nuxt` 的依赖树走，出问题时报的版本对不上你写的那份。

## 页面与配置：另两个文件

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  compatibilityDate: '2025-07-15',
})
```

```vue [app/app.vue]
<script setup lang="ts">
// 最小页面：只为了看清"哪几个文件是必需的"。
</script>

<template>
  <main>
    <h1>最小页面能打开</h1>
    <p data-testid="scratch-marker">这一段要出现在生成出来的 HTML 里</p>
  </main>
</template>
```

两件容易写歪的地方：

- **`app/` 这一层不能省。** Nuxt 4 把源码目录定成 `app/`，页面、组件、布局都在它下面。写在根下的 `pages/` 里，框架扫不到，表现是"路由存在但页面空白"或 404，而且不报错。
- **`defineNuxtConfig` 不用 import。** 这个文件由框架按约定注入，写了 import 反而会重复声明。它是配置的唯一入口，4.x 起 `vite.config.ts`、`nitro.config.ts` 这类独立文件不再被读取，相关内容要写进这份配置的对应小节。

## 让它"打开"：两种不需要服务器的验法

日常开发是 `npm run dev`，浏览器开 `http://localhost:3000`。这一步要在没有终端常驻进程的情况下确认"真的渲出东西了"，有两条路，各跑一次：

```bash
npm run build      # 打成 Node 服务产物
npm run generate   # 直接输出静态 HTML
```

这台机器上的输出：

```text
build    → Σ Total size: 1.96 MB (490 kB gzip)
           .output/server/index.mjs 与 chunks/ 等，最后打印
           "[nitro] ✔ You can preview this build using node .output/server/index.mjs"
generate → ℹ Prerendering 3 initial routes with crawler
           ℹ Prerendered 4 routes in 0.621 seconds
           ✔ Generated public .output/public
```

两条的产物形状不一样，这一点经常被混着讲：`build` 出的是**服务端**（`.output/server/`，页面在请求时渲染）；`generate` 出的是**纯静态**（`.output/public/`，页面已经写成 HTML 文件）。这一趟 `generate` 跑完之后，`.output/` 里只剩 public：

```text
.output/public：156,917 字节 / 13 个文件（其中 _nuxt/ 里 8 个）
.output/server：不存在
.output/public/index.html：1,441 字节
   含 <h1>最小页面能打开</h1>      → 是
   含那句 data-testid 的正文       → 是
   <script> 标签数                 → 4
```

"页面能打开"这件事在这里被换成一个不需要浏览器的判据：**生成的 HTML 里那句话在不在**。在，说明路由解析、组件编译、服务端渲染三步都通了；不在，报错会直接落在前两步之一。

顺带对个照：这一篇的最小工程有 5 个 script 字段，而整个系列做完之后仓库里的 `package.json` 有 10 个——多出来的每个字段都是某一步为了一条能重复执行的命令加的（lint、typecheck、三层测试、数据库迁移）。这一步不用管它们。

## 卡住的时候查这几处

| 现象 | 先查哪里 |
| --- | --- |
| 装完依赖页面 404、终端不报错 | 页面是不是放在 `app/` 下面；`app/pages/index.vue` 与 `app/app.vue` 同时存在时以 `app.vue` 为外壳 |
| 编辑器一堆"找不到模块"红线 | 跑一次 `nuxt prepare`（或重装依赖触发它），`.nuxt/` 没生成类型就没有提示 |
| `npm install` 出现 `allow-scripts` 警告 | 这台机器的 npm 策略要求逐个批准安装脚本（本次 `esbuild@0.28.2` 的 postinstall 就没自动跑）。构建照样通过；如果你的机器上构建报 esbuild 相关错误，先按提示批准那个脚本，别去改依赖版本 |
| `npm run dev` 打开后热更新很慢（Windows） | 官方在安装那节建议把地址从 `localhost:3000` 换成 `127.0.0.1:3000`，Windows 上 DNS 解析会省一截 |
| 依赖版本号上带 `^` | 文档那份最小 `package.json` 写的是 `latest`，装完会被替换成 `^4.5.2` 这种范围。锁版本是下一篇的事，这里先别手改 |
| 想提交 `.nuxt/` 或 `node_modules/` | 都不要提交；前者是生成物，后者由 lockfile 复现 |

下一步 [728《工程基线（3）：第一次构建，dev 与 build 的产物差在哪》](/posts/50bb6a428564997ce0f67d6345a6d056/)：把 `.output` 拆开看，并回答"dev 能跑为什么不算跑通"。

## 本文依据

- 最小 `package.json` 的字段与形状（四个 scripts 与 `postinstall` 那一份原文）：`https://nuxt.com/docs/4.x/directory-structure/package`
- Node.js 版本要求、偶数版本建议、`npm create nuxt@latest`、Windows 下用 `127.0.0.1` 的建议：`https://nuxt.com/docs/4.x/getting-started/installation`
- `app/` 这一层与 `app.vue` 的角色（页面、组件、布局都住在它下面）：`https://nuxt.com/docs/4.x/directory-structure/app/app` 与目录总览 `https://nuxt.com/docs/4.x/directory-structure`
- 构建与静态生成的命令语义：`https://nuxt.com/docs/4.x/api/commands/build`、`https://nuxt.com/docs/4.x/getting-started/deployment`
- 本轮所有数字来自同一台机器上的最小工程实跑（Node v24.11.1、npm 11.17.0），路径是仓库根下的临时目录 `_scratch-min/`，与正式项目分开
