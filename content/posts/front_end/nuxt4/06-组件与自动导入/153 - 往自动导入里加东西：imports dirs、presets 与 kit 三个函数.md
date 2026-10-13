---
url: /posts/981761cd98dfa9f5d2514b736d9b7f43/
title: 153 - 往自动导入里加东西：imports dirs、presets 与 kit 三个函数
date: 2026-12-02T09:00:00+08:00
updated: 2026-12-02T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：152 讲怎么关，这一篇讲怎么加，官方给了三条入口。**配置文件**那一侧：`imports.dirs`——"**An array of custom directories that will be auto-imported. Note that this option will not override the default directories (`~/composables`, `~/utils`).**"（官方示例 `dirs: ['stores']`，注释是 "Auto-import pinia stores defined in `~/stores`"）；`imports.presets` 只出现在 guide 的示例里——`presets: [{ from: 'vue-i18n', imports: ['useI18n'] }]`，配一句省事的 tip："**If you are using the Nuxt module for that package, it is likely that the module has already configured auto-imports for that package.**"。**kit 三个函数**：`addImports`（"Add imports to the Nuxt application. It makes your imports available in the Nuxt app context **without the need to import them manually**."，参数八个字段，其中 `priority` 是"**if multiple imports have the same name, the one with the highest priority will be used**"、`type` 是"If this import is a pure type import."、`typeFrom` 是"Use this as the `from` value when generating type declarations."）、`addImportsDir`（"Add imports from a directory to the Nuxt application. It will automatically import **all files from the directory**..."，`options.prepend` 为 true 时"the imports will be prepended to the list of imports"）、`addImportsSources`（"Add listed imports to the Nuxt application."，两种条目形态：`NuxtImportPreset` 的 `from`+`imports` 都必需、`NuxtPackageImportPreset` 只有必需的 `package`）。官方说明这层机制的底座："**These utilities are powered by `unimport`, which provides the underlying auto-import mechanism used in Nuxt.**"，并划清职责："These functions are designed for registering your own utils, composables and Vue APIs. For pages, components and plugins, please refer to the specific sections: Pages, Components, Plugins."。**构建期钩子**那四个：`imports:sources`（参数 `presets`，"Called at setup allowing modules to extend sources."）、`imports:extend`（参数 `imports`，"Called at setup allowing modules to extend imports."）、`imports:context`（参数 `context`，"Called when the [unimport] context is created."）、`imports:dirs`（参数 `dirs`，"Allows extending import directories."）。服务端一侧对应的是 `addServerImports` 与 `addServerImportsDir`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 自动导入
  - kit
  - unimport
  - 钩子

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


自动导入的名单不是框架写死的。把自己写的工具、某个包的几个导出、甚至一整个目录挂进去，官方给了三条入口：配置文件、kit 函数、构建期钩子。这一篇把三条的字段与差别摆齐。

## 三条入口的位置

| 入口 | 谁在用 | 典型场合 |
| --- | --- | --- |
| `nuxt.config` 的 `imports.dirs` / `imports.presets` | 应用作者 | 让自己的某个目录、某个包的几个名字免写 import |
| kit 的 `addImports` / `addImportsDir` / `addImportsSources` | 模块（含本地模块） | 发布可复用的组合式函数、带一套 runtime 目录 |
| `imports:sources` / `imports:extend` / `imports:dirs` / `imports:context` 钩子 | 模块需要在 setup 期改名单时 | 与上面同族，直接操作那几份数据 |

kit 页开篇把范围说得干净：

> Nuxt Kit provides a set of utilities to help you work with auto-imports. These functions allow you to register your own utils, composables and Vue APIs.

以及一句"这不是万能注册器"：

> These functions are designed for registering your own utils, composables and Vue APIs. For pages, components and plugins, please refer to the specific sections: Pages, Components, Plugins.

**页面、组件、插件各有自己的 kit 节**（组件那三个函数我在 160 讲）。这一层的底座也写明了：

> These utilities are powered by `unimport`, which provides the underlying auto-import mechanism used in Nuxt.

## 入口一：`imports.dirs`

官方定义与那条"不是替换"：

