---
url: /posts/85737d134124550978edc151ef2972e2/
title: 407 - 钩子与模块类型：NuxtConfig 与自定义字段
date: 2027-02-24T11:00:00+08:00
updated: 2027-02-24T11:00:00+08:00
author: cmdragon

summary:
  摘要：模块作者关心的"类型"有两块：模块选项怎么变成 `nuxt.config` 里一个有提示的键，以及 `setup` 里那个 `nuxt` 对象凭什么有类型。官方给的答案都收在 `defineNuxtModule` 上：低层写法是 `(inlineOptions, nuxt) => {}` 的异步函数（能读 `nuxt.options.dev`、调 `nuxt.hook('ready', ...)`），但官方明说**不推荐**低层函数写法、推荐带 `meta` 的对象语法，尤其要发布到 npm 时。`meta.configKey` 的定义就是"The key in nuxt.config that holds your module options"——这一个键让自定义选项进得了 `nuxt.config`。`defineNuxtModule` 返回的包装函数官方列了八条行为，其中"type hints and automated type inference"就是类型来源，另有 `meta.name`/`configKey` 算唯一键保证只装一次、自动检查兼容性。

categories:
  - 前端开发

tags:
  - nuxt4
  - defineNuxtModule
  - configKey
  - 模块作者
  - 类型推断
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

写模块的人会遇到两个类型问题：用户在我模块里配的选项，怎么在 `nuxt.config` 有补全；`setup(options, nuxt)` 里的 `nuxt` 凭什么有完整类型。官方在 module-anatomy 那一页把两件事都挂在 `defineNuxtModule` 上。

## 两种写法，官方只推荐一种

低层写法（官方给了示例，但明确不推荐）：

```ts
export default function (inlineOptions, nuxt) {
  console.log(inlineOptions.token) // `123`
  console.log(nuxt.options.dev) // `true` or `false`
  nuxt.hook('ready', (nuxt) => {
    console.log('Nuxt is ready')
  })
}
```

官方的定性是"a Nuxt module definition is a simple, potentially asynchronous, function accepting inline user options and a `nuxt` object to interact with Nuxt"——它没有类型辅助。想要提示，就得换成 `defineNuxtModule`（官方原话："You can get type hinting for this function using the higher-level `defineNuxtModule` helper provided by Nuxt Kit."）。

然后是最明确的一句：

> However, **we do not recommend** using this low-level function definition. Instead, to define a module, **we recommend** using the object-syntax with `meta` property to identify your module, especially when publishing to npm.

## 对象语法：自定义字段进 nuxt.config 的位置

```ts
import { defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  meta: {
    // Usually the npm package name of your module
    name: '@nuxtjs/example',
    // The key in `nuxt.config` that holds your module options
    configKey: 'sample',
    // Compatibility constraints
    compatibility: {
      // Semver version of supported nuxt versions
      nuxt: '>=3.0.0',
    },
  },
  defaults: {},
  hooks: {},
  moduleDependencies: {},
  setup (moduleOptions, nuxt) {
    // ...
  },
})
```

`meta` 三个字段的官方注释各自回答了三个问题：

| 字段 | 官方注释 | 作用 |
| --- | --- | --- |
| `meta.name` | "Usually the npm package name of your module" | 标识模块；参与唯一键计算 |
| `meta.configKey` | "The key in `nuxt.config` that holds your module options" | **模块选项在 `nuxt.config` 里的那个键名** |
| `meta.compatibility.nuxt` | "Semver version of supported nuxt versions" | 支持的 Nuxt 版本约束 |

`configKey` 就是"自定义字段进 nuxt.config"的官方机制：定了 `configKey: 'sample'`，用户就能在 `nuxt.config` 里写 `sample: { ... }` 给模块传选项。

其余顶层字段（官方注释原样）：`defaults` 是"Default configuration options for your module, can also be a function returning those"；`hooks` 是"Shorthand sugar to register Nuxt hooks"（构建期钩子的简写注册，与 294/360 那批钩子同一套名字）；`setup` 是"The function holding your module logic, it can be asynchronous"。

## 包装函数做了什么（官方八条）

`defineNuxtModule` 返回的还是那个低层 `(inlineOptions, nuxt)` 签名的函数，只是先做完默认值等步骤再调你的 `setup`。官方列的八条里，跟类型直接相关的是第二条：

- 支持 `defaults` 与 `meta.configKey` 自动合并模块选项
- **Type hints and automated type inference**
- 用从 `meta.name` 或 `meta.configKey` 算出的唯一键保证模块只被安装一次
- 自动注册 Nuxt 钩子
- 基于模块 meta 自动检查兼容性问题
- 对内暴露 `getOptions` 与 `getMeta`
- 只要用最新版 `@nuxt/kit` 的 `defineNuxtModule` 就保证向后向上兼容
- 与模块构建工具集成

第二条"automated type inference"正是选项类型从哪来的答案：`setup(moduleOptions, ...)` 里的 `moduleOptions` 由 `defaults` 与传入选项推断出来，不用你手写接口。

## `moduleDependencies`：改别人的配置，但不保证顺序

这个字段容易被误读成功能性的"依赖排序"，官方注释先打了预防针：

> Configuration for other modules - this does not ensure the module runs before your module, but it allows you to change the other module's configuration before it runs.

四个子字段（官方描述逐条）：

| 字段 | 官方描述 |
| --- | --- |
| `version` | "You can specify a version constraint for the module. If the user has a different version installed, Nuxt will throw an error on startup." |
| `optional` | "By default moduleDependencies will be added to the list of modules to be installed by Nuxt unless `optional` is set." |
| `overrides` | "Any configuration that should override `nuxt.options`." |
| `defaults` | "Any configuration that should be set. It will override module defaults but will not override any configuration set in `nuxt.options`." |

`overrides` 与 `defaults` 的差别就在优先级：一个压过 `nuxt.options`，一个只压模块默认值、压不过用户在 `nuxt.options` 里写的。版本不符是**启动时抛错**，不是警告。

## 模块侧的一条类型边界

模块的 runtime 目录里不能用自动导入（`node_modules` 内出于性能不启用），必须显式从 `#imports` 引——官方在这一节用 `::warning` 写了这条（404 已展开）。这一条对模块作者的类型影响最实际：在你项目里"免费"的那些名字，进了 `src/runtime/` 就得自己引。

## 自查

- 低层 `(inlineOptions, nuxt)` 函数官方不推荐；要类型提示就用 `defineNuxtModule`，发布到 npm 更要用带 `meta` 的对象语法。
- 模块选项进 `nuxt.config` 靠 `meta.configKey`（官方注释"The key in `nuxt.config` that holds your module options"）。
- 选项类型来自 `defineNuxtModule` 的 automated type inference（`defaults` 加传入值），不用手写接口。
- `moduleDependencies` 不保证执行顺序，只能提前改对方配置；`version` 不符启动抛错；`overrides` 压 `nuxt.options`、`defaults` 不压。
- 模块 runtime 里必须显式 `from '#imports'`（`node_modules` 不启用自动导入）。

## 下一篇

下一篇 408 处理"配置与类型怎么都不对"的最后一步：`nuxt cleanup` 删掉哪五样、再 `nuxt prepare` 重来，以及为什么手改 `.nuxt` 必然被覆盖。
