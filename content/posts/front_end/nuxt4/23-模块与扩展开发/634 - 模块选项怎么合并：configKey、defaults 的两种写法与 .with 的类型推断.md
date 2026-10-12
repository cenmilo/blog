---
url: /posts/9d1740af4ec0fb7d44465c4186c9fe49/
title: 模块选项怎么合并：configKey、defaults 的两种写法与 .with 的类型推断
date: 2027-05-11T10:00:00+08:00
updated: 2027-05-11T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：用户在 `nuxt.config.ts` 里写 `myModule: { enabled: false }`，你的 `defaults` 里写的是 `enabled: true`——**`setup` 收到的到底是哪个**？Kit 那一页给的定义句就把答案写出来了：`defineNuxtModule` 的职责是**自动把默认值与用户提供的选项合并**、装上提供的钩子、再调用 setup。参数表一共**八个键**（`meta`、`defaults`、`schema`、`hooks`、`moduleDependencies`、`onInstall`、`onUpgrade`、`setup`，全部非必填）。三条容易踩的规则：**把配置键整个写成 `false` 会阻止 setup 运行但仍生成选项类型**；**兼容性不满足时是 WARN 并把模块禁用**，不是启动失败；**用了 `.with()` 之后 TypeScript 才知道默认值把可选属性变成了必有值**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 模块选项
  - 默认值合并
  - 生命周期钩子
  - TypeScript
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模块选项这件事上最容易做错的一步，是**在 `setup` 里手动判断"用户有没有传这个值"**。合并已经由 `defineNuxtModule` 做完了。这一篇把合并规则、参数表、以及三个会改变行为状态的写法逐条落地。

## 定义句就是答案

Kit 页对 `defineNuxtModule` 的一句定性：**定义一个 Nuxt 模块，自动把默认值与用户提供的选项合并，安装提供的钩子，并调用一个可选的 setup 函数以获得完全控制**。

三个动作里第一个就是合并。**它的方向在后面 `moduleDependencies` 的措辞里被写得更细**：`defaults` 是"应用**在 `nuxt.options` 之下**的配置，用户配置优先于它"；`overrides` 是"应用**在 `nuxt.options` 之上**、优先于用户配置"。同一族的两个词在自身模块上就是：用户写的赢你的 `defaults`。

## 参数表八行

`definition` 可以是模块定义对象，也可以是模块函数。对象里能出现的键，**八个全部标注为非必填**：

| 属性 | 类型 | 必填 | 文档描述 |
| --- | --- | --- | --- |
| `meta` | `ModuleMeta` | `false` | 模块元数据，定义模块的名称、版本、config key 与兼容性 |
| `defaults` | `T` 或 `(nuxt: Nuxt) => T` | `false` | 模块默认选项；**给函数时以 Nuxt 实例作为第一个参数调用** |
| `schema` | `T` | `false` | 模块选项的模式；提供时选项会应用到该模式上 |
| `hooks` | `Partial<NuxtHooks>` | `false` | 要为模块安装的钩子；提供时模块会安装它们 |
| `moduleDependencies` | `Record<string, ModuleDependency>` 或接收 Nuxt 实例的函数 | `false` | 对其他模块的依赖，带版本约束与配置 |
| `onInstall` | `(nuxt) => Awaitable<void>` | `false` | 模块首次安装时调用的生命周期钩子，**要求 `meta.name` 与 `meta.version` 都已定义** |
| `onUpgrade` | `(nuxt, options, previousVersion) => Awaitable<void>` | `false` | 模块升级到新版本时调用的生命周期钩子，同样要求 `meta.name` 与 `meta.version` |
| `setup` | `(this: void, resolvedOptions, nuxt) => Awaitable<void \| false \| ModuleSetupInstallResult>` | `false` | 模块的 setup 函数；提供时模块会调用它 |

两处值得停一下。

**`defaults` 可以是函数**，形如 `(nuxt) => T`。需要按当前配置算默认值时用它，而不是在 `setup` 里补。

**`setup` 的返回值类型里有 `false`**。文档给了这个类型但**没有解释返回 `false` 的含义**，这一点要标"文档未说明"。

## configKey 决定用户往哪写

`meta.configKey` 的作用在 Kit 页有个完整来回：

```ts
export default defineNuxtModule({
  meta: {
    name: 'my-module',
    configKey: 'myModule',
  },
  defaults: {
    enabled: true,
  },
  setup (options) {
    if (options.enabled) {
      console.log('My Nuxt module is enabled!')
    }
  },
})
```

用户在 `nuxt.config` 里对应的键下面给选项：

```ts
export default defineNuxtConfig({
  myModule: {
    enabled: false,
  },
})
```

第三种状态是把键整个设成 `false`，文档这一句同时给了**副作用边界**：

> Users can also completely disable a module by setting the config key to `false`. This prevents the module's setup function from running while still generating types for module options.