> An array of custom directories that will be auto-imported. **Note that this option will not override the default directories (`~/composables`, `~/utils`).**

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  imports: {
    // Auto-import pinia stores defined in `~/stores`
    dirs: ['stores'],
  },
})
```

两件事连着看：

- **这是加法**：`dirs: ['stores']` 之后 `~/composables`、`~/utils` 仍然被扫（152 那条"换不掉默认目录"是同一句原文）。
- 151 里那个 `shared/` 的坑要在这里补一次：想让 `shared/utils/` 的下层子目录进入自动导入，**得同时加 `imports.dirs` 与 `nitro.imports.dirs`**——app 侧与服务端侧的名单是两套注册。

## 入口二：`imports.presets`

guide 那一节给的形状只有一个，但足够用：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  imports: {
    presets: [
      {
        from: 'vue-i18n',
        imports: ['useI18n'],
      },
    ],
  },
})
```

前置那句 tip 值得先执行再配置：

> If you are using the Nuxt module for that package, it is likely that the module has already configured auto-imports for that package.

**一条容易记错的记载**：`api/nuxt-config` 那一节的 `imports` 节**只列了 `dirs`、`global`、`scan` 三个子项，没有 `presets` 的独立条目**——`presets` 这个键只在 guide 的示例里出现（kit 侧则有对应的 `NuxtImportPreset` 类型）。所以写 `presets` 时，我能依据的形状是示例本身（`from` + `imports` 数组），而不是配置参考页里的一行定义。这类"文档两处不对称"我记在正文里，是为了让读到这里的你不用再去猜是不是自己漏看了。

## 入口三：kit 的三个函数

### `addImports`：按名字注册

> Add imports to the Nuxt application. It makes your imports available in the Nuxt app context **without the need to import them manually**.

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

类型是 `function addImports (imports: NuxtImport | NuxtImport[]): void`，单个对象的八个字段：

| 字段 | 必需 | 官方定义 |
| --- | --- | --- |
| `name` | 是 | "Import name to be detected." |
| `from` | 是 | "Module specifier to import from." |
| `priority` | 否 | "Priority of the import; **if multiple imports have the same name, the one with the highest priority will be used.**" |
| `disabled` | 否 | "If this import is disabled." |
| `meta` | 否 | "Metadata of the import." |
| `type` | 否 | "If this import is a pure type import." |
| `typeFrom` | 否 | "Use this as the `from` value when generating type declarations." |
| `as` | 否 | "Import as this name." |

三个容易漏的：

- **`priority` 是撞名裁决**（151 那张表里同一个字段），不是"加载顺序"。
- **`type` 与 `typeFrom`** 是给纯类型用的：`type: true` 表示这是纯类型导入，`typeFrom` 让生成的类型声明从另一个来源取。
- **示例为什么把 `as` 也写成同名**：官方那段 `names.forEach(name => addImports({ name, as: name, from: '@storyblok/vue' }))` 里 `as` 与 `name` 相同，这是"保持原名"的显式写法；真正要用别名时，`as` 才是模板里那个名字。

这一节还挂着一条 tip 指回跨侧问题：

> To add imports for the Nitro server context, refer to the `addServerImports` function.

### `addImportsDir`：按目录注册

> Add imports from a directory to the Nuxt application. It will automatically import **all files from the directory** and make them available in the Nuxt application without the need to import them manually.

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

类型 `function addImportsDir (dirs: string | string[], options?: { prepend?: boolean }): void`：

| 字段 | 必需 | 官方定义 |
| --- | --- | --- |
| `dirs` | 是 | "A string or an array of strings with the path to the directory to import from." |
| `options` | 否 | "Options to pass to the import. **If `prepend` is set to `true`, the imports will be prepended to the list of imports.**" |

**注意官方措辞是"all files from the directory"**：这一层是"目录整包进名单"，而 `imports.dirs` 是"配置里加一个自定义目录"——两者是同一件事的两种入口，模块里用前者（要配合 `createResolver` 解析相对路径），应用配置里用后者。

### `addImportsSources`：把某个包的导出列进来

> Add listed imports to the Nuxt application.

官方示例同时给了**两种条目形态**：

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

类型 `function addImportsSources (importSources: NuxtImportPresetSource | NuxtImportPresetSource[]): void`，那个联合的两种成员各自的表：

| 形态 | 字段 | 必需 | 官方定义 |
| --- | --- | --- | --- |
| `NuxtImportPreset` | `from` | 是 | "Module specifier to import from." |
| | `imports` | 是 | "An array of entries, which can be import names, import objects or nested presets." |
| `NuxtPackageImportPreset` | `package` | 是 | "Name of the package." |

**两种形态的分工从字段就能读出来**：`package` 一个字符串就把整包导出交给自动导入（示例里的 `@vueuse/core`），`from` + `imports` 则**点名要哪几个**（示例里从 `h3` 只取五个）。后者更可控——这也正是 `imports.presets` 示例用的形状（`from` + `imports`）。

