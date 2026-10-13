---
url: /posts/5c2e55ba8262f9f5cc4b074bd341d3d3/
title: 576 - typescript：十一个子项与那份 17 项 hoist 默认表
date: 2027-04-22T09:00:00+08:00
updated: 2027-04-22T09:00:00+08:00
author: cmdragon

summary:
  摘要：`## typescript` 有定性句（"Configuration for Nuxt's TypeScript integration."）和**十一个 `###` 子项**，是配置参考里记载得比较满的一节，但三处不一致要当心：**`builder` 只有 `**Default:** null` 而没有类型行**；**`typeCheck` 的类型写 `boolean`，正文却让你设成 `build`**；四个 `*TsConfig` 里只有 `appTsConfig` 与 `serverTsConfig` 写了 "take precedence over `tsConfig`"，`nodeTsConfig` 与 `sharedTsConfig` 没写。`tsConfig` 那段最长的说明给出的规则是：`compilerOptions` 进全部四个生成文件，而 `include`、`exclude`、`vueCompilerOptions` **只进 app 那份**，`types`、`paths`、`noEmit` 由 Nuxt 按上下文自己管。

categories:
  - 前端开发

tags:
  - nuxt4
  - TypeScript
  - tsconfig
  - 类型检查
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`.nuxt/tsconfig.json` 是生成的，你改不了也不该改。要让 Nuxt 生成的类型配置按你的意思来，入口就是 `typescript` 这一节——它下面有十一个子项，分工比看起来细。

## 十一个子项与它们的记载完整度

| 子项 | Type | Default | 记载要点 |
| --- | --- | --- | --- |
| `appTsConfig` | 无 | 无 | 用来扩展生成的 `.nuxt/tsconfig.app.json`（以及 legacy `.nuxt/tsconfig.json`）；**这里设的选项优先于 `tsConfig`** |
| `builder` | **无类型行** | `null` | "Which builder types to include for your project."；默认按你的 `builder` 选项推断（**默认为 'vite'**）；可写 `false` 完全不纳入构建器环境类型、或选 `'shared'`；`'shared'` 官方建议给模块作者（要支持多种构建器时） |
| `hoist` | `array` | 17 项清单（见下） | 为这些模块在 `compilerOptions.paths` 里生成深度别名；**尚不支持子路径**；在 pnpm monorepo 且 `shamefully-hoist=false` 时可能是必需的 |
| `includeWorkspace` | `boolean` | `false` | 把父 workspace 纳入项目；官方写"mostly useful for themes and module authors" |
| `nodeTsConfig` | 无 | 无 | 扩展生成的 `.nuxt/tsconfig.node.json`；**没有写与 `tsConfig` 的优先关系** |
| `serverTsConfig` | 无 | 无 | 扩展 `.nuxt/tsconfig.server.json`；写了"优先于 `tsConfig`" |
| `sharedTsConfig` | 无 | 无 | 扩展 `.nuxt/tsconfig.shared.json`；**没有写优先关系** |
| `shim` | `boolean` | `false` | 生成 `*.vue` 的 shim；官方建议改让 Vue 官方扩展生成准确类型；并说"如果你用 ESLint 这类无法理解 `.vue` 文件类型的工具，可能想把它设成 `true`" |
| `strict` | `boolean` | `true` | 说明句是泛泛的 TypeScript 迁移建议加一个外链，**没有写它在 Nuxt 里具体打开哪些检查** |
| `tsConfig` | 无 | 无 | 扩展所有生成的 tsconfig；规则见下一节 |
| `typeCheck` | `boolean` | `false` | 开启构建期类型检查；设 `true` 时**在开发期也做类型检查**，要收窄到只在构建时检查就设成 `build`；前置条件是把 `typescript` 与 `vue-tsc` 装成 dev 依赖 |

三处不一致是这一节最该记住的：

- **`builder` 有默认值却没有类型行**，`null`、`false`、`'shared'` 三种取值只能从正文读（正文用的是 "'shared' option" 这种说法，不是字面量写法）。
- **`typeCheck` 类型行写 `boolean`，正文却给了一个字符串取值 `build`。** 文档没有把类型改成 `boolean | 'build'`，所以按类型行写的人会在 IDE 里看到 `build` 被标红。
- 四个 `*TsConfig` 里**只有两个**声明了自己优先于 `tsConfig`。剩下两个（`node`、`shared`）是漏写还是真的不优先，**文档未说明**。

## tsConfig 那段长说明，拆成六条

`### tsConfig` 的说明是整节最长的一段，规则密度很高。逐条拆开：

