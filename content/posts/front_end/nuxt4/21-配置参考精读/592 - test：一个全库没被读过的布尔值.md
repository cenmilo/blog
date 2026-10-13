---
url: /posts/d307203750fa5b629387cc5d52ff48ec/
title: 592 - test：一个全库没被读过的布尔值
date: 2027-04-27T10:00:00+08:00
updated: 2027-04-27T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：`## test` 的记载是**一句 ＋ 类型 ＋ 默认值**："Whether your app is being unit tested."、`**Type**: boolean`、`**Default:** false`，**没有示例、没有 See、也没有第二句**。反常之处在于：**整套 4.x 文档里没有一处把这个键读出来用**——文档中找不到任何 `nuxt.options.test` 的写法，也没有任何一处说明"设成 `true` 会改变什么"。而唯一提到"测试模式"的第二处记载，是特性页里 `devLogs` 那句 "By default, this is enabled in development (**when test mode is not active**)"，可 `nuxt test` 命令那一节又说它设的是 **`process.env.NODE_ENV` 为 `test`**——**所以那句"test mode"到底指配置键还是指环境变量，文档没有区分。**

categories:
  - 前端开发

tags:
  - nuxt4
  - 单元测试
  - 测试模式
  - test-utils
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

配置参考里有一个 `test` 键，写着"你的应用是否正在被单元测试"。想靠它切换点什么的时候会发现一个尴尬事实：**这套文档里没有任何地方读它。**

## 这一节的全部记载

> Whether your app is being unit tested.

- **Type**: `boolean`
- **Default:** `false`

一句、一型、一默认，到此为止。**没有 Example，没有 See，没有任何一句解释设成 `true` 会发生什么。**

主语还是 Nuxt（"你的应用**是否正被**单元测试"），和 `dev` 那节同一种句式，读法也是**状态**而不是开关。但 `dev` 那节至少写了 "Normally, you should not need to set this."，**这一节连这句都没有**——它没有告诉你这一项该由谁来设。

## 全库没有一处读它

文档里 `dev` 这个状态有明确的使用面：模块结构页 `console.log(nuxt.options.dev)`、构建器页"bundle 实现在 `nuxt.options.dev` 上分支"、模块作者迁移页的 `if (nuxt.options.dev) { }` 守卫。

**`test` 这边一条都没有。** 整套 4.x 文档里找不到 `nuxt.options.test` 的写法，也找不到任何一句"当 `test` 为真时，X 行为改变"。所以：

- 设它有没有效果——**无依据**。
- 它期望由谁来设（测试工具自动注入？还是用户写在配置里？）——**无记载**。
- 它与 `@nuxt/test-utils` 的关系——`getting-started/testing` 那一节通篇讲 test-utils 的安装与用法，**没有提到这个配置键**。

## 唯一相邻的那句"test mode"

特性页 `devLogs` 条目的默认状态句：

> By default, this is enabled in development (when test mode is not active).

这是全库第二处涉及"测试模式"的表述，也是唯一一处**某个行为明确随测试模式改变**的记载。但紧接着的歧义是：

`nuxt test` 命令那一节写的是："This command sets `process.env.NODE_ENV` to `test` **if not already set**."（同样的话在那一节的 `::note` 里又重复了一次。）那一节给的用法行是这样的：

```bash [Terminal]
npx nuxt test [ROOTDIR] [--cwd=<directory>] [--dev] [--watch]
```

也就是说，跑 `nuxt test` 时被改变的公开契约是 **`NODE_ENV=test`**，而不是这一节那个配置键。**"test mode" 这个词在文档里指哪一个，没有定义**——如果指的是 `NODE_ENV`，那 `## test` 这一节就完全没人读；如果指的是配置键，那又没有任何一句说明它怎么被设成真。两种读法都堵着。

## 三个同名东西要分开

| 名字 | 是什么 | 出处 |
| --- | --- | --- |
| `test`（配置节） | 一个布尔状态位，**本系列未见任何读取处** | `api/nuxt-config` |
| `nuxt test`（命令） | 用 `@nuxt/test-utils` 跑测试；设 `process.env.NODE_ENV` 为 `test`；参数 `ROOTDIR`（默认 `.`），选项 `--cwd=<directory>`、`--dev`（Run in dev mode）、`--watch`（Watch mode） | `api/commands/test` |
| `@nuxt/test-utils`（库） | 端到端与单元测试的工具集，可选依赖里在 `happy-dom`/`jsdom` 之间选运行时、在 `vitest`/`cucumber`/`jest`/`playwright` 之间选 runner | `getting-started/testing` |

顺带把第四个坑记下来：`nuxt test` 有个 **`--dev`** 选项（描述是 "Run in dev mode"）。所以"测试"和"dev 模式"并不互斥——这与 `devLogs` 那句"开发里默认开、测试模式不激活时"要放在一起读，别以为跑测试就一定没有 dev 行为。

## 自查

- `## test`：**一句 ＋ `boolean` ＋ `false`**，无 Example、无 See、无第二句。
- **全库没有任何一处读它**（无 `nuxt.options.test` 写法、无"设了会怎样"的说明）；由谁设、有何效果均无记载。
- 唯一随"测试模式"变的行为是 `features.devLogs` 那句，而 `nuxt test` 命令给的公开机制是 **`NODE_ENV=test`**——**两处口径未对齐**。
- 分清配置键 / `nuxt test` 命令 / `@nuxt/test-utils` 库三件事；命令还有 `--dev` 选项，测试与 dev 模式不互斥。

## 下一篇

下一篇 593 看 `## spaLoadingTemplate`：这一节的**定性句自己就是类型说明**（"Boolean or a path to an HTML file with the contents of which will be inserted into any HTML page rendered with `ssr: false`."），**有默认值 `null` 却没有类型行**，三条分支（不设／`false`／`true`）在原文里被挤成同一段，另有两个 spinner 素材外链和一份 CSS loader 的 HTML 示例。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## test` 一句原文、`**Type**: boolean` 与 `**Default:** false`，无 Example/See 的事实；`## dev` 的对照句式）
- https://nuxt.com/docs/4.x/api/commands/test（命令用法行、"sets `process.env.NODE_ENV` to `test` if not already set" 正文与 `::note`、Arguments 一行与 Options 三行含 `--dev`/`--watch` 描述）
- https://nuxt.com/docs/4.x/getting-started/testing（`@nuxt/test-utils` 定位句、可选依赖三行：`happy-dom`/`jsdom`、`vitest`/`cucumber`/`jest`/`playwright`、`playwright-core`；那一节未提及 `test` 配置键）
- https://nuxt.com/docs/4.x/guide/going-further/features（`devLogs`："By default, this is enabled in development (when test mode is not active)."）
