---
url: /posts/bad0b73402e6b30f993537fb1fbe08db/
title: 684 - 模块的配置校验与错误提示：schema 那一行、defaults 的两种写法与三种提示位置
date: 2027-05-28T09:00:00+08:00
updated: 2027-05-28T09:00:00+08:00
author: cmdragon

summary:
  摘要：模块发现用户的配置不对，能做的事比想象中少。官方对选项校验只写了 `schema` 一行："如果提供了它，选项会被应用到这个 schema 上"，没写支持哪种校验库、失败在哪一步报。所以真实的做法是三件：用 `defaults` 兜住缺省，在 `setup` 里自己检查，然后用合适的位置把话说出来——日志实例、抛错，或者交互式终端里的通知。另外有一条容易被忽略的规矩：生命周期钩子里抛出的错误**只会被记录下来，不会中断构建**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 模块开发
  - 配置校验
  - 错误处理
  - 开发者体验
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 先看你手上有什么

`defineNuxtModule` 的定义对象一共八个键，跟"用户配置传进来对不对"有关的只有三行，把原文抄下来：

| 键 | 类型 | 必填 | 文档说明 |
| --- | --- | --- | --- |
| `defaults` | `T \| ((nuxt: Nuxt) => T)` | 否 | 模块的默认选项。如果传的是函数，会以 Nuxt 实例作为第一个参数调用它 |
| `schema` | `T` | 否 | 模块选项的 schema。**如果提供了它，选项会被应用到这个 schema 上** |
| `hooks` | `Partial<NuxtHooks>` | 否 | 为模块安装的钩子；提供了，模块就会安装这些钩子 |

`schema` 这一句就是官方在整页里关于"校验"的全部文字。它没有说这个 schema 要长什么样、支持 Zod 还是 Valibot、校验失败是在 `setup` 之前抛还是在构建末尾汇总、错误消息由谁格式化。**这四点在 4.x 的文档里都是未说明**，要确认只能自己造一个模块试。

也就是说，默认路径是：类型层面靠 TypeScript，运行时的检查得你自己写。

## `defaults` 是你的第一道兜底

官方示例给的就是这个形状：

```ts
import { defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  meta: {
    name: 'my-module',
    configKey: 'myModule',
  },
  defaults: {
    // Module options
    enabled: true,
  },
  setup (options) {
    if (options.enabled) {
      console.log('My Nuxt module is enabled!')
    }
  },
})
```

用户在 `nuxt.config` 的 `myModule` 键下覆盖它。`defaults` 有两种写法（值或函数），差别在于函数能读到 Nuxt 实例——需要按项目情况给不同缺省时才用函数。

还有一条规定值得提前知道，因为它决定了你的提示什么时候**不该**出现：

> 用户还可以把配置键整个设成 `false` 来彻底禁用一个模块。这会阻止该模块的 setup 函数运行，但仍然会为其选项生成类型。

所以检查逻辑写在 `setup` 里天然安全；写在 `setup` 外面（模块文件的顶层）就会被禁用之后照样执行。

## 钩子里抛错不会中断构建，这是原话

讲生命周期钩子那节的第二句很关键：

> 生命周期钩子在主 `setup` 函数之前运行；如果某个钩子抛出错误，这个错误会被记录下来，但不会中断构建过程。

配合另外三条事实一起读：`onInstall` 只在模块**第一次**被加入项目时运行一次；`onUpgrade` 在版本每次升高时运行（用 semver 比较），每个版本只跑一次；这两个钩子都要求 `meta.name` 和 `meta.version` 都已定义。

它的实际含义是：**别指望用抛错来拦住一次错误的安装**。想在装模块的时候拦住不合适的组合，得靠 `setup` 自己抛，或者用 `moduleDependencies` 去声明版本约束（那属于依赖声明，另一篇讲它的四个字段）。

官方给的钩子示例里，`onUpgrade` 是这样提示用户的：

```ts
  onUpgrade (nuxt, options, previousVersion) {
    // This runs when the module is upgraded to a newer version
    console.log(`Upgrading my-awesome-module from ${previousVersion} to 1.2.0`)

    if (isLess(previousVersion, '1.1.0')) {
      console.log('⚠️  Breaking changes in 1.1.0 - please check the migration guide')
    }
  },
```

（`isLess` 来自 `verkit`，是文档示例里的用法；示例本身用 `console.log`，下一节说为什么实际更该用日志实例。）

## 话说出来有三个位置，各管一件事

Kit 在日志那一节给了两个函数。逐项拆解——`useLogger` 的两个参数、`NuxtTerminal` 的五个成员——在讲 Kit 日志函数的那篇已经做过，这里只解决一个问题：**这种情况该用哪一个**。

| 情况 | 该用 | 理由 |
| --- | --- | --- |
| 缺了一个可兜底的选项，想用缺省继续跑 | `useLogger('my-module')` 的 info/warn | 只是告知，用户不需要动作；`tag` 会显示在时间戳右侧，能认出是谁打的 |
| 缺的东西没法兜底，继续跑只会产出坏结果 | 在 `setup` 里抛错 | 生命周期钩子里抛错不会中断构建，`setup` 里抛才有拦截效果 |
| 需要用户点一下头（装依赖、生成文件） | `useTerminal()` 的 `prompt`，且先读 `interactive` | CI 里没有交互宿主，`prompt` 永远等不到答案 |
| 用户必须读完并确认才能走 | `notify` | 它会停在屏幕上直到用户确认或调用 `dismiss()`，所以不能放在每次构建都走到的分支上 |

另外，把日志级别做成可关的是官方示例里的写法：`useLogger('my-module', { level: options.quiet ? 0 : 3 })`——它假设你的模块选项里有一个 `quiet`，把它放进 `defaults` 就能兜住。

## 慢这件事，Nuxt 已经替你警告了

配置没问题但模块很慢，用户看到的是另一条消息。最佳实践那一节的原话：

> 要注意异步行为，因为 Nuxt 会等你的模块 setup 完成，才继续下一个模块、才启动开发服务器和构建流程。**把耗时的逻辑推迟到 Nuxt 钩子里去做。**

> 如果你的模块 setup 耗时超过 **1 秒**，Nuxt 会就此发出警告。

这条警告不是你能关掉的。它同时也指了另一条路：一次性初始化的工作（生成配置文件、建数据库结构）放 `onInstall`，版本相关的迁移放 `onUpgrade`，`setup` 里只留每次构建都必须做的事。

## 一份最小提示策略

- `defaults` 里给出所有可兜底的缺省，别让用户漏填就跑不起来。
- 检查放 `setup` 开头。被 `configKey: false` 禁用时，这些检查自然不执行。
- 说缺省值用了 `logger`，并且 tag 用模块名；给一个 `quiet` 类选项，用 `{ level: ... }` 接住。
- 需要拦截的错误在 `setup` 里抛，不要放在生命周期钩子里抛。
- 需要用户回答之前先读 `interactive`；CI 场景要有默认分支。
- 耗时超过一秒的事挪到钩子或 `onInstall`，别留在 `setup` 里。
- `schema` 的校验行为官方没写，若你要依赖它，先在自己的 playground 里验证失败时的表现。

下一篇把镜头拉远，谈模块的版本兼容：Kit 的七个检查函数，和 `isNuxt2`、`isNuxt3` 上那句 `This may be removed in @nuxt/kit v5`。

## 下一篇

下一篇 685：模块与服务端：注册一条 API 路由要从哪几个约定开始

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/modules
- https://nuxt.com/docs/4.x/api/kit/logging
- https://nuxt.com/docs/4.x/guide/modules/best-practices