1. 用它可以给生成的 tsconfig 文件加**共享选项**。
2. `compilerOptions` 应用到**全部四个**生成文件：`.nuxt/tsconfig.app.json`、`.nuxt/tsconfig.server.json`、`.nuxt/tsconfig.node.json`、`.nuxt/tsconfig.shared.json`。
3. `include`、`exclude`、`vueCompilerOptions` **只应用到 `.nuxt/tsconfig.app.json`**（以及 legacy 的 `.nuxt/tsconfig.json`）。
4. 有两组 `compilerOptions` 是例外：DOM 与 Vue 相关的选项（官方举例 `lib`、`jsx`、`jsxImportSource`）**只应用到 app 那份**。
5. `types`、`paths`、`noEmit` 由 **Nuxt 按上下文管理**，因此**不能**为 `node`、`shared`、`server` 三份 tsconfig 全局设置它们。
6. 要按上下文覆盖，就用 `appTsConfig`、`serverTsConfig`、`nodeTsConfig`、`sharedTsConfig`，它们**优先于** `tsConfig`。

第 6 条与前面那个"只有两个子项声明了优先关系"是冲突的来源：**这段总说声明四个都优先，而 `nodeTsConfig`、`sharedTsConfig` 自己的条目里没写。** 按总说理解是四者都优先，但文档没有把这条冲突挑明。

## hoist 的默认表：17 个模块名

官方给的默认值是一串包名，逐个列出来（顺序照原文）：

```json
[
  "nitropack/types",
  "nitropack/runtime",
  "nitropack",
  "defu",
  "h3",
  "consola",
  "ofetch",
  "@unhead/vue",
  "@nuxt/devtools",
  "vue",
  "@vue/runtime-core",
  "@vue/compiler-sfc",
  "vue-router",
  "vue-router/auto-routes",
  "unplugin-vue-router/client",
  "@nuxt/schema",
  "nuxt"
]
```

**一共 17 项。** 这一项解决的是 pnpm 严格布局下的类型解析：给这些包在 `compilerOptions.paths` 里生成深度别名。那句限制要一起记住——"**This does not yet support subpaths.**"，也就是不能靠它去映射某个包的子路径；`nitropack/types`、`vue-router/auto-routes`、`unplugin-vue-router/client` 这几项本身就是子路径形式的**条目**，但那是官方列好的，不等于你可以给自己的子路径加进去。

## 生成的文件放在哪儿

这一节里四个 `*TsConfig` 的名字（app / server / node / shared）对应的就是生成的四份配置文件，另有 legacy `.nuxt/tsconfig.json` 的说法出现在 `appTsConfig` 那条里。生成的类型与 `tsconfig.json` 文件放到哪个目录，**由 `typesDir` 那一节管**，不在本节展开。

## 自查

按你自己项目核对：

- 想让 IDE 做类型检查：`typeCheck` 默认 `false`；设 `true` 是开发期也查，只想构建时查要设 `build`（**类型行没写这个字符串**）。
- `strict` 默认是 `true`，不是 `false`——觉得类型"太严"时先来看这一行，而不是去改 `tsConfig`。
- 想扩展生成的 tsconfig：先分清是"四个都要"（`tsConfig.compilerOptions`）还是"只动某一份"（对应的那个 `*TsConfig`）；`include`/`exclude`/`vueCompilerOptions` 写进 `tsConfig` **只影响 app 那一份**。
- 不要试图用 `tsConfig` 全局设 `types`、`paths`、`noEmit`——官方写明这三个由 Nuxt 按上下文管理。
- pnpm monorepo 且 `shamefully-hoist=false`：看 `hoist`，它只支持整包名，不支持子路径。
- `typescript.builder` 与 `server.builder` 是两回事：这一项只管**类型**要纳入哪个构建器的环境类型（`null` 推断、`false` 关闭、`'shared'` 给模块作者）。

## 下一篇

下一篇 577 看 `## vue`：**五个子项**（`compilerOptions`、`config`、`propsDestructure`、`runtimeCompiler`、`transformAssetUrls`），其中 `transformAssetUrls` **自己一句说明都没有**，却挂着五个带默认数组的子项；而 `runtimeCompiler` 那条带一句实打实的安全警告。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## typescript` 定性句；十一个 `###` 子项的原文、Type 与 Default 行；`tsConfig` 那段六条规则原文；`hoist` 默认 JSON 全 17 项与 "does not yet support subpaths" 句；`builder` 的推断句与 `false`／`'shared'`；`typeCheck` 的 `build` 与 `typescript`＋`vue-tsc` 前置；`shim` 的 ESLint 场景句；`strict` 说明句；生成类型落点归属另见同页 `## typesDir`）
