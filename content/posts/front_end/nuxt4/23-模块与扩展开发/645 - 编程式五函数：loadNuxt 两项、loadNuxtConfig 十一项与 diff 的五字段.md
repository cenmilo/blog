---
url: /posts/640b3af3157ad9dcdd2263d7b1e7df11/
title: 645 - 编程式五函数：loadNuxt 两项、loadNuxtConfig 十一项与 diff 的五字段
date: 2027-05-15T09:00:00+08:00
updated: 2027-05-15T09:00:00+08:00
author: cmdragon

summary:
  摘要：不敲命令、用代码把 Nuxt 跑起来，靠的是这一节的**五个函数**：`loadNuxt`、`buildNuxt`、`loadNuxtConfig`、`diffNuxtConfig`、`writeTypes`。两处细节最容易被漏：**`loadNuxt` 的 `ready` 默认是 `true`，设成 `false` 之后必须自己调 `nuxt.ready()`**；**`loadNuxt` 底层用的是 c12，所以它接受 `c12.loadConfig` 的同款选项**。`loadNuxtConfig` 的参数表有**十一项**（`cwd`、`configFile`、`rcFile`、`globalRc`、`overrides`、`defaults`、`dotenv`、`envName`、`resolve`、`import`、`onConfigResolved`），而 `diffNuxtConfig` 比的正是 `onConfigResolved` 给你的那份 `rawConfig` 快照——**这份快照刻意排掉了 schema 默认值、`overrides`、`defaults` 与 `defaultConfig`，为的是"项目没变时快照也不变"**，diff 出来的每条带**五个字段**，其中 `path` 是数组、`label` 是可展示的访问器字符串。

categories:
  - 前端开发

tags:
  - nuxt4
  - 编程式用法
  - 配置加载
  - 配置比对
  - CLI工具
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

写 CLI、写测试工具、或者只想在脚本里拿一份最终的 Nuxt 配置——这些都需要绕开命令行。官方把这条路叫做编程式用法，工具一共五个。

## 这一节给的场景

页面开场两句：**这些函数允许你加载 Nuxt、构建 Nuxt、加载 Nuxt 配置**；**编程式用法有用的场合，例如构建 CLI 工具或测试工具**（两处各给了一个仓库链接：nuxt/cli 与 nuxt/test-utils）。

五个函数的分工：

| 函数 | 干什么 | 返回 |
| --- | --- | --- |
| `loadNuxt` | 加载配置、实例化，**并返回 Nuxt 实例的 promise** | `Promise<Nuxt>` |
| `buildNuxt` | **调用构建器**把应用打包 | `Promise<any>` |
| `loadNuxtConfig` | 只拿配置对象 | `Promise<NuxtOptions>` |
| `diffNuxtConfig` | 比较两份配置快照、返回差异 | 差异条目数组 |
| `writeTypes` | 生成 `tsconfig.json` 并写进项目的 buildDir | `void` |

## `loadNuxt`：两个选项，其中一个是坑

```ts
function loadNuxt (loadOptions?: LoadNuxtOptions): Promise<Nuxt>
```

关于它的选项来源有一句很关键的说明：**`loadNuxt` 底层使用 c12，因此它接受与 `c12.loadConfig` 相同的选项，另外再加几个**：

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `dev` | `boolean` | `false` | 设为 `true` 时，Nuxt 以开发模式加载 |
| `ready` | `boolean` | **`true`** | 为 `true` 时调用完 `loadNuxt` 就可使用；**为 `false` 时你需要自己调 `nuxt.ready()` 才能确认它可用** |

`ready` 是这一节唯一一个**默认值为 `true`** 的开关，而它的文档写法是"Required: `true`"，实际含义是**默认就 true、可以显式关掉**。关掉它的收益是启动更早返回；代价是**忘了调 `nuxt.ready()` 时，后面所有依赖实例的动作都在一个还没准备好的对象上跑**。

## `buildNuxt`：一句话和一句限定

```ts
function buildNuxt (nuxt: Nuxt): Promise<any>
```

描述里有一句版本信息：**它会调用构建器（目前是 `@nuxt/vite-builder` 或 `@nuxt/webpack-builder`）来打包应用**。参数只有一个 `nuxt` 实例，并且补了一句来源：**可以通过 `useNuxt()` 从上下文取得**。

**"目前"这两个字值得留意**：这一节只列了 vite 与 webpack 两种构建器，而兼容性那一节的约束表里把 `rspack` 也当成一个可指名的 builder——**两处口径不一致，本系列按各自原文记录，不合并推断**。

## `loadNuxtConfig`：十一项，逐项有默认值

```ts
function loadNuxtConfig (options: LoadNuxtConfigOptions): Promise<NuxtOptions>
```

`options` 的定性是"控制配置如何被定位、合并与加载"。十一项全列：

| 属性 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `cwd` | `string` | `process.cwd()` | 从哪个目录加载 `nuxt.config` |
| `configFile` | `string` | `'nuxt.config'` | 要加载的配置文件名，**不含扩展名** |
| `rcFile` | `string \| false` | `'.nuxtrc'` | 与配置文件一起加载的 `.rc` 文件名；**给 `false` 表示不加载** |
| `globalRc` | `boolean` | `true` | 同时加载用户级与工作区级的 `.nuxtrc` |
| `overrides` | `NuxtConfig` | `undefined` | 应用**在每一层之上**的配置，**包括根项目自己的 `nuxt.config`** |
| `defaults` | `NuxtConfig` | `undefined` | 应用**在每一层之下**、**在 schema 默认值之前**的配置 |
| `dotenv` | `boolean \| NuxtOptions` 形式 | `true` | 在解析配置前把 `.env` 读进 `process.env`；**环境已经填好时设为 `false`** |
| `envName` | `string \| false` | `undefined` | 用于挑选 `$env.*` 配置覆盖项的环境名；**优先级高于 `nuxt.config` 里设的 `envName`** |
| `resolve` | `(source, context) => ResolvedNuxtLayer \| nullish` | `undefined` | **自己解析 `extends` 条目**；返回 nullish 就回落到默认解析 |
| `import` | `(id: string) => Promise<unknown>` | `undefined` | 用自定义加载器导入配置文件，**例如不必让 Nuxt 去用 jiti 就能加载 TS 配置** |
| `onConfigResolved` | `(context) => void` | `undefined` | 配置成功加载后被调用一次、**并且会被 await**；加载抛错时不会被调用 |

