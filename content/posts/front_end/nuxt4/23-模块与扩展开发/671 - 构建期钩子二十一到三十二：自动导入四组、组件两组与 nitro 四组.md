---
url: /posts/380c2ff61dd2c013e4a70bc27407c4bb/
title: 构建期钩子二十一到三十二：自动导入四组、组件两组与 nitro 四组
date: 2027-05-23T11:00:00+08:00
updated: 2027-05-23T11:00:00+08:00
author: cmdragon

summary:
  摘要：这十二行是三个族。**自动导入四行**：`imports:sources`（参数 `presets`）、`imports:extend`（参数 `imports`）、`imports:context`（**unimport 上下文被创建时**）、`imports:dirs`（扩展导入目录）——其中**两行的描述都写着"在 setup 期间调用，允许模块扩展"**。**组件两行**里有一条全表少见信息：**`components:dirs` 的描述直接写明"在 `app:resolve` 之内调用"**，这是这张表里唯一把钩子嵌套关系写出来的地方。**Nitro 四行**给的是改服务端的不同位置：`nitro:config`（**初始化 Nitro 之前，可自定义它的配置**）、`nitro:init`（**初始化之后，可注册 Nitro 钩子、直接与 Nitro 交互**）、`nitro:build:before`、`nitro:build:public-assets`（**在公共资源复制之后、Nitro 服务端构建之前修改它们**）。最后一行 `prerender:routes` 是"扩展要预渲染的路由"。

categories:
  - 前端开发

tags:
  - nuxt4
  - 构建期钩子
  - 自动导入
  - 组件注册
  - Nitro配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`addImports`、`addComponent`、`addServerHandler` 这些函数背后都有对应的钩子。这一组十二行就是把"用函数"和"用钩子"接起来的地方，还有一行把钩子的嵌套关系写明了。

## 自动导入四行

| 钩子 | 参数 | 描述 |
| --- | --- | --- |
| `imports:sources` | `presets` | 在 setup 期间调用，允许模块扩展 sources |
| `imports:extend` | `imports` | 在 setup 期间调用，允许模块扩展 imports |
| `imports:context` | `context` | **当 unimport 上下文被创建时调用** |
| `imports:dirs` | `dirs` | 允许扩展导入目录 |

四行对应四种粒度，与 Kit 那三个函数一一能对上：

| 你要扩展的东西 | 钩子 | Kit 函数 |
| --- | --- | --- |
| 预设（整包来源） | `imports:sources` | `addImportsSources` |
| 单条导入 | `imports:extend` | `addImports` |
| 目录 | `imports:dirs` | `addImportsDir` |
| 那个 unimport 上下文本身 | `imports:context` | —— |

**第四行没有对应函数**：它是上下文被创建这个事件，参数是 `context`。**自动导入的底层机制由 unimport 提供**这一点在自动导入那一页已经写明，这里是它唯一出现在钩子表里的位置。

值得注意的是**两行的描述用了"Called at setup allowing modules to extend ..."这种句式**：**它告诉你这些钩子发生在模块 setup 的时段**，所以你的模块如果在更晚的时机（比如某个 `done` 钩子里）去改这些数组，**已经生成完的东西不会回头重算**。

## 组件两行：一条写明嵌套关系

| 钩子 | 参数 | 描述 |
| --- | --- | --- |
| `components:dirs` | `dirs` | **在 `app:resolve` 之内调用**，用来自定义被扫描的、可自动导入组件的目录 |
| `components:extend` | `components` | 允许扩展新的组件 |

**`components:dirs` 是全表里唯一把"我在谁之内"写进描述的一行**：**它发生在 `app:resolve` 里面**。

这一条信息能解开一个常见困惑：**为什么在 `app:resolve` 之后往 `dirs` 里 push 东西不生效**——因为扫描目录的清单在 `app:resolve` 内部就已经问完了。**要改目录，必须挂在 `components:dirs` 上，而不是挂在时间更晚的钩子里改同一份数组**。

