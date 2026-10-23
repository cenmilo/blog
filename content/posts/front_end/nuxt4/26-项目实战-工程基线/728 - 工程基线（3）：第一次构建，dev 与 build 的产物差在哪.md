---
url: /posts/50bb6a428564997ce0f67d6345a6d056/
title: 728 - 工程基线（3）：第一次构建，dev 与 build 的产物差在哪
date: 2026-10-18T11:00:00+08:00
updated: 2026-10-18T11:00:00+08:00
author: cmdragon

summary:
  摘要：同一个最小页面，dev、build、generate 三条命令留下的东西完全不一样。这一篇用一次真实构建把 `.output` 拆开看：服务端目录里有什么、生产依赖为什么只剩十九项、为什么普通构建的 public 里没有 index.html，以及"整个目录会被重建"这条警告怎么自己验一遍。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 构建产物
  - output 目录
  - Nitro
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 两个数量级的差

上一篇装完依赖，`node_modules` 是 **172,376,447 字节、14,080 个文件**。跑一次构建之后，终端最后给出的是：

```text
[nitro] ℹ Building Nuxt Nitro server (preset: node-server, compatibility date: 2025-07-15)
[nitro] ✔ Nuxt Nitro server built
  ├─ .output/server/index.mjs (1.67 kB) (737 B gzip)
Σ Total size: 1.96 MB (491 kB gzip)
[nitro] ✔ You can preview this build using node .output/server/index.mjs
```

**1.96 MB 对 172 MB。** 差出去的两个数量级就是"开发时要的东西"与"跑起来要的东西"的差：前者包含编译器、类型工具、源映射、构建框架本身；后者只包含运行时真需要的东西。这个数不是安慰剂，它决定了后面能不能判断"产物里混进了开发依赖"这类问题。

这一步要把三件事分清楚：`dev` 留下的是内存与缓存目录里的东西、`build` 产出的是**服务端**、`generate` 产出的是**静态文件**。它们经常被叫成"部署方式"，但形状差别会直接改变你要验什么。

## 四条命令各自留下什么

| 命令 | 留下什么 | 谁去跑它 |
| --- | --- | --- |
| `nuxt dev` | `.nuxt/` 与 `node_modules/.cache/nuxt/`，**不产 `.output`**；带 HMR 的开发服务，默认 `http://localhost:3000` | 你，写代码时 |
| `nuxt build` | `.output/server/`（Node 服务，页面在请求时渲染）＋ `.output/public/_nuxt/`（客户端资源） | CI 与部署 |
| `nuxt generate` | `.output/public/`（页面已经写成 HTML 文件），**`.output/server/` 不在** | 静态托管 |
| `nuxt preview` | 不产新东西，起一个服务跑上一次 build 的结果 | 本地想验产物时 |

同一台机器上的实际形状（`build` 之后）：

```text
.output/server  → 1,958,344 字节 / 122 个文件
.output/public  → 153,181 字节 / 9 个文件
.output/public 顶层只有 ["_nuxt"]
.output/public/index.html → 不存在
.output/server/index.mjs  → 1,671 字节
```

**普通构建的 public 里没有 `index.html`**——这是最容易踩的一条：`build` 只交客户端资源，HTML 由服务端在每次请求时渲。想让 HTML 提前变成文件，要么 `nuxt generate`，要么 `nuxt build --prerender`（讲命令那一节明写 `--prerender` 会把 target 强制设成 `static`）。上一篇末尾那次 `generate` 之后，`index.html` 就有了，而且 `.output/server` 反过来不见了：同一条命令链，两种交付形状。

## 产物里也有一份 node_modules，但只有十二个包

第一反应会是"服务端产物怎么带 `node_modules`"——它确实带，`.output/server/` 顶层是这四样加一个目录：

```text
.output/server/  chunks/  index.mjs  index.mjs.map  node_modules/  package.json
.output/server/node_modules → 顶层 12 个目录 / 95 个文件 / 1,698,849 字节
                               （@babel、@vue、devalue、entities、estree-walker、
                                 hookable、nostics、source-map-js、ufo、unhead、vue、vue-bundle-renderer）
```

对照上一篇那台开发机上的 `node_modules`：**429 个顶层目录、14,080 个文件、172,376,447 字节**。产物这份只有它的百分之一。它的内容由产物自己那份 `package.json` 决定，一共 **19 项**，全是被服务端运行时真引用的（Vue 的渲染链、序列化用的 `devalue`、head 管理 `unhead`、URL 工具 `ufo`）：

```json [.output/server/package.json（节选）]
{
  "type": "module",
  "dependencies": {
    "@vue/compiler-core": "3.5.44",
    "@vue/server-renderer": "3.5.44",
    "devalue": "5.9.4",
    "hookable": "6.1.2",
    "ufo": "1.6.4",
    "unhead": "3.4.3",
    "vue": "3.5.44",
    "vue-bundle-renderer": "2.4.0"
  }
}
```

注意版本号是精确的，没有 `^`：产物锁的是构建时那一份。这条对部署方式的含义很直接——把 `.output` 整个拷到一台只装了 Node 的机器上，`node .output/server/index.mjs` 就能起，因为它把要用的那十二个包带在身上了。这也是后面发布那一步要求"产物独立运行"的原因；反过来，如果你在这份清单里看到 `vite`、`@nuxt/schema` 这类名字，就说明有开发依赖被运行时引用了，要回源码里查是谁 import 的。

