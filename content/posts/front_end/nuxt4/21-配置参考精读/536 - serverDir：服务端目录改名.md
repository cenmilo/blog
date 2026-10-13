---
url: /posts/b4f2d18c9f1f48715dab202a88c1231e/
title: 536 - serverDir：服务端目录改名
date: 2027-04-08T11:00:00+08:00
updated: 2027-04-08T11:00:00+08:00
author: cmdragon

summary:
  摘要：`serverDir` 定义服务端代码放哪，官方一句说清了里面装什么：**"Define the server directory of your Nuxt application, where Nitro routes, middleware and plugins are kept."**——路由、中间件、插件三类。相对路径按 `rootDir` 算，默认 `"/<rootDir>/server"`。值得注意的是它**不在 `dir` 那九个子项里**（`dir` 管的是 `app`、`assets`、`layouts`、`middleware`、`modules`、`pages`、`plugins`、`public`、`shared`），服务端目录被单独立成一个顶层键；也不要和 `server` 那一节混——`server.builder` 管的是用哪个服务端构建器，不是目录。改这个目录会连带影响的是 URL 前缀的落点：`api` 前缀规则是按目录位置生效的。

categories:
  - 前端开发

tags:
  - nuxt4
  - 目录结构
  - serverDir
  - 服务端路由
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`server/` 里三个子目录各司其职：`api`、`middleware`、`plugins`。想把整个目录改名，你先得知道该改哪个键——因为文档里有两个看起来都像答案的地方。

## serverDir 那一节的全部

`## serverDir` 只有四行信息：

> Define the server directory of your Nuxt application, where Nitro routes, middleware and plugins are kept.
>
> If a relative path is specified, it will be relative to your `rootDir`.

- **类型**：`string`
- **默认**：`"/<rootDir>/server"`

**"where Nitro routes, middleware and plugins are kept"** 这句是这节的核心：它同时说明归属（Nitro 的东西）与内容（三类）。也就是说 `serverDir` 管的是整个服务端目录，里面那三类的相对位置由目录约定决定，不在这个键里细分。

默认值又是那套占位写法：`<rootDir>` 取解析出来的应用根，所以默认位置就是**根下的 `server`**。这一点与 `srcDir` 不同——**服务端目录不跟着源码根进 `app/`**，官方那棵 15 行的目录树里 `server/` 就与 `app/` 同级。

## 为什么它不在 dir 里

配置参考的 `dir` 一节给的是九个内置目录的改名入口，子项是：`app`（默认 `"app"`）、`assets`（`"app/assets"`）、`layouts`（`"app/layouts"`）、`middleware`（`"app/middleware"`）、`modules`（`"modules"`）、`pages`（`"app/pages"`）、`plugins`（`"app/plugins"`）、`public`（`"public"`）、`shared`（`"shared"`）。

**这九项里没有 `server`。** 服务端目录是独立的顶层键 `serverDir`，因此：

| 你想改 | 该用的键 |
| --- | --- |
| 整个服务端代码目录 | `serverDir` |
| 页面目录 | `dir.pages` |
| 客户端插件目录 | `dir.plugins` |
| 客户端中间件目录 | `dir.middleware` |

一处容易踩的命名冲突：`dir.middleware` 与 `dir.plugins` 指的是 **`app/` 下的客户端中间件与插件**，而 `serverDir` 里同名的 `server/middleware`、`server/plugins` 是服务端那两类——**两组键管的是不同侧的同名目录**。官方在两处都没写这层对照，读到 `plugins` 时要先确认自己在说哪一侧。

## 别和 server 那一节混

顶层还有一个 `## server` 节，但它管的是构建器不是目录：**"Configuration for Nuxt's server builder."**，其下只有一个子项 `builder`（`string | { bundle: (nuxt: Nuxt) => Promise<void> }`，默认 `"@nuxt/nitro-server"`）。

那一节还带一条 `::callout` 警告：**"This option is intended for internal use and the API is not finalized. Please open an issue before relying on the current implementation."** ——**`server.builder` 是内部用途、API 未定**，这一看就不是给"改目录"用的。所以 `server` 与 `serverDir` 是两个不同层的键，一个选引擎一个选路径。

## 改名会连带什么

`serverDir` 只改位置，不改规则。服务端目录里的 URL 规则来自目录约定本身：

- `server/api` 下的文件**自动带 `/api` 前缀**
- 不要前缀就放 `server/routes`
- 官方示例给的可达性是 `server/routes/hello.ts` 对应 `http://localhost:3000/hello`

所以把 `serverDir` 改成 `"nuxt-server"` 之后，原来说法不变、只是文件搬了：**`nuxt-server/api/hello.ts` 仍然对 `/api/hello`，`nuxt-server/routes/hello.ts` 仍然对 `/hello`。** 前缀是按"该目录下的 `api`/`routes` 子目录"算的，与 `serverDir` 叫什么无关。

另外有两条与目录改名无关但常被混进来的项：`serverHandlers`（Nitro 服务端 handlers，在配置里直接注册）与 `nitro` 节（Nitro 自己的配置）。**这两处都不改变目录位置**，只是往服务端里加东西。

**文档没有说明改 `serverDir` 时那些已经写死的相对 import 路径会怎样**（例如 `server/utils` 里的 `~~/shared/...`）。改之前建议先把目录改名与一次完整构建放在一起验证，看服务端路由是否仍按预期注册。

## 自查

- `serverDir` 是 `string`、默认 `"/<rootDir>/server"`，官方定义点名里面装 **Nitro 的路由、中间件、插件**；相对路径按 `rootDir` 算。
- 它是独立顶层键，**不在 `dir` 的九个子项里**；`dir` 里没有 server 这一项。
- `dir.middleware`／`dir.plugins` 指客户端那两类，与 `server/` 下同名的两类是不同侧的东西。
- 顶层 `server` 节管的是服务端构建器（只有 `builder`，默认 `@nuxt/nitro-server`），并带"内部用途、API 未定"的警告，与目录无关。
- 改名只换位置不换规则：`api` 子目录仍带 `/api` 前缀，`routes` 子目录仍不加前缀；改后要用一次完整构建验证服务端路由注册正常。

## 下一篇

下一篇 537 把 `dir` 那九个子项一次列全：每个目录的默认值都写着什么、为什么默认值里已经带了 `app/` 前缀、以及官方那句"最好保持默认"出现在哪一行。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## serverDir` 整节：定义句含 Nitro routes/middleware/plugins、相对 rootDir 那句、Type `string`、Default `"/<rootDir>/server"`；`## dir` 九个子项与各自默认值；`## server` 节与 `builder` 子项的类型、默认 `"@nuxt/nitro-server"` 及其 `::callout` 警告；`## serverHandlers`；`## srcDir` 的目录树把 `server/` 与 `app/` 并列）
- https://nuxt.com/docs/4.x/directory-structure/server（Server Routes 的 `/api` 前缀规则、`server/routes` 不加前缀、`server/routes/hello.ts` 与 `http://localhost:3000/hello` 示例、Server Middleware 与 Server Plugins 的自动读取）
