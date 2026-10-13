---
url: /posts/5421245ddda9cc9312731901891c83f3/
title: 670 - 构建期钩子九到二十：app 生成、构建两段、manifest 与页面扫描
date: 2027-05-23T10:00:00+08:00
updated: 2027-05-23T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：这十二行覆盖四个族。**`app:` 三行是模板的递进**：`app:resolve`（解析完 `app` 实例之后）、`app:templates`（**在生成 `NuxtApp` 期间，用来自定义、修改或往构建目录加新文件——虚拟的或写到 `.nuxt` 的都行**）、`app:templatesGenerated`（**模板被编译进虚拟文件系统之后**）。**`build:` 两行只是"打包器前后"**（`build:before`／`build:done`，都无参数），而 **`build:manifest` 是唯一说明"下游谁在用"的一行：manifest 是 Nitro 用来渲染最终 HTML 里 `<script>` 与 `<link>` 标签的那份**。`builder:` 两行里，**`builder:watch` 只在开发期的构建过程中触发**。`pages:` 三行分两段扫描，其中 **`pages:routerOptions` 给出了一张结构最完整的参数（`{ files: Array<{ path, optional? }> }`）和一条覆盖规则：数组里靠后的覆盖靠前的**。最后 `server:routes` 的用途写得很具体：**Nuxt 需要服务端要提供的全部路由处理器，好让 `$fetch` 与 `useFetch` 按它们来推断类型**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 构建期钩子
  - 模板生成
  - 路由扫描
  - manifest
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

往项目里注入一个模板、想改路由清单、或者想知道生成的 HTML 里那些标签是谁决定的——答案都在这一组十二行里。

## app 一族：模板的三个时间点

| 钩子 | 参数 | 描述 |
| --- | --- | --- |
| `app:resolve` | `app` | 在解析 `app` 实例之后调用 |
| `app:templates` | `app` | 在生成 `NuxtApp` 期间调用，用来自定义、修改或往构建目录添加新文件（**虚拟的，或写到 `.nuxt`**） |
| `app:templatesGenerated` | `app` | **在模板被编译进虚拟文件系统（vfs）之后调用** |

三行的参数都是 `app`，**描述却各自落在不同阶段**，可以排成一条链：**先解析实例，再在生成期间改文件清单，最后模板进 vfs 之后**。

`app:templates` 是这三行里唯一写明"能做什么"的：**自定义、修改、添加新文件到构建目录**，而且**两种形态都支持——只存在于虚拟文件系统，或者真的写到 `.nuxt` 磁盘上**。

`app:templatesGenerated` 那句把 vfs 说成一个明确概念，并链到目录结构里讲 `.nuxt` 的那一节。

**这一族与其他页的连接很直接**：`addTemplate` 这类注册动作发生在模块 setup 里，而**"注册之后什么时候被编译进 vfs"就是 `app:templatesGenerated`**。

## build 两行：只说前后

| 钩子 | 参数 | 描述 |
| --- | --- | --- |
| `build:before` | - | 在 Nuxt 打包器之前调用 |
| `build:done` | - | 在 Nuxt 打包器完成之后调用 |

**两行都没有参数**，描述也各只有一句"在……之前／之后"。**"Nuxt bundle builder"具体指什么、这两个点与 `builder:*` 那两行的关系，表里没有展开**。

要"构建完成后做点什么"，这一族里能选的是 `build:done`。

## `build:manifest`：唯一说明下游是谁的一行

描述是三行里最长的一句：

> Called during the manifest build by Vite and webpack. This allows customizing the manifest that Nitro will use to render `<script>` and `<link>` tags in the final HTML.

三个信息：**由 Vite 与 webpack 在构建 manifest 期间调用**、**参数是 `manifest`**、**它能自定义那份 manifest，而 Nitro 会用这份 manifest 去渲染最终 HTML 里的 `<script>` 与 `<link>` 标签**。

这行值得单独记，因为**它是全套表里少数直接告诉你"你改的东西被谁消费"的一行**——静态资源加载顺序、预加载标签这类问题的上游就在这里。

**它由两个 builder 都触发**（Vite 与 webpack），而**表里没说这两条路径的 manifest 形状是否一致**。

## builder 两行：一词之差与只在开发

| 钩子 | 参数 | 描述 |
| --- | --- | --- |
| `builder:generateApp` | `options` | 在生成 app 之前调用 |
| `builder:watch` | `event, path` | **在开发期的构建过程中**，watcher 发现项目里某个文件或目录变化时调用 |

