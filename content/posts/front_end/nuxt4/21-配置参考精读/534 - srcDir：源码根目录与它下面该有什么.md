---
url: /posts/fc5a4b43674a01a9976afe30b11a1aaa/
title: 534 - srcDir：源码根目录与它下面该有什么
date: 2027-04-08T09:00:00+08:00
updated: 2027-04-08T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：`srcDir` 定义应用源码放在哪个目录，官方说明是 "Define the source directory of your Nuxt application."，**相对路径按 `rootDir` 算**。默认值这一行同时给了两个版本：**Nuxt 4 是 `"app"`，Nuxt 3（`compatibilityMode: 3`）是 `"."`**——这就是很多人从旧项目搬过来时目录层级对不上的根源。官方示例 `srcDir: 'app/'` 后面紧跟一份 15 行的目录树，明确划出**哪些东西在 `app/` 下面（`assets`、`components`、`composables`、`layouts`、`middleware`、`pages`、`plugins`、`utils`、`app.config.ts`、`app.vue`、`error.vue`），哪些仍在外面（`server/`、`shared/`、`public/`、`modules/`、`layers/`、`nuxt.config.ts`、`package.json`）**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 目录结构
  - srcDir
  - 配置
  - 迁移
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

页面组件明明在 `pages/` 里，路由却出不来——把它挪进 `app/pages/` 之后立刻好了。

这一类问题的根源是一个配置项的默认值：`srcDir`。

## 定义与两个默认值

配置参考那节全文很短，四条信息：

> Define the source directory of your Nuxt application.
>
> If a relative path is specified, it will be relative to the `rootDir`.

- **类型**：`string`
- **默认**：`"app"`（Nuxt 4），`"."`（`compatibilityMode: 3` 的 Nuxt 3）

**默认值那一行同时给了两种版本**，这是这一节最容易读漏的地方：Nuxt 4 里源码根是 `app` 这个子目录；Nuxt 3 时代它就是项目根（`.`）。所以"我的 `pages/` 该放哪"取决于兼容性设置，而不是靠记忆。

相对路径按 `rootDir` 算——这一点和同族的 `serverDir`、`typesDir`、`analyzeDir` 一样，都写的是"相对 `rootDir`"，而 `rootDir` 本身默认就是项目根。

官方示例：

```ts
export default defineNuxtConfig({
  srcDir: 'app/',
})
```

## 那份 15 行的目录树

紧跟示例的一句是关键：**"This expects the following folder structure"**，然后官方把整棵树铺开：

```bash
-| app/
---| assets/
---| components/
---| composables/
---| layouts/
---| middleware/
---| pages/
---| plugins/
---| utils/
---| app.config.ts
---| app.vue
---| error.vue
-| server/
-| shared/
-| public/
-| modules/
-| layers/
-| nuxt.config.ts
-| package.json
```

这份树的信息量在于**分界线**：缩进在 `app/` 下面的十二项属于源码根，后面七项与 `app/` 同级、**不随 `srcDir` 挪动**。

| 在 `srcDir` 下面 | 在项目根（不跟着挪） |
| --- | --- |
| `assets/`、`components/`、`composables/`、`layouts/`、`middleware/`、`pages/`、`plugins/`、`utils/` | `server/`、`shared/`、`public/`、`modules/`、`layers/` |
| `app.config.ts`、`app.vue`、`error.vue` | `nuxt.config.ts`、`package.json` |

三个具体推论，都能从这张表直接读出来：

- **`server/` 不在 `srcDir` 里**。它由 `serverDir` 单独定义（默认 `"/<rootDir>/server"`），所以改了 `srcDir` 不会把服务端目录一起搬走。
- **`shared/` 与 `public/` 同样在项目根**：`shared` 是应用与服务端共用的那一份，`public` 是原样服务的静态目录，两者都不属于源码根。
- **`app.vue`、`error.vue`、`app.config.ts` 在 `srcDir` 里**——这三个文件在别的语境里常被当成"应用入口"，位置也确实在源码根下面。

## 改了 srcDir 会连带什么

文档在这一节没写"改 `srcDir` 会影响哪些别名"，但相邻那节能对上：`alias` 的默认表里，`~` 与 `@` 指向 `/<rootDir>/app`、`~~` 与 `@@` 指向 `/<rootDir>`、`assets` 指向 `/<rootDir>/app/assets`。也就是说**别名默认值是按"根下的 app"写死的**，动 `srcDir` 时这一批路径是否跟着变，配置参考没有逐条说明——要确认就打印解析后的 `nuxt.options.alias` 看实际值。

另外 `dir` 那一节给了九个内置目录的独立改名入口（`app` 的默认就是 `"app"`、`pages` 是 `"app/pages"`、`plugins` 是 `"app/plugins"` 等），**默认值里已经带上了 `app/` 前缀**。所以"目录层级"这件事在文档里有两处入口：`srcDir` 管源码根，`dir` 管根下每个具体目录。两处同时改会发生什么，官方没有写。

## 自查

- `srcDir` 是 `string`，相对路径按 `rootDir` 算；默认值同时给两种：**Nuxt 4 为 `app`，`compatibilityMode: 3` 的 Nuxt 3 为 `.`**。
- 官方那棵 15 行树划出分界：`app/` 下是八个目录加 `app.config.ts`/`app.vue`/`error.vue`；`server/`、`shared/`、`public/`、`modules/`、`layers/`、`nuxt.config.ts`、`package.json` 留在根。
- 服务端目录由 `serverDir` 单独定义，不随 `srcDir` 移动。
- `alias` 的默认值是按 `<rootDir>/app` 写的，改 `srcDir` 后别名是否跟着变，文档未逐条说明——打印 `nuxt.options.alias` 确认。
- `dir` 一节九个键的默认值本身就带 `app/` 前缀；`srcDir` 与 `dir` 的叠加关系官方没写。

## 下一篇

下一篇 535 读 `rootDir` 与 `workspaceDir`：两个官方都写着"通常不需要配"的目录项，一个会被命令行位置参数覆盖，一个由 Nuxt 自动探测。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## srcDir` 整节：定义句、相对 rootDir 那句、Type `string`、Default 同时给 `"app"`（Nuxt 4）与 `"."`（Nuxt 3 with `compatibilityMode: 3`）、`srcDir: 'app/'` 示例与"This expects the following folder structure"及其 15 行目录树；相邻的 `## serverDir`、`## dir` 九个子项默认值、`## alias` 十行默认表）
- https://nuxt.com/docs/4.x/directory-structure/app/app 与 https://nuxt.com/docs/4.x/directory-structure/server（app 源码目录与服务端目录的归属，用于对照分界）