顺带认一下文件名：`.output/public/_nuxt/` 里是 `DMD6Pc9a.js`、`error-404.Bb87HomL.css` 这种带 hash 的名字，共 8 项（5 个 js、2 个 css、1 个 `builds/`）。**hash 在文件名里，意味着内容一变名字就变**，可以放心给长缓存；而 `public/` 目录里你手工放的文件既不 hash 也不带缓存头——这个差别在讲静态资源那一站会量出来。

## "别手改 .output" 是一条可以自己验的警告

讲 `.output` 那一节有两句：这个目录要进 `.gitignore`；以及**你不该动里面的任何文件，因为 `nuxt build` 会重建整个目录**。第二句很容易当成客套话，验一次只要半分钟：

```bash
echo anything > .output/MARKER.txt     # 手工往产物里塞一个文件
npm run build
ls .output                             # MARKER.txt 已经不在了
```

这台机器上的结果：

```text
写 MARKER 后存在: true
重新构建后还在吗: false
```

所以任何"改一下产物让它先跑起来"的动作，寿命都是一次构建。真有要改的东西，改源文件、改配置，让构建把它带进产物——这条纪律在后面每次遇到"产物里怎么少了某个文件"时都会救你一次。

## dev 与 build 的差别藏在哪

`nuxt dev` 期间你看到的页面与构建后的产物不是同一份东西，三个证据：

- **落点不同。** 构建的中间产物在 `node_modules/.cache/nuxt/.nuxt/dist/` 这一带（`generate` 的日志里会直接打出这些路径），而 dev 用的 `.nuxt/` 只是类型与自动导入清单。也就是说：**dev 一切正常，不代表 build 能过**——反过来也成立得少一些，凡是只有 dev 看得到的现象，都要在产物上再确认一次。
- 端口与环境的解析链不同。开发服务的端口按 `NUXT_PORT || NITRO_PORT || PORT || devServer.port` 解析，`nuxt preview` 走同一条链；但**直接用 `node .output/server/index.mjs` 起进程时，认的是 `PORT`**——`NUXT_PORT` 对这一种起法不生效。本系列后面跑端到端测试时就是第三种起法，端口必须显式给 `PORT`。
- **配置分层的默认档不同。** 讲 `build` 命令那一节写着：`--envName` 在构建时默认 `production`、在 dev 时默认 `development`。所以同一份 `nuxt.config.ts` 里写给不同环境的覆盖块，dev 与 build 读到的不是同一份——第五篇讲配置分层时正面处理它。

顺带一条日志噪音：这台机器的构建日志里出现过 `[DEP0155] DeprecationWarning`（来自 `@vue/shared` 的 exports 尾部斜杠映射）。它不是你配置错，也不会让构建失败；看到时先确认构建结尾是不是 `✨ Build complete!`。

## 这一步的自查清单

1. `npm run build` 结尾有 `✨ Build complete!`，且日志里没有 `ERROR`。
2. `.output/server/index.mjs` 存在（1,671 字节量级），`.output/public/_nuxt/` 有若干带 hash 的文件。
3. `.output/public/index.html` 不存在（普通构建）——如果你要它存在，用 `nuxt generate` 或 `--prerender`。
4. `.output/server/package.json` 的 dependencies 数量级对得上（这里 19 项）；如果它涨到几百项，说明有开发依赖被运行时引用了。
5. 把 `.output` 加进 `.gitignore`；已经在版本库里的话要移出索引。
6. 想本地看产物：`npm run preview`，或 `node .output/server/index.mjs` 并显式给 `PORT`。

| 现象 | 先查哪里 |
| --- | --- |
| dev 打开正常，build 报缺模块 | 只在 dev 用过的路径别名；`.nuxt/` 缺了先 `nuxt prepare` |
| 产物起来了但页面空白 | 浏览器控制台看资源 404：`baseURL`/静态资源目录不对，或服务没有把 `_nuxt/` 交出去 |
| `generate` 之后 CSS 或图片 404 | 页面是预渲染的那份，资源路径由配置里的 `app.buildAssetsDir`/CDN 前缀决定 |
| 改了 `.output` 里的文件没生效 | 重建会清掉，见上一节那个实验 |
| 起产物服务时端口不是你要的那个 | 直接 `node` 起时只读 `PORT`（`nuxt preview` 才读那串链） |

下一步 [729《工程基线（4）：Node 版本与包管理器：engines、lockfile 与一次真实重装》](/posts/a84cfdb58daaff6ae84f184fd57a8e46/)：把版本这件事钉住——`engines` 到底拦不拦人、`packageManager` 归谁读、lockfile 在一次真实重装里做了什么。

## 本文依据

- `.output` 的用途、要进 `.gitignore`、整目录会在 `nuxt build` 时重建：`https://nuxt.com/docs/4.x/directory-structure/output`
- `nuxt build` 的选项表（`--prerender` 强制 target 为 `static`、`--target`/`--preset`、`--envName` 默认值、`--profile`）：`https://nuxt.com/docs/4.x/api/commands/build`
- `nuxt dev` 的端口解析链、TUI 与默认 3000：`https://nuxt.com/docs/4.x/api/commands/dev`
- `nuxt preview` 跑上一次构建的结果、端口链与 `node_modules/.cache/nuxt/preview` 记录：`https://nuxt.com/docs/4.x/api/commands/preview`
- 静态生成与预渲染的差别：`https://nuxt.com/docs/4.x/getting-started/deployment`
- 本篇所有数字来自仓库根下临时最小工程 `_scratch-min/` 的同一次跑（Node v24.11.1、npm 11.17.0），与正式项目分开