`build:` 与 `builder:` **只差一个 er，表里没有解释两者分工**——从描述能读出的是前者讲"打包器前后"，后者讲"生成 app 之前"和"文件变化"。

**`builder:watch` 是唯一一行把"只在开发期"写进描述的构建期钩子**，也是被用得最多的一行：**Kit 的 `updateTemplates` 官方示例就是挂在它上面**（`builder:watch` 里判断路径再重算模板），而那条示例还会**跳过 `event === 'change'`**。

参数 `event, path` 两个依次传：**`event` 是变化类型、`path` 是相对路径**（示例里第二个参数写作 `relativePath`，并用 `resolve(nuxt.options.srcDir, relativePath)` 还原成绝对路径）——**"这里的 path 是相对路径"这件事，是靠示例而不是靠这张表说明的**。

## pages 三行：两段扫描与一条覆盖规则

| 钩子 | 参数 | 描述 |
| --- | --- | --- |
| `pages:extend` | `pages` | **在页面路由从文件系统被扫描出来之后**调用 |
| `pages:resolved` | `pages` | **在页面路由被补充了扫描到的元数据之后**调用 |
| `pages:routerOptions` | `{ files: Array<{ path: string, optional?: boolean }> }` | 在解析 `router.options` 文件时调用；**数组里靠后的项覆盖靠前的项** |

前两行的区别只在描述里那个阶段词：**先"扫出来"，后"补上元数据"**——这给出了一个可用判断：**要改路由结构用 `pages:extend`，要读元数据用 `pages:resolved`**。

**`pages:extend` 是运行时那个 `extendPages` 的钩子版本**：Kit 函数那一节写的是"使用 `extendPages` 或使用 `pages:extend` 钩子时都可以增广 `NuxtPageMeta`"，两处等价；**而那条"必须原地修改数组"的硬要求是 Kit 函数那节给的**。

第三行有两个独特点：**参数形状是全表写得最完整的**（一个对象，里面是 `files` 数组，每项含 `path` 与可选 `optional`），**而且它给出了一条明确的覆盖规则：数组靠后的覆盖靠前的**。这条规则在 `addComponentsDir` 的 `prepend`、`addRouteMiddleware` 的 `prepend` 那里是以"插到前面"表达的，**这里是直接用"靠后覆盖靠前"说**——两处方向一致：**排在后面的赢**。

## `server:routes`：为了类型而存在

描述把用途写死了：

> Called when Nuxt needs the full set of route handlers the server will serve, so that it can type `$fetch` and `useFetch` against them.

**参数是 `routes`, `context` 两个**，作用是**当 Nuxt 需要服务端将要提供的全部路由处理器时使用，目的是让 `$fetch` 与 `useFetch` 能按这些路由推断类型**。

这行与两条已知事实连得上：**带类型的 `$fetch` 由 `experimental.routeTypedFetch` 控制**（那是兼容性版本清单里的一项），而**路由集合的来源之一就是 `server:routes`**。

所以它的实际含义是：**服务端路由不只在运行时被注册，还会在类型生成阶段被收集一次**——**你自己动态注册的路由要能被推断出来，就得走这个钩子给的通道**。

## 用法对照

| 你要做的 | 挂哪一行 |
| --- | --- |
| 加一个虚拟文件或往 `.nuxt` 写文件 | `app:templates` |
| 在模板编译完成后读结果 | `app:templatesGenerated` |
| 改最终 HTML 里资源标签的来源 | `build:manifest` |
| 文件变化时重算自己的模板 | `builder:watch`（**只在开发期**） |
| 增删改路由条目 | `pages:extend` |
| 读路由的元数据 | `pages:resolved` |
| 让自定义路由进入 `$fetch` 类型 | `server:routes` |

这一组未记载的还有：`app:templates` 与 `app:templatesGenerated` 之间模板是否已可读取、`build:before`／`build:done` 与 `vite:compiled`／`webpack:compiled` 的关系、`pages:extend` 与 `pages:resolved` 各自被调用几次（多图层时是否每层一次）。

下一篇读二十一到三十二行：自动导入那四组、组件那两组、以及 Nitro 那四组。

## 下一篇

下一篇 671：构建期钩子二十一到三十二：自动导入四组、组件两组与 nitro 四组

## 本文依据

- https://nuxt.com/docs/4.x/api/advanced/hooks