**setup 不跑，但模块选项的类型照样生成**。`::tip` 补了用途：这在要禁用从**分层**继承来的模块时特别有用。

三态对照：

| 用户写法 | 结果 |
| --- | --- |
| 不写这个键 | 只有 `defaults`，setup 正常跑 |
| 写成对象 | 用户的值覆盖 `defaults` 里同名的部分，setup 正常跑 |
| 写成 `false` | **setup 不跑**，选项类型仍然生成 |

## 兼容性不满足时不是报错

`compatibility.nuxt` 用 semver 写，文档示例里 `>=3.0.0` 与 `^3.0.0` 两种都出现过。不匹配时的表现给的是**控制台警告**，文案是：

```terminal
 WARN  Module @nuxt/icon is disabled due to incompatibility issues:
 - [nuxt] Nuxt version ^3.1.0 is required but currently using 3.0.0
```

注意那句话的动词：**is disabled**。也就是说这不是构建失败，而是**模块被静默跳过、只留一条 WARN**。排查"我的模块怎么完全没反应"时，这条警告是第一个要看的地方；而 **这一页没有说被禁用时 `defaults` 与类型是否还参与生成**，不能假定。

## `.with()` 解决的是类型，不是运行时

需要合并后的选项有准确类型时用 `.with()`：

```ts
interface ModuleOptions {
  apiKey: string
  baseURL: string
  timeout?: number
  retries?: number
}

export default defineNuxtModule<ModuleOptions>().with({
  meta: {
    name: '@nuxtjs/my-api',
    configKey: 'myApi',
  },
  defaults: {
    baseURL: 'https://api.example.com',
    timeout: 5000,
    retries: 3,
  },
  setup (resolvedOptions, nuxt) {
    console.log(resolvedOptions.baseURL) // 一定有值
    console.log(resolvedOptions.timeout) // 一定有值
    console.log(resolvedOptions.retries) // 一定有值
  },
})
```

文档把差别说得很直接：**不用 `.with()` 时，`resolvedOptions` 会被标成原始的 `ModuleOptions` 接口**，于是 `timeout` 和 `retries` **即使你给了默认值也仍然是"可能是 undefined"**。`.with()` 让 TypeScript 理解"默认值把这些属性变成非可选"。

推论：**运行时一直是有默认值的，缺的只是类型**。所以如果你发现运行正常但 TS 在报"possibly undefined"，先检查有没有 `.with()`，而不是加 `??` 兜底。

## 两个生命周期钩子的四条硬规则

`onInstall` 与 `onUpgrade` 的规则集中在一段里：

- **必须同时提供 `meta.name` 和 `meta.version`**，因为钩子用这两个值在项目的 **`.nuxtrc` 文件**里追踪模块的安装状态。
- **两个钩子都跑在主 `setup` 之前**。
- **钩子抛错时会被记录，但不会中断构建**。
- `onInstall` **只在模块首次被加进项目时跑一次**；`onUpgrade` 在版本号每次提升时跑（按 semver 比较），但**每个版本只跑一次**。

示例里的用法是按旧版本分支：

```ts
onUpgrade (nuxt, options, previousVersion) {
  if (isLess(previousVersion, '1.1.0')) {
    console.log('Breaking changes in 1.1.0 - please check the migration guide')
  }
}
```

第三条"抛错不中断构建"是有代价的：**一次性初始化失败时，构建仍然会成功**，只是配置没生成。这类问题不会以红字出现，得看日志。

## `installModule` 还在文档里，但已被废弃

```ts
async function installModule (moduleToInstall: string | NuxtModule, inlineOptions?: any, nuxt?: Nuxt)
```

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `moduleToInstall` | `string \| NuxtModule` | `true` | 要安装的模块，可以是模块名字符串，也可以是模块对象本身 |
| `inlineOptions` | `any` | `false` | 传给该模块 `setup` 的选项对象 |
| `nuxt` | `Nuxt` | `false` | Nuxt 实例；不提供时通过 `useNuxt()` 从上下文取 |

废弃说明的原话是：改用 `defineNuxtModule` 里的 `moduleDependencies` 选项，**`installModule` 未来会被移除（或可能变成非阻塞）**。整页唯一还写它价值的地方是那个 `nuxt` 参数——`moduleDependencies` 无法把一个模块实例本身作为依赖目标来装。

依赖还能写成函数，按当前 Nuxt 配置动态决定：

```ts
moduleDependencies (nuxt) {
  const dependencies: Record<string, any> = {
    '@nuxtjs/tailwindcss': { version: '>=6.0.0' },
  }
  if (nuxt.options.experimental?.someFeature) {
    dependencies['@nuxtjs/fontaine'] = { optional: true }
  }
  return dependencies
}
```

合并规则说完了，下一篇是往项目里塞东西：九节基础配方各用哪个函数。

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/modules
- https://nuxt.com/docs/4.x/guide/modules/module-anatomy
