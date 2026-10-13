---
url: /posts/4888c5e722898655f5d0edfee364466b/
title: 672 - 构建期钩子三十三到四十四：构建报错、类型准备、listen、schema 与 vite
date: 2027-05-24T09:00:00+08:00
updated: 2027-05-24T09:00:00+08:00
author: cmdragon

summary:
  摘要：这十二行里最该记住的是 **`prepare:types`——它的描述点明了"谁在它之后动手"：在 `@nuxt/cli` 写出 TypeScript 配置文件（`.nuxt/tsconfig.app.json`、`.nuxt/tsconfig.server.json` 等）与 `.nuxt/nuxt.d.ts` 之前调用，允许往 `nuxt.d.ts` 里加自定义 references 与 declarations，或直接修改生成配置里的选项**。**schema 四连**是默认 schema 从扩展、解析到写出的一圈（`schema:extend`、`schema:resolved`、`schema:beforeWrite`、`schema:written`，最后一行无参数）。**Vite 五行里有两行带着弃用标记**：`vite:extendConfig` 与 `vite:configResolved` **在 Nuxt 5+ 弃用，理由写在同一句里——Nuxt 5 里它作用于共享配置，而不是分开的 client/server 配置**；而 `vite:extend` 没有这个标记。另有 `build:error`（构建期出错时）与 `listen`（**dev server 正在加载时**，参数 `listenerServer, listener`）。

categories:
  - 前端开发

tags:
  - nuxt4
  - 构建期钩子
  - 类型准备
  - schema
  - Vite钩子
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模块要给项目补类型、或者想知道自己的类型声明为什么没生效，位置就在 `prepare:types`。这一组十二行还包含第一批被标弃用的构建期钩子。

## 十二行照录

| 钩子 | 参数 | 描述 |
| --- | --- | --- |
| `build:error` | `error` | 在构建期发生错误时调用 |
| `prepare:types` | `options` | 在 `@nuxt/cli` 写出 TypeScript 配置文件与 `nuxt.d.ts` **之前**调用 |
| `listen` | `listenerServer, listener` | 当开发服务器正在加载时调用 |
| `schema:extend` | `schemas` | 允许扩展默认 schemas |
| `schema:resolved` | `schema` | 允许扩展已解析的 schema |
| `schema:beforeWrite` | `schema` | 在写出给定 schema 之前调用 |
| `schema:written` | - | 在 schema 被写出之后调用 |
| `vite:extend` | `viteBuildContext` | 允许扩展 Vite 默认上下文 |
| `vite:extendConfig` | `viteInlineConfig, env` | 允许扩展 Vite 默认配置。**Nuxt 5+ 弃用**：Nuxt 5 里它作用于共享配置而非分开的 client/server 配置 |
| `vite:configResolved` | `viteInlineConfig, env` | 允许读取已解析的 Vite 配置。**Nuxt 5+ 弃用**，同上 |
| `vite:serverCreated` | `viteServer, env` | 当 Vite 服务器被创建时调用 |
| `vite:compiled` | - | 在 Vite 服务器被编译之后调用 |

## `prepare:types` 那句为什么重要

它的描述是这张表里最长的一段，因为它把**时机、产物、可做的事**三样一起写了：

> Called before `@nuxt/cli` writes TypeScript configuration files (`.nuxt/tsconfig.app.json`, `.nuxt/tsconfig.server.json`, etc.) and `.nuxt/nuxt.d.ts`, allowing addition of custom references and declarations in `nuxt.d.ts`, or directly modifying the options in generated configurations

拆成四点：**时机是"在 `@nuxt/cli` 写文件之前"**；**被写的东西是那些 tsconfig 与 `nuxt.d.ts`**；**你能做两件事：往 `nuxt.d.ts` 里加自定义 references 与 declarations，或直接修改生成配置里的选项**；参数写作 `options`。

为什么这行必须记住：**官方文档里"我的类型怎么不生效"的标准答案就在这**。前面见过的两处都挂它：

- **模块高级配方给的做法**：`nuxt.hook('prepare:types', ({ references }) => { references.push({ path: template.dst }) })`——注意推的是 **`template.dst`（目标路径）**，不是 `filename`。
- **扩展 tsconfig 那条路**：同一个钩子解构出 `references`、`sharedReferences`、`nodeReferences` 三个数组，分别对应 app、shared、node 上下文；**服务端上下文要用 `nitro:prepare:types`**（**这个名字在本组十二行里没有出现**，它在配方页，也是 `nitro:` 前缀在钩子表里的一个例外位置）。