`imports` 那句"can be import names, import objects or nested presets"意味着这个数组可以混着写：字符串、对象、甚至再套一层 preset。

## 构建期那四个钩子

官方钩子表里与自动导入相关的四行（都在"Called at setup"这一族）：

| Hook | Arguments | Description（原文） |
| --- | --- | --- |
| `imports:sources` | `presets` | Called at setup allowing modules to extend sources. |
| `imports:extend` | `imports` | Called at setup allowing modules to extend imports. |
| `imports:context` | `context` | Called when the unimport context is created. |
| `imports:dirs` | `dirs` | Allows extending import directories. |

按参数名读，前三行与三个 kit 函数几乎一一对应：`presets`↔`addImportsSources`、`imports`↔`addImports`、`dirs`↔`addImportsDir`；`imports:context` 是唯一直接暴露 unimport 实例的那个。**这是我按参数名做的对照，官方没有给这张表**——真要写模块，我建议用三个函数而不是钩子：函数是公开 API，钩子参数形状属于框架内部约定。

## 服务端一侧

151 已经引过两条定位句，这里只放并列关系：

| 侧 | 函数 | 官方定义 |
| --- | --- | --- |
| app | `addImports` / `addImportsDir` / `addImportsSources` | 见上 |
| Nitro | `addServerImports` / `addServerImportsDir` | "Add imports to the server. It makes your imports available in Nitro without the need to import them manually." / "Add a directory to be scanned for auto-imports by Nitro." |

**两侧字段几乎同名**（`addServerImports` 的表里同样有 `priority`、`disabled`、`meta`、`type`、`typeFrom`、`as`），但那是两套注册表。所以 151 那条硬性要求仍然适用：跨侧共用的工具"must be imported from the same source file for both `addImports` and `addServerImports` and should have identical signature"，且那个源文件不得引用上下文相关代码。

## 我该用哪一条

| 需求 | 用 |
| --- | --- |
| 我的 `~/stores` 之类目录免写 import | `imports.dirs`（配置，最省事） |
| 只想暴露某包的三五个导出 | `imports.presets` 的 `from` + `imports` |
| 整包导出都放开 | `addImportsSources({ package: '...' })`（模块里） |
| 我在写模块，runtime 里有一整个 composables 目录 | `addImportsDir(resolver.resolve('./runtime/composables'))` |
| 单个工具精确注册、还要控制撞名优先级或别名 | `addImports({ name, as, from, priority })` |
| 服务端也要能同样用 | `addServerImports` / `addServerImportsDir`，跨侧共用时同源同签名 |
| 只是想让某个页面用一次某个组件 | 别碰自动导入，用 `#components` 显式导入（161） |

## 自查

1. `imports.dirs` 会不会替换 `~/composables` 与 `~/utils`？官方哪句话回答了这个？
2. `imports.presets` 的示例形状是什么？`api/nuxt-config` 的 `imports` 节里有没有它的独立条目？
3. kit 三个函数各自定位句里的关键词是什么（按名字 / 按目录整包 / 按列出的导出）？
4. `addImports` 的 `priority`、`type`、`typeFrom`、`as` 分别做什么？
5. `addImportsDir` 的 `prepend: true` 改的是什么？官方说它会导入目录里的什么文件？
6. `addImportsSources` 的两种条目形态分别需要哪些字段？`imports` 数组里能混什么？
7. 四个 `imports:*` 钩子分别暴露什么参数？其中哪个直接给出 unimport 实例？
8. 跨侧共用的工具要满足哪四条？（回到 151 那句原文）
9. 想让服务端也扫描一个新目录，用哪个函数？

## 下一篇

154 讲 `#imports` 这个别名：哪三种场合官方明确要写显式导入、模块与服务端代码里的类型边界，以及自动导入名单与 `#imports` 的关系。

## 本文依据