第二行 `components:extend` 描述只有一句"允许扩展新的组件"，参数是 `components`——**它是"组件清单成型之后"的位置**，与 `addComponent` 那个函数是两条不同路径（**页面没有说明二者先后或优先级**）。

## Nitro 四行：四个不同阶段

| 钩子 | 参数 | 描述 |
| --- | --- | --- |
| `nitro:config` | `nitroConfig` | **在初始化 Nitro 之前**调用，允许自定义 Nitro 的配置 |
| `nitro:init` | `nitro` | **在 Nitro 被初始化之后**调用，可以注册 Nitro 钩子、直接与 Nitro 交互 |
| `nitro:build:before` | `nitro` | 在构建 Nitro 实例之前调用 |
| `nitro:build:public-assets` | `nitro` | **在公共资源被复制之后调用**；允许在 Nitro 服务端构建之前修改公共资源 |

四行按描述能排出三个阶段：**配置期（`nitro:config`）→ 实例可用（`nitro:init`）→ 构建期（`nitro:build:before`、`nitro:build:public-assets`）**。

**`nitro:config` 是这套文档里出现频率最高的一行**：基础配方里那个"暴露一整个 public 目录"的官方示例正是挂在它上面（`nitroConfig.publicAssets ||= []` 然后 push 一个带 `dir` 与 `maxAge` 的条目）——**这也解释了为什么那个示例里要做 `||= []` 兜底：你拿到的是 Nitro 的配置对象，它可能还没有这个数组**。

**`nitro:init` 的描述给了两件能力：注册 Nitro 钩子、直接与 Nitro 交互**。这与取实例那条形成了对照：

| 想做的事 | 位置 |
| --- | --- |
| 改 Nitro 的配置 | `nitro:config`（初始化之前） |
| 用 Nitro 实例、注册它的钩子 | `nitro:init`（初始化之后） |
| 在别处拿到实例 | `useNitro()`，**但它只能在 `ready` 之后调用** |

**`nitro:build:public-assets` 是唯一一行管静态资源内容的一行**：**复制之后、服务端构建之前**，所以在这里改动能进入最终产物。

## `server:devHandler` 与 `prerender:routes`

| 钩子 | 参数 | 描述 |
| --- | --- | --- |
| `server:devHandler` | `handler` | 当 dev 中间件正被注册到 Nitro 开发服务器上时调用 |
| `prerender:routes` | `ctx` | 允许扩展要被预渲染的路由 |

第一行**只在开发服务器的注册时机上出现**，参数是 `handler`。它对应的是 dev 那一侧的中间件注册（`addDevServerHandler` 那族动作与之相邻，**但页面没有把两者连起来说**）。

第二行的描述是一句"允许扩展要预渲染的路由"，参数写作 `ctx`。**它与 `addPrerenderRoutes` 的关系文档没写**——一个是钩子、一个是函数，**从名字与用途看是同一件事的两条入口，但这属于推断**，两页之间没有任何一句互相引用。

## 用法对照

| 你要做的 | 挂哪一行 |
| --- | --- |
| 给自动导入加预设来源 | `imports:sources` |
| 加单条导入项 | `imports:extend` |
| 改导入目录 | `imports:dirs` |
| 改组件扫描目录 | `components:dirs`（**注意它在 `app:resolve` 之内**） |
| 往组件清单里加东西 | `components:extend` |
| 改 Nitro 配置（含 publicAssets） | `nitro:config` |
| 用 Nitro 实例、注册它的钩子 | `nitro:init` |
| 动最终产物里的公共资源 | `nitro:build:public-assets` |
| 增加要预渲染的路由 | `prerender:routes` |

这一组没有记载的：`imports:*` 四行与 `imports:context` 的先后、`components:dirs` 与图层目录的合并规则（多图层时 `dirs` 里都有谁）、`nitro:build:public-assets` 之后是否还会再复制一次、以及 `prerender:routes` 的 `ctx` 对象里有什么字段。

下一篇读三十三到四十四行：构建报错、`prepare:types`、schema 四连，以及 Vite 那五行里被标弃用的两行。

## 本文依据

- https://nuxt.com/docs/4.x/api/advanced/hooks
