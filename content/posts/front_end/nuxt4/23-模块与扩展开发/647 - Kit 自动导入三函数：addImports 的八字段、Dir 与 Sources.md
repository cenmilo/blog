---
url: /posts/f66b9c31fdb5ba094c5b2c637fc732de/
title: 647 - Kit 自动导入三函数：addImports 的八字段、Dir 与 Sources
date: 2027-05-15T11:00:00+08:00
updated: 2027-05-15T11:00:00+08:00
author: cmdragon

summary:
  摘要：让宿主的代码里不写 import 就能用你提供的函数，靠这一节的三个工具，**它们的分工是"条目、目录、第三方包"三种粒度**：`addImports` 逐个登记（八个字段里只有 `name` 与 `from` 必填，**同名时 `priority` 高的那个胜出**）；`addImportsDir` 把一个目录整个扫进来，**可选 `prepend` 决定插到列表前面还是后面**；`addImportsSources` 登记第三方包，两种形状——**给 `package` 就是整包，给 `from` ＋ `imports` 就是白名单**。三个函数**都由 unimport 提供底层机制**，而且**都只作用于应用侧上下文：服务端要换成 `addServerImports`**。页面还有一条 note：这三个是给 utils、composables 与 Vue API 用的，**页面、组件、插件各有专门的章节**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 自动导入
  - NuxtKit
  - unimport
  - 模块注入
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

自动导入是 Nuxt 让人少写 import 的那个机制。作为模块作者，你也可以往里加东西——但**加一条、加一个目录、加一个第三方包，是三件不同的事**，各有一个函数。

## 机制与适用面

页面先复述了机制本身：**Nuxt 会自动导入 helper 函数、组合式函数与 Vue API，让你在应用里使用它们时不必显式 import；基于目录结构，每个 Nuxt 应用还会为它自己的 composables 与 plugins 生成自动导入**。

然后一句定性把三个函数分派清楚：**用 `addImports` 与 `addImportsDir` 可以给 Nuxt 应用添加导入；`addImportsSources` 允许你把第三方包里列出来的那些导入加进应用**。

底层是谁也写了：**这些工具由 `unimport` 驱动，它提供 Nuxt 里使用的自动导入机制**。

一条 `::note` 直接划走了三个常见误会：

> These functions are designed for registering your own utils, composables and Vue APIs. For pages, components and plugins, please refer to the specific sections: Pages, Components, Plugins.

**页面、组件、插件不在这一节**——它们各有专门章节，硬用 `addImports` 去登记一个组件名，不会得到组件注册的那套行为。

## 三个函数怎么选

| 场景 | 用哪个 | 关键差异 |
| --- | --- | --- |
| 只暴露两三个函数，来源是某个包 | `addImports` | 逐条给 `name`、`from`，可加别名 |
| runtime 目录下有一整个 composables 文件夹 | `addImportsDir` | **整个目录扫进来**，不用逐条列 |
| 想把第三方包的导出接进来 | `addImportsSources` | 可以整包，也可以只列白名单 |

## `addImports`：八个字段，两个必填

```ts
function addImports (imports: NuxtImport | NuxtImport[]): void
```

参数是**一个对象或对象数组**，字段八个：

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `name` | `string` | `true` | **要被识别的导入名** |
| `from` | `string` | `true` | 从哪个模块 specifier 导入 |
| `priority` | `number` | `false` | 导入的优先级；**多个导入同名时，优先级最高的那个被使用** |
| `disabled` | `boolean` | `false` | 这条导入是否被禁用 |
| `meta` | `Record<string, any>` | `false` | 这条导入的元数据 |
| `type` | `boolean` | `false` | **这条导入是否是纯类型导入** |
| `typeFrom` | `string` | `false` | **生成类型声明时用作 `from` 的值** |
| `as` | `string` | `false` | 以这个名字导入（对外暴露的名字） |

三个字段的用途要分开记：

**`priority` 是冲突解决开关**。自动导入里出现两个同名是常态（用户自己也有一个 `useAuth`），文档给的规则只有"最高的那个用"，**没有说默认值是多少**。

**`type` 与 `typeFrom` 是一对**：前者标"这是纯类型"，后者在生成类型声明时替换 `from` 的值。**页面没有说 `typeFrom` 在不设 `type` 时的行为**。

**`as` 决定用户在代码里敲什么名字**，`name` 决定源文件里导出的名字。官方示例里两者写成同一个值：

