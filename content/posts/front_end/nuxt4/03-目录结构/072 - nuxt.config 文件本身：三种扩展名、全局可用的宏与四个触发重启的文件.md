---
url: /posts/a163020f3cb30e394a851da47581fbfe/
title: 072 - nuxt.config 文件本身：三种扩展名、全局可用的宏与那四个触发重启的文件
date: 2026-11-05T09:00:00+08:00
updated: 2026-11-05T09:00:00+08:00
author: cmdragon

summary:
  摘要：nuxt.config 这一节给的规则很短：文件扩展名可以是 .js、.ts 或 .mjs 三种；用 defineNuxtConfig 包一个对象导出，而这个 helper 官方用 tip 明确"全局可用、无需导入"，你也可以按偏好从 nuxt/config 显式导入。但这一节有一条影响日常体感的规则：为了确保配置是最新的，Nuxt 在检测到主配置文件、.env、.nuxtignore 与 .nuxtrc 这几个 dotfile 发生变化时会做一次完全重启。这条与 033 抄过的那句连起来读才有意思——dev 在重启时会说明原因，而 nuxt.config 的变化还会列出真正不同的那些键，所以只改格式的编辑与确实改了配置能区分开。另外三件配套事实值得放一起记：050 用"包含 nuxt.config.ts 的那个目录"来定义根目录，068 要求每一层图层都必须有自己的 nuxt.config.ts（即使是空的）才会被识别为有效图层，而 034 抄过的 dev 调试那一节说明 nuxt.config、模块与构建钩子运行在 CLI 进程里而不是服务端工作线程，所以调试它要连的是下一个端口（默认 9230）。

categories:
  - 前端开发

tags:
  - nuxt4
  - 配置
  - nuxt-config
  - 重启
  - 目录结构

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

018 讲的是 `nuxt.config` 里面写什么。这一篇讲这个文件本身，扩展名怎么选、那个不用导入的宏是什么、以及为什么你改了某个文件之后，整个 dev 服务器会重启而不是热更新。

三种扩展名，和一份最小形态。官方原话，`nuxt.config` 文件的扩展名可以是 `.js`、`.ts` 或 `.mjs`。

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  // My Nuxt config
})
```

只有三种，注意没有 `.cjs`、也没有 `.mts`。对照 051 讲的 `alias`、015 讲的 ESM 与 CJS，这份文件本身就在构建期那一侧被读取，扩展名集合是它自己定的。

那个宏，全局可用，也能显式导入。官方用 tip 标了一句，`defineNuxtConfig` 这个 helper 全局可用，无需导入。

官方接着说，如果你更偏好，也可以显式地从 `nuxt/config` 导入 `defineNuxtConfig`。

```ts [nuxt.config.ts]
import { defineNuxtConfig } from 'nuxt/config'

export default defineNuxtConfig({
  // My Nuxt config
})
```

这两段合起来是一篇小文章，为什么它能不导入就用。因为这份文件是由 CLI 进程读取的，下面它跑在哪一节会讲，框架在加载它时已经把这个宏备好了。全局可用是便利，不是魔法，而 017 讲约定的代价时提醒过，这类隐式全局，正是排查时要认出来的东西。

那四个会触发完全重启的文件。这一节最后那句，是这一篇最该记住的，为了确保你的配置保持最新，Nuxt 在检测到以下变化时会做一次完全重启，主配置文件、`.env`、`.nuxtrc` 与 `.nuxtignore` 这几个 dotfile。

四样列得很具体。`nuxt.config` 管项目主配置，这一节与 050 讲过。`.env` 管环境变量，dev、build、generate 期被读，024、029 讲过。`.nuxtrc` 是另一种配置语法，对全局配置有用，050、047 讲过。`.nuxtignore` 在构建阶段忽略根目录里的文件，050 讲过。

这条规则的实操含义有两条。

一，改 `.env` 会让 dev 完全重启，不是热更新。这正是 018、029 那套运行时配置由环境变量覆盖的机制在开发期的表现，因为 `runtimeConfig` 是在进程启动时按环境解析的。

二，它与 033 抄过的那句正好配对。当 dev 服务器 reload 或 restart 时，它会说明是什么引起的，而 `nuxt.config` 的变化还会列出真正不同的那些键，所以由一次只改格式的编辑触发的重启，可以和一次确实改了配置的重启区分开。reload 原地发生，restart 替换进程，这是 `nuxt.config` 或已安装依赖的变化所需要的。

两句合起来读，这四类文件的改动走的是替换进程那条路，restart，而它会把原因打给你看，`nuxt.config` 那一条甚至会列出具体是哪个键变了。所以我改了配置怎么没生效的第一个排查动作，是去终端看它有没有报这次 restart。

它跑在哪，这就是为什么它不能用浏览器 API。034 抄过 dev 调试那一节的一段，正好用在这里，`nuxt.config`、模块与构建钩子运行在 CLI 进程里，它有自己的 inspector 在下一个端口，默认 `9230`。在 `chrome://inspect` 里把 `localhost:9230` 加为一个目标，点 Configure，或让你的编辑器指向它。