- `https://nuxt.com/docs/4.x/api/kit/autoimports`
  - 原文（开篇与定位）："Nuxt Kit provides a set of utilities to help you work with auto-imports. These functions allow you to register your own utils, composables and Vue APIs."；"Nuxt auto-imports helper functions, composables and Vue APIs to use across your application without explicitly importing them. Based on the directory structure, every Nuxt application can also use auto-imports for its own composables and plugins."；"With Nuxt Kit you can also add your own auto-imports. `addImports` and `addImportsDir` allow you to add imports to the Nuxt application. `addImportsSources` allows you to add listed imports from 3rd party packages to the Nuxt application."；"These utilities are powered by `unimport`, which provides the underlying auto-import mechanism used in Nuxt."；note "These functions are designed for registering your own utils, composables and Vue APIs. For pages, components and plugins, please refer to the specific sections: Pages, Components, Plugins."
  - 原文（`addImports`）："Add imports to the Nuxt application. It makes your imports available in the Nuxt app context without the need to import them manually."；tip "To add imports for the Nitro server context, refer to the `addServerImports` function."；示例 `names.forEach(name => addImports({ name, as: name, from: '@storyblok/vue' }))`；类型 `function addImports (imports: NuxtImport | NuxtImport[]): void`；参数表八字段：`name` — "Import name to be detected."、`from` — "Module specifier to import from."、`priority` — "Priority of the import; if multiple imports have the same name, the one with the highest priority will be used."、`disabled` — "If this import is disabled."、`meta` — "Metadata of the import."、`type` — "If this import is a pure type import."、`typeFrom` — "Use this as the `from` value when generating type declarations."、`as` — "Import as this name."
  - 原文（`addImportsDir`）："Add imports from a directory to the Nuxt application. It will automatically import all files from the directory and make them available in the Nuxt application without the need to import them manually."；示例 `meta: { name: '@vueuse/motion', configKey: 'motion' }` + `addImportsDir(resolver.resolve('./runtime/composables'))`；类型 `function addImportsDir (dirs: string | string[], options?: { prepend?: boolean }): void`；`dirs` — "A string or an array of strings with the path to the directory to import from."；`options` — "Options to pass to the import. If `prepend` is set to `true`, the imports will be prepended to the list of imports."
  - 原文（`addImportsSources`）："Add listed imports to the Nuxt application."；示例 `addImportsSources([{ package: '@vueuse/core' }, { from: 'h3', imports: ['defineEventHandler', 'getQuery', 'getRouterParams', 'readBody', 'sendRedirect'] }])`；类型 `function addImportsSources (importSources: NuxtImportPresetSource | NuxtImportPresetSource[]): void`；`NuxtImportPreset`：`from`（required，"Module specifier to import from."）、`imports`（required，"An array of entries, which can be import names, import objects or nested presets."）；`NuxtPackageImportPreset`：`package`（required，"Name of the package."）
  - 原文（Source）：`packages/kit/src/imports.ts`
- `https://nuxt.com/docs/4.x/api/nuxt-config`
  - 原文（`imports` 节）："Configure how Nuxt auto-imports composables into your application."；`dirs` — "An array of custom directories that will be auto-imported. Note that this option will not override the default directories (`~/composables`, `~/utils`)." + 示例 `dirs: ['stores']` 与注释 "Auto-import pinia stores defined in `~/stores`"；`global` — 仅 Type `boolean`、Default `false`；`scan` — "Whether to scan your `app/composables/` and `app/utils/` directories for composables to auto-import. Auto-imports registered by Nuxt or other modules, such as imports from `vue` or `nuxt`, will still be enabled."
  - 事实核对：那一节 `imports` 节只有 `dirs`、`global`、`scan` 三个子项，**没有 `presets` 条目**
- `https://nuxt.com/docs/4.x/guide/concepts/auto-imports`
  - 原文（第三方包）："Nuxt also allows auto-importing from third-party packages."；tip "If you are using the Nuxt module for that package, it is likely that the module has already configured auto-imports for that package."；示例 `imports: { presets: [{ from: 'vue-i18n', imports: ['useI18n'] }] }`；note "You can also auto-import functions exported from custom folders or third-party packages by configuring the `imports` section of your `nuxt.config` file."
- `https://nuxt.com/docs/4.x/api/advanced/hooks`
  - 原文（Nuxt Hooks 表四行）："`imports:sources` | `presets` | Called at setup allowing modules to extend sources."；"`imports:extend` | `imports` | Called at setup allowing modules to extend imports."；"`imports:context` | `context` | Called when the unimport context is created."；"`imports:dirs` | `dirs` | Allows extending import directories."
- `https://nuxt.com/docs/4.x/api/kit/nitro`
  - 原文（服务端两个函数）：`addServerImports` — "Add imports to the server. It makes your imports available in Nitro without the need to import them manually." 及其 warning "the function must be imported from the same source file for both `addImports` and `addServerImports` and should have identical signature"；`addServerImportsDir` — "Add a directory to be scanned for auto-imports by Nitro."
- `https://nuxt.com/docs/4.x/directory-structure/shared`
  - 对照用（见 151）：子目录要同时加进 `imports.dirs` 与 `nitro.imports.dirs`