还有一条与它相连的运维事实：**这些 tsconfig 是生成物、不进版本控制**，所以**CI 里跑测试或类型检查之前要先 `nuxt prepare`**，否则会报 `TS5083: Cannot read file '~/.nuxt/tsconfig.json'`。**`prepare:types` 正是那次准备里被触发的钩子**，而生成 tsconfig 本体的函数是 `writeTypes`。

## schema 四连

四行的描述各不相同，但**串起来是一条流水线**：

| 顺序 | 钩子 | 描述 |
| --- | --- | --- |
| 扩展 | `schema:extend` | 允许扩展默认 schemas（参数是复数 `schemas`） |
| 解析后 | `schema:resolved` | 允许扩展已解析的 schema（参数单数 `schema`） |
| 写之前 | `schema:beforeWrite` | 在写出给定 schema 之前调用 |
| 写之后 | `schema:written` | 在 schema 被写出之后调用（**无参数**） |

**参数从复数变单数这件事本身就是一条信息**：`extend` 拿到的是多个 schema，`resolved` 与 `beforeWrite` 拿到的是那一份已解析／将要写出的 schema。

这四行管的是**配置项的元定义（schema）本身**，而不是 `nuxt.config` 的值——**普通模块不需要碰它**。要判断自己是否该用，标准是：**你在新增或修改一个配置键的类型定义**。

## `build:error` 与 `listen`

**`build:error`**：构建期发生错误时调用，参数是 `error`。

这一行值得单独提，是因为**它是"错误发生"而不是"错误处理"**：错误页那一章讲的是运行时的用户可见错误，**构建失败在这里给了一个可挂的点**，但**页面没有说挂了之后能否阻止构建失败**（**文档未说明**）。

**`listen`**：当开发服务器正在加载时调用，参数是两个——`listenerServer, listener`。

这行**是全表少数与"端口/监听"有关的**，参数有两个但**描述只说"正在加载时"**，没有说明哪个是 Node server、哪个是 listener 对象（**未说明**）。

## Vite 五行：两行弃用、三行还在

这一组第一次在构建期表里出现弃用标记，而且**理由直接写在同一句里**：

> **Deprecated in Nuxt 5+.** In Nuxt 5, this operates on a shared configuration rather than separate client/server configs.

**弃用的是"分 client/server 配置"这个模型本身**，所以那两个钩子的语义在 Nuxt 5 里不成立——这与 `nuxt.config` 里 Vite 那一族配置项的处境是同一件事的两面。

三行没有标记的各有分工：

| 钩子 | 能力 |
| --- | --- |
| `vite:extend` | 扩展 Vite 的默认**上下文**（参数 `viteBuildContext`） |
| `vite:serverCreated` | Vite 服务器被创建时（参数 `viteServer, env`） |
| `vite:compiled` | Vite 服务器编译之后（**无参数**） |

**`vite:extend` 是 Kit 唯一示例页那段代码用的钩子**——那一节用它把配置取出来，然后靠抛错叫停构建。**这正好说明为什么它还活着：它拿的是上下文，不是"某一侧的配置"**。

另外四个 Vite 钩子的参数里都有 `env`（`vite:extendConfig`、`vite:configResolved`、`vite:serverCreated`），**`env` 具体是什么，表里没写**——只能从"它区分 client/server"这个上下文推出来，而这恰是 Nuxt 5 要取消的东西。

## 什么时候用哪一行

| 你要做的事 | 挂哪一行 |
| --- | --- |
| 往 `nuxt.d.ts` 加引用或声明 | **`prepare:types`** |
| 读或改生成的 tsconfig 选项 | `prepare:types`（同一行的第二种能力） |
| 构建失败时做点收尾 | `build:error`（**能否改变失败结果未说明**） |
| 新增/修改配置项的类型定义 | `schema:extend` → `schema:resolved` → `schema:beforeWrite` |
| 取 Vite 上下文（示例页那种用法） | `vite:extend` |
| 按 client/server 分别改 Vite 配置 | **不要用 `vite:extendConfig`**，Nuxt 5+ 已弃用；改用带 `applyToEnvironment` 的插件 |

这一组未记载的还有：`prepare:types` 与 `nitro:prepare:types` 的先后、`listen` 两个参数各自类型、`schema:written` 之后配置类型文件是否会再写一次、以及 `vite:compiled` 与 `webpack:compiled` 是否共用同一套完成语义（webpack 那八行在下一组）。

下一组读四十五到五十二行，全是 webpack；再往后是 Nitro 运行时那九行。

## 下一篇

下一篇 673：构建期钩子四十五到五十二：webpack 八行与那三条 WebpackBar 事件

## 本文依据

- https://nuxt.com/docs/4.x/api/advanced/hooks