这一句能推出两条使用规则。

`nuxt.config` 里不要碰浏览器对象，`window`、`document`，它根本不在浏览器里跑。

要调试配置或模块加载，断点要打在 9230 那一侧，而不是服务端工作线程那个 9229，034 讲过。

同理，官方在 alias 与图层那两处示例里给的都是构建期写法，`fileURLToPath(new URL(...))`、`createResolver(import.meta.url)`，051、067 讲过。

它同时是两件事的判据。这一篇收尾时把 `nuxt.config` 这个文件在其他四篇里承担的角色串一遍，它的意义远不止写配置。

一，它定义根目录。050 的原文是，Nuxt 应用的根目录，是包含 `nuxt.config.ts` 文件的那个目录。

二，它是图层有效性的门槛。068 抄过，每个图层必须有一个 `nuxt.config.ts` 文件才会被识别为一个有效图层，即使它是空的。

三，图层那份会与主配置合并。068 那份清单第一行的官方说明就是图层专属配置，会与主配置合并。

四，它还能反过来影响自己的生成物。036 抄过，要定制类型配置，入口就是 `nuxt.config` 里的 `typescript.*`，而不是去改 tsconfig 文件。

别在这份文件里做的事，两件。

一，不要放密钥，对外的部分。018 那两条硬规则的落地方式是，能被浏览器读到的只有 `runtimeConfig.public`，`app.config.ts` 更是官方用 caution 明令禁放密钥，055 讲过。`nuxt.config` 顶层写的值是服务端侧的，但它会被打进构建环境，写进 git 就等于交出去，所以真正该走环境变量，029 那条生产环境没人替你加载 `.env`。

这一句是按 018、029、055 三节的规则合起来给的工程判断，官方在这一节没单独写。

二，不要指望 `nuxt.config` 里能读到运行时配置。它跑在构建期 CLI 进程，034 讲过，运行时那份要 `useRuntimeConfig()`，018、065 那个 `useRuntimeConfig(event)` 的例子。

全部选项去哪查。官方在这一节末尾给的入口是配置文档，在 Nuxt configuration 文档里发现所有可用选项。

这一系列里对应的位置，018 讲默认值与环境覆盖的两条硬规则，530 到 599 那七十篇逐节精读这份参考。现在你只要知道这份文件有边界、有基准、有重启代价，读参考时就知道自己在找什么。

## 自查

- [ ] `nuxt.config` 能用的三种扩展名分别是什么？`.cjs` 行不行？
- [ ] `defineNuxtConfig` 为什么要写 import？官方怎么标它的？
- [ ] 改哪四类文件会让 dev 完全重启？重启时终端会告诉你什么？
- [ ] 想在 `nuxt.config` 里加 `window` 相关的判断，会怎样？为什么？
- [ ] 图层为什么「必须有 `nuxt.config.ts` 即使空的」？项目根目录是靠什么定义的？
- [ ] 要定制类型配置，正确入口是哪个文件的哪个键？

## 下一篇

073 讲 `.env`：官方给的时间窗很硬——它只在 `nuxt dev`、`nuxt build` 与 `nuxt generate` 期间被读取，跑构建产物时不读；而 `nuxt preview` 是那个「为了便利」的例外。