```ts
import { addImports, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  setup (options, nuxt) {
    const names = [
      'useStoryblok',
      'useStoryblokApi',
      'useStoryblokBridge',
      'renderRichText',
      'RichTextSchema',
    ]

    names.forEach(name =>
      addImports({ name, as: name, from: '@storyblok/vue' }),
    )
  },
})
```

这个例子里能看出两件事：**数组写法是 `forEach` 逐个调**（参数本身支持数组，示例没这么写），**最后一项 `RichTextSchema` 其实是个类型而不是 composable**——示例里没给它加 `type: true`。

`addImports` 上面挂了一条 `::tip`，是这一节最容易踩空的地方：

> To add imports for the Nitro server context, refer to the `addServerImports` function.

**要往服务端上下文加自动导入，得换成 `addServerImports`（它在 nitro 那一节）**，`addImports` 只管应用侧。

## `addImportsDir`：整目录，带一个方向选项

```ts
function addImportsDir (dirs: string | string[], options?: { prepend?: boolean }): void
```

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `dirs` | `string \| string[]` | `true` | 要从中导入的目录路径，字符串或数组 |
| `options` | `{ prepend?: boolean }` | `false` | **`prepend` 为 `true` 时，这些导入会被加到导入列表前面** |

官方示例就是模块 runtime 的标准写法：

```ts
import { addImportsDir, createResolver, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  meta: {
    name: '@vueuse/motion',
    configKey: 'motion',
  },
  setup (options, nuxt) {
    const resolver = createResolver(import.meta.url)
    addImportsDir(resolver.resolve('./runtime/composables'))
  },
})
```

**`resolver.resolve` 的作用在这里体现得很清楚：路径是相对当前模块文件算的**，直接写 `'./runtime/composables'` 会以项目根为基准而指错。

`prepend` 这一项和 `priority` 不是一回事：前者说的是**加进列表的位置**，后者说的是**同名时谁赢**。页面没有把两者的关系写清。

## `addImportsSources`：两种形状

```ts
function addImportsSources (importSources: NuxtImportPresetSource | NuxtImportPresetSource[]): void
```

页面上把它写成两个类型的表（**注意这是"或"关系**）：

**`NuxtImportPreset`**

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `from` | `string` | `true` | 模块 specifier |
| `imports` | `(NuxtImportEntry \| NuxtImportPreset)[]` | `true` | 条目数组，**可以是导入名、导入对象，或者嵌套的 preset** |

**`NuxtPackageImportPreset`**

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `package` | `string` | `true` | 包名 |

两种形状在同一条调用里混着用，示例就是混的：

```ts
import { addImportsSources, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  setup () {
    addImportsSources([
      { package: '@vueuse/core' },
      {
        from: 'h3',
        imports: [
          'defineEventHandler',
          'getQuery',
          'getRouterParams',
          'readBody',
          'sendRedirect',
        ],
      },
    ])
  },
})
```

**`{ package: '@vueuse/core' }` 是一整包的预设，`{ from: 'h3', imports: [...] }` 是点名五个导出**。这一节没写 `package` 那种形状具体会带进来哪些名字（那是 unimport 预设的事），所以**用整包形状时最好先跑一次看看生成了什么**。

顺带一提，示例里那五个 h3 的名字（`defineEventHandler`、`getQuery`、`getRouterParams`、`readBody`、`sendRedirect`）在服务端兼容性那一章正是"两代 API 行为不同"的那几个——**把它们接进应用侧自动导入，等于把版本差异搬到了你眼前**。

## 三处误用

- **在 runtime 文件里 import 这三个函数**：Kit 只给模块用，运行时产物里没它。
- **想用 `addImports` 服务服务端代码**：换 `addServerImports`。
- **想注册组件或页面**：分别是 components 那一节与 pages 那一节的事，这一节的 note 已经点名。

## 与配置那一侧的关系

宿主的 `nuxt.config` 里有一个 `imports` 配置节（`autoImport`、`presets`、`global` 等键），**它管的是"要不要自动导入、预设从哪来"；这一节三个函数管的是"往列表里加哪一条"**。两边都能写 presets，页面没有说明同一份 preset 同时出现时的合并方向。

下一篇讲组件那两个函数：两张各 14 行的参数表。

## 下一篇

下一篇 648：Kit 组件两函数：两张 14 行参数表与那四个同名选项

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/autoimports
