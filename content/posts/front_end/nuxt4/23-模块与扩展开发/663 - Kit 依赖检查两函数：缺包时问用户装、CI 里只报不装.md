---
url: /posts/92d169a1f2dd5b90eb74077d0be3cbef/
title: Kit 依赖检查两函数：缺包时问用户装、CI 里只报不装
date: 2027-05-21T09:00:00+08:00
updated: 2027-05-21T09:00:00+08:00
author: cmdragon

summary:
  摘要：有些包不是人人都要——用 SCSS 才需要 `sass`。这一页给的两个函数就是为这种"可选依赖"准备的：`ensureDependencyInstalled` 检查是否装好、缺了就提示用户安装；`getAddDependencyCommand` 返回一条与用户包管理器匹配的安装命令，从 `cwd` 检测、检测不到回落到 `npm`。三条硬规则最容易记错：**返回形态随入参而变——传一个字符串得 `Promise<boolean>`，传数组得 `Promise<true | string[]>`，缺哪些就返回哪些名字**；**"没有交互终端"与"在 CI 里"一律不装，只把缺的包连命令一起报成诊断**；**`prompt` 是三态：`true` 永远问、`false` 永远不问（在 StackBlitz 上会直接装）、不设则除 StackBlitz 外都问**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 可选依赖
  - 包管理器
  - NuxtKit
  - 交互提示
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模块里写 `import 'sass'` 之前，得先确认用户装没装。这一页给的不是 `try/catch`，而是两个会**跟用户说话**的函数。

## 这页要解决的问题

页面定性：**Nuxt Kit 提供工具来检查可选依赖是否已安装，并告诉用户如何安装它们**。

开场把两种坏做法点掉了：**不要假设那个包就在那里**，**也不要打印一条用户根本不用的包管理器的安装命令**。它给的做法是：**检查依赖能否被解析，并提出用项目自己的包管理器来安装**。

这两句就是这页的设计目标：**一个函数负责"要不要装"，一个负责"怎么装才对"**。

## `ensureDependencyInstalled`：检查并提示

```ts
function ensureDependencyInstalled (names: string, options?: EnsureDependencyInstalledOptions): Promise<boolean>
function ensureDependencyInstalled (names: string[], options?: EnsureDependencyInstalledOptions): Promise<true | string[]>
```

**两个重载的返回类型不一样**，这是调用时必须先决定的事。定性：**检查一个或多个依赖是否已安装，并提示用户安装缺失的那些**。

官方示例就是那个"缺了就走"的形状：

```ts
import { defineNuxtModule, ensureDependencyInstalled } from '@nuxt/kit'

export default defineNuxtModule({
  async setup () {
    if (!await ensureDependencyInstalled('sass')) {
      return
    }
  },
})
```

参数两项：**`names` 是要检查、缺失时就安装的一个或多个包名**；`options` 四项都有默认值：

| 属性 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `rootDir` | `string` | `nuxt.options.rootDir` | **从中解析包、并装进去的目录** |
| `searchPaths` | `string[]` | `nuxt.options.modulesDir` | 额外的解析目录 |
| `from` | `string` | `undefined` | 额外的一个 URL 或路径用来解析包，例如 `import.meta.url`；**当你自己的包把一个依赖作为 peer dependency 带进来时特别有用** |
| `prompt` | `boolean` | `undefined` | 安装前是否提示；**`true` 永远提示，`false` 永远不提示（在 StackBlitz 上会自动安装），不设则除了 StackBlitz 之外都提示** |

`prompt` 那一格是三态而不是布尔：**"不设"与"设成 `false`"在 StackBlitz 上结果相同（都不问），在别处不同**——前者问、后者不问。

## 返回值与"不装"的三种原因

Return Value 一节把两种入参形态的结果都写清了：

**传单个包名**：可用返回 `true`，不可用返回 `false`。

**传数组**：**每个包都可用时返回 `true`，否则返回那些仍然缺失的包名数组**。缺失原因页面列了三种：**用户拒绝、安装失败、没有交互终端**。

然后那条 `::note` 是这一节最该抄进设计的规则：

> Nothing is installed in CI or when there is no interactive terminal. In that case the missing packages are reported as a diagnostic that includes the install command, and the function reports them as missing so your module can degrade gracefully.

拆成三句：**CI 里或没有交互终端时什么都不装**；**这种情况下缺失的包会被报成一条包含安装命令的诊断**；**函数同时把它们报成缺失，好让你的模块能优雅降级**。

所以写法上的结论很直接：**不要因为 `false` 就抛错**，那正是设计让你"退化为不启用某功能"的信号。示例里那个 `return`（静默放弃本次功能）就是一种降级写法。

## `getAddDependencyCommand`：命令要匹配包管理器

```ts
function getAddDependencyCommand (names: string | string[], cwd: string, options?: { dev?: boolean }): Promise<string>
```

定性句把它的用途写死了：**取得用户应该运行的、给项目添加依赖的命令；**从 `cwd` 检测包管理器，**检测不到时回落到 `npm`**。**每当你打印一条安装指令时都应该用它，这样命令才与项目的包管理器一致**。

参数三项：**`names` 一个或多个包名**、**`cwd` 是"从哪个目录检测包管理器"**（注意它没有默认值，必须传）、**`options.dev` 决定命令是否把这些包装成 dev 依赖**。

它与上面那个函数怎么配合，示例给了完整一对：

```ts
import { defineNuxtModule, getAddDependencyCommand, useLogger } from '@nuxt/kit'

export default defineNuxtModule({
  async setup (options, nuxt) {
    const logger = useLogger('my-module')
    const command = await getAddDependencyCommand('sass', nuxt.options.rootDir, { dev: true })

    logger.warn(`\`sass\` is required. Run \`${command}\` to install it.`)
  },
})
```

三处细节：**传的是 `nuxt.options.rootDir`**（与 `ensureDependencyInstalled` 的默认解析目录一致）、**`{ dev: true }` 让它成为 dev 依赖**（`sass` 这类只在构建期用，所以放 dev）、**输出走 `useLogger('my-module').warn`** 而不是 `console.warn`，于是消息带上模块标签。

## 什么时候用哪个

| 你要做的 | 用哪个 |
| --- | --- |
| 缺了就要么装上、要么本次不启用功能 | `ensureDependencyInstalled` |
| 只是给用户印一条正确命令 | `getAddDependencyCommand` |
| 必需依赖 | **都不用**——写进你包的 `dependencies`，或用 `moduleDependencies` 声明模块依赖 |
| 需要按版本约束检查 | `moduleDependencies` 的 `version`（**本页没有版本相关参数**） |

这条分工值得说清：**`moduleDependencies` 处理的是"Nuxt 模块"之间的依赖，本页两个函数处理的是"npm 包"**——比如 `sass`、`autoprefixer` 这类 loader 需要的包，不是模块。

文档未记载的：**`ensureDependencyInstalled` 安装时的具体命令是什么**（它只说"用项目自己的包管理器"）、**装完是否会自动重启或触发重建**、**`from` 那个"peer dependency"场景的判断依据**、以及**提示被拒绝后重复调用会不会再问**（**页面完全没写**。若需要"只问一次"的行为，得自己记状态）。

另有一条来自 Kit 总页的前提仍然适用：**这两个函数是构建期工具，不该出现在运行时代码里**。

下一篇讲 Kit 的 examples 页——它只有四个函数以外的一例，示范怎么把 Vite 配置取出来。

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/dependencies