三项最有用：

**`overrides` 与 `defaults` 给的是同一次加载里两端的位置**，一个压在所有层之上（连项目自己的配置也被它压住）、一个垫在所有层之下（但在 schema 默认值之前）。**做工具时用它们注入值，比改用户文件干净**。

**`envName` 的优先级方向是"外部盖内部"**：命令行侧传进来的比 `nuxt.config` 里写的更优先。

**`import` 那条点出了 jiti 的位置**：默认加载 TS 配置走的是 jiti，给了自定义 loader 就能替换这一步。

## `onConfigResolved` 给你的那份快照

这一项自己有一个小节，因为传进去的 context 有四个字段：

| 属性 | 类型 | 说明 |
| --- | --- | --- |
| `rawConfig` | `NuxtConfig` | **跨所有层合并后的用户配置**，未套用 schema 默认值，并**排除 `overrides`、`defaults` 与 `defaultConfig`**，所以同一个未改动的项目反复加载会得到不变的快照 |
| `layers` | `NuxtConfigLayer[]` | 已解析的配置层，**优先级高的排在前面** |
| `configFile` | `string?` | 根 `nuxt.config` 的绝对路径，**如果找到了的话** |
| `cwd` | `string` | 配置从哪个目录加载的 |

`rawConfig` 那串排除条件是**为 diff 设计的**：如果快照里混进了默认值和你注入的 overrides，任何两次加载都可能不一样，比对就失去意义。

## `diffNuxtConfig`：五条字段与一条排除

```ts
function diffNuxtConfig (oldConfig: NuxtConfig, newConfig: NuxtConfig): NuxtConfigDiffEntry[]
```

用途句说得很具体：**比较两份 `rawConfig` 快照，返回它们的差异**——**当你自己监听配置文件、需要在决定要不要重启之前知道哪些键变了**，这个就有用。

每条差异有五个字段：

| 属性 | 类型 | 说明 |
| --- | --- | --- |
| `type` | `'added' \| 'removed' \| 'changed'` | 值是怎么变的 |
| `path` | `Array<string \| number>` | 变化值的属性路径，**数组下标是数字**，例如 `['runtimeConfig', 'public', 'foo']`；**要从配置对象里把值读回来时用这个** |
| `label` | `string` | `path` 写成属性访问器的形式，例如 `runtimeConfig.public.foo` 或 `modules[0]`；**用于展示，或跟已知键名做匹配** |
| `newValue` | `unknown` | **`added` 与 `changed` 才有** |
| `oldValue` | `unknown` | **`removed` 与 `changed` 才有** |

另有一条排除规则：**源码没变的函数不会被报成变化**，所以配置里声明了内联函数的项目**不会每次加载都跟自己 diff 出东西**。

官方给的用法是把两次 `rawConfig` 存起来比：

```ts
import { diffNuxtConfig, loadNuxtConfig } from '@nuxt/kit'

let previous

async function load () {
  await loadNuxtConfig({
    cwd: process.cwd(),
    onConfigResolved ({ rawConfig }) {
      if (previous) {
        for (const entry of diffNuxtConfig(previous, rawConfig)) {
          console.log(`${entry.label} was ${entry.type}`)
        }
      }
      previous = rawConfig
    },
  })
}
```

`path` 与 `label` 的分工是这里唯一要做选择的：**判断"哪个键变了"用 `label` 去匹配字符串，取回变化后的值用 `path` 去索引对象**——因为 `label` 里的数组下标那种形态（`modules[0]`）并不能直接当索引链用。

## `writeTypes`：一行

```ts
function writeTypes (nuxt?: Nuxt): void
```

**生成 `tsconfig.json` 并写入项目的 buildDir**。参数是可选的实例，同样可以从 `useNuxt()` 拿。它对应的是"CI 里跑类型检查之前必须先 prepare"那条要求的底层动作。

## 组合起来的三种用法

| 你想做的事 | 顺序 |
| --- | --- |
| 脚本里完整构建一次 | `loadNuxt()` → `buildNuxt(nuxt)` |
| 只要一份最终配置 | `loadNuxtConfig({ cwd, ... })`，**不实例化 Nuxt** |
| 监听配置、决定要不要重启 | `loadNuxtConfig` 里挂 `onConfigResolved`，把 `rawConfig` 存下来，下次进来先 `diffNuxtConfig` |

这一节没写的：`loadNuxt` 那份 c12 选项清单具体还有哪些、`dev: true` 时实例与 `nuxt dev` 起的服务器差别、以及 `diffNuxtConfig` 在函数源码**变了**时会不会报 `changed`（只写了"源码没变不报"）。

下一篇讲兼容性判断那一节，七个函数，其中两个官方让你别用。

## 下一篇

下一篇 646：Kit 兼容性七函数：三个 check 系列与那两个该换掉的 isNuxt2、isNuxt3

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/programmatic
