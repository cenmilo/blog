---
url: /posts/029b812bd8156a750d07f7ce58ff751b/
title: 目录类配置串讲：srcDir、serverDir、dir 与 buildDir
date: 2027-02-07T11:00:00+08:00
updated: 2027-02-07T11:00:00+08:00
author: cmdragon

summary:
  摘要：nuxt.config 里有几个决定"代码放哪、产物放哪"的目录类配置，放一起讲最清楚。rootDir 是含 nuxt.config 的目录、多数路径以它为基准。srcDir 是应用源码目录（Nuxt 4 默认 app，3 兼容模式默认 '.'），基准 rootDir。serverDir 是服务端目录（默认 <rootDir>/server），注意它基准也是 rootDir 而不是 srcDir。dir 是个对象，细分 app/assets/pages/layouts/middleware/plugins/modules/public/shared 九项（多数默认带 app/ 前缀，但 server 不在 dir 里）。buildDir 是生成的 `.nuxt` 所在（默认 <rootDir>/.nuxt）。这一篇把默认值、基准和"改名的连带影响"对齐。

categories:
  - 前端开发

tags:
  - nuxt4
  - srcDir
  - serverDir
  - dir
  - buildDir
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

079/086/087 那几篇在讲目录结构时零散提过这些配置项。这一篇把 nuxt.config 里的目录类配置集中对齐——它们的默认值、以谁为基准、改了会牵动什么。

## 先摆基准：rootDir

一切相对路径的锚。官方口径（configuration/directory-structure）：`rootDir` 是**含 nuxt.config.ts 的那个目录**。下面几个配置里凡说"相对"，除特别注明外都是相对 rootDir。

## srcDir：应用源码目录

nuxt-config 的 `## srcDir`：

> Define the source directory of your Nuxt application. If a relative path is specified, it will be relative to the `rootDir`.
> - Default: `"app"` (Nuxt 4), `"."` (Nuxt 3 with `compatibilityMode: 3`)

要点：

- Nuxt 4 默认 `app`——所以 app.vue、pages/、components/ 都在 `app/` 下。
- Nuxt 3 兼容模式下默认 `.`（根）。
- 相对路径基准是 **rootDir**。

## serverDir：服务端目录

`## serverDir`：

> Define the server directory of your Nuxt application, where Nitro routes, middleware and plugins are kept. If a relative path is specified, it will be relative to your `rootDir`.
> - Default: `"/<rootDir>/server"`

一个易错点：**serverDir 的基准是 rootDir，不是 srcDir**。这和 `watch` 配置基准是 srcDir 形成对比。而且 server 这个目录**不在下面的 `dir` 里**——它有自己的 `serverDir` 键。

## dir：细分目录对象

`## dir` 是"定制默认目录结构"，官方一句 "It is better to stick with defaults unless needed."。它有九个子项，默认值分两类：

- **带 `app/` 前缀**（因为默认 srcDir 就是 app）：`app`（默认 `app`）、`assets`（`app/assets`）、`pages`、`layouts`、`middleware`、`plugins`。
- **不带 `app/`**：`modules`、`public`（`public`）、`shared`（`shared`）。

记一句：**`dir` 里没有 server**——server 走 serverDir。改 `dir.pages` 这类会动路由/自动导入扫描的位置，除非特殊结构一般别动。

## buildDir：生成目录

`## buildDir`：

> Define the directory where your built Nuxt files will be placed. Many tools assume that `.nuxt` is a hidden directory (because it starts with a `.`). If that is a problem, you can use this option to prevent that.
> - Default: `"/<rootDir>/.nuxt"`

`.nuxt` 里是 Nuxt 生成的中间产物与类型（不能手改、dev 会重建，052）。改名要连带 `.gitignore`、`ignore`（默认含 `.nuxt`）等，`nuxt.lock` 位置也会变。

顺带一条：`typesDir`（358）默认跟随 `buildDir`，但官方刻意把它分开——只设 buildDir 时类型仍留在 `.nuxt`。

## 改名的连带影响

| 改的键 | 会牵动 |
| --- | --- |
| srcDir | 所有默认带 `app/` 的 dir 子项、别名 `~`/`@` 的落点 |
| serverDir | `#server` 别名、`/api` 前缀来源、server/types 扫描 |
| dir.* | 对应目录的自动导入/路由扫描位置 |
| buildDir | `.gitignore`、`ignore` 默认项、`nuxt.lock`、`analyze` 目录 |

官方对这些的态度统一是"保持默认、除非必要"。

## 自查

- 记基准：rootDir 是含 nuxt.config 的目录；srcDir、serverDir、buildDir 的相对路径都相对 rootDir；serverDir 基准不是 srcDir。
- 记默认：srcDir=`app`（Nuxt4）、serverDir=`<rootDir>/server`、buildDir=`<rootDir>/.nuxt`；dir 九项里 server 缺席。
- 别乱改这些——牵动别名、自动导入扫描、gitignore 一串；官方也说"除非必要保持默认"。

## 下一篇

下一篇 357 讲 alias 与 extensions：模块解析的两把钥匙——别名默认表、webpack 上下文必须加 `~`，以及默认解析的六种扩展名。

## 本文依据

- nuxt-config srcDir / serverDir / dir / buildDir 节（`api/nuxt-config`）：srcDir（"relative to the rootDir"，默认 `"app"`(Nuxt4)/`"."`(Nuxt3 compat)）；serverDir（"where Nitro routes, middleware and plugins are kept"、"relative to your rootDir"、默认 `<rootDir>/server`）；dir（"It is better to stick with defaults unless needed."、子项 app=app/assets=app/assets 等九项）；buildDir（默认 `<rootDir>/.nuxt`、"Many tools assume that .nuxt is a hidden directory ... you can use this option to prevent that"）；typesDir 默认随 buildDir（见 358）。
- 目录结构基准与改名连带（见 079/086/087/051/052）。
