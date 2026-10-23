---
url: /posts/5c33f8224970e04f45a7f9b5d2b8795d/
title: 397 - typescript 配置节：typeCheck、shim、strict 与四份 tsconfig
date: 2027-02-21T10:00:00+08:00
updated: 2027-02-21T10:00:00+08:00
author: cmdragon

summary:
  摘要：`nuxt.config` 的 `typescript` 节是"别手改生成的 tsconfig"那条禁令的正解——官方定性它是 "Configuration for Nuxt's TypeScript integration."。这一节共十一个键，逐个讲：三个开关（`typeCheck` 可 `true`/`'build'`/`false` 且要装 `typescript`+`vue-tsc`、`strict` 默认 `true`、`shim` 默认 `false` 且官方更推荐交给 Vue 官方扩展）、一份共享扩展（`tsConfig`）、四份按上下文的扩展（`appTsConfig`/`serverTsConfig`/`nodeTsConfig`/`sharedTsConfig`，且前两者优先于 `tsConfig`）、两个偏模块作者的（`builder` 默认 `null`、`includeWorkspace` 默认 `false`）、`hoist`（pnpm monorepo 的深别名清单）。还有一条必须知道：`tsConfig` 里 `types`/`paths`/`noEmit` **设了也没用**。

categories:
  - 前端开发

tags:
  - nuxt4
  - typescript节
  - typeCheck
  - tsConfig
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

395 说不许直接改生成的 `tsconfig.json`，要扩展就走 `nuxt.config`。走的就是 `typescript` 这一节。官方给它的定位一句话："Configuration for Nuxt's TypeScript integration."。这一节底下十一个键，逐条落地。

## 三个开关

### `typeCheck`

- 类型：`boolean`（官方文档行写 `boolean`，但正文另给 `'build'` 一档）
- 默认：`false`

官方说明：

> Enable build-time type checking. If set to true, this will type check in development. You can restrict this to build-time type checking by setting it to `build`. Requires to install `typescript` and `vue-tsc` as dev dependencies.

三档含义：`false`（默认，不检查）、`true`（开发期也检查）、`'build'`（只在构建期检查）。**前提是装了 `typescript` 和 `vue-tsc`**——没装就无效，这也是 395 那条"默认不检查类型"的配置侧。

### `strict`

- 类型：`boolean`
- 默认：`true`

官方说明："TypeScript comes with certain checks to give you more safety and analysis of your program. Once you've converted your codebase to TypeScript, you can start enabling these checks for greater safety."

从 JS 往 TS 迁、暂时吃不消一堆严格报错时，`strict: false` 是官方给的临时出口（typescript 那页同句）。

### `shim`

- 类型：`boolean`
- 默认：`false`

官方说明：

> Generate a `*.vue` shim. We recommend instead letting the official Vue extension generate accurate types for your components. Note that you may wish to set this to `true` if you are using other libraries, such as ESLint, that are unable to understand the type of `.vue` files.

也就是说官方**不推荐**开它——更建议让 Vue 官方扩展（Volar）去生成准确的组件类型；只有在用了看不懂 `.vue` 的其它工具（比如 ESLint）时才设 `true`。

## 一份共享扩展 + 四份按上下文扩展

### `tsConfig`

官方说明把边界写得很细，这段是本篇最关键的一条：

> You can extend the generated tsconfig files with shared options using this option. `compilerOptions` apply to all generated tsconfigs ... while `include`, `exclude` and `vueCompilerOptions` apply only to `.nuxt/tsconfig.app.json` (and the legacy `.nuxt/tsconfig.json`).

然后是两个例外：

> Two groups of `compilerOptions` are exceptions: DOM- and Vue-specific options (such as `lib`, `jsx` and `jsxImportSource`) apply only to `.nuxt/tsconfig.app.json`, and `types`, `paths` and `noEmit` are managed by Nuxt per context, so they cannot be set globally for the `node`, `shared` and `server` tsconfigs.

归成三条规则：

1. `compilerOptions` 进所有生成的 tsconfig；`include`/`exclude`/`vueCompilerOptions` **只作用 app**（和遗留那份）。
2. `lib`/`jsx`/`jsxImportSource` 这类 DOM/Vue 相关项**也只作用 app**。
3. `types`、`paths`、`noEmit` 由 Nuxt 按上下文管理，**在 `tsConfig` 里设了不生效**。

第 3 条就是"配了没反应"最常见的成因。要按上下文覆盖，改用下面四个键——官方明说它们优先于 `tsConfig`（"Use `appTsConfig`, `serverTsConfig`, `nodeTsConfig` or `sharedTsConfig` for context-specific overrides; they take precedence over this option."）。

### `appTsConfig` / `serverTsConfig` / `nodeTsConfig` / `sharedTsConfig`

分别扩展生成的 `.nuxt/tsconfig.app.json`、`tsconfig.server.json`、`tsconfig.node.json`、`tsconfig.shared.json`。其中 `appTsConfig` 与 `serverTsConfig` 的描述都额外带一句 "Options set here take precedence over `tsConfig`."。

`directory-structure/tsconfig` 那页补了一条联动关系：

> `typescript.serverTsConfig` extends `tsconfig.server.json`. `nitro.typescript.tsConfig` is kept in sync with it but is deprecated, so set `typescript.serverTsConfig` instead.

也就是说旧的 `nitro.typescript.tsConfig` 已经废弃，服务端那份要改就写 `typescript.serverTsConfig`。

## 两个偏模块作者的键

### `builder`

官方说明：

> Which builder types to include for your project. By default Nuxt infers this based on your `builder` option (defaulting to 'vite') but you can either turn off builder environment types (with `false`) to handle this fully yourself, or opt for a 'shared' option. The 'shared' option is advised for module authors, who will want to support multiple possible builders.

- 默认：`null`（即按 `builder` 选项推断，缺省 vite）
- `false`＝不要构建器环境类型、自己全管；`'shared'`＝模块作者要同时支持多种构建器时用这个

### `includeWorkspace`

- 类型：`boolean`，默认 `false`
- 官方说明："Include parent workspace in the Nuxt project. Mostly useful for themes and module authors."

### `hoist`

- 类型：`array`
- 官方说明："Modules to generate deep aliases for within `compilerOptions.paths`. This does not yet support subpaths. It may be necessary when using Nuxt within a pnpm monorepo with `shamefully-hoist=false`."

默认清单（逐条，共 17 项）：`nitropack/types`、`nitropack/runtime`、`nitropack`、`defu`、`h3`、`consola`、`ofetch`、`@unhead/vue`、`@nuxt/devtools`、`vue`、`@vue/runtime-core`、`@vue/compiler-sfc`、`vue-router`、`vue-router/auto-routes`、`unplugin-vue-router/client`、`@nuxt/schema`、`nuxt`。

这条是 monorepo 专属：`shamefully-hoist=false` 的 pnpm 工作区里类型找不到包，就动 `hoist`。注意官方写了"**尚不支持子路径**"。

## 全节一览

| 键 | 类型/默认 | 管什么 |
| --- | --- | --- |
| `typeCheck` | `false`（可 `true`/`'build'`） | 开类型检查，需装 typescript+vue-tsc |
| `strict` | `true` | 严格检查，迁移期可关 |
| `shim` | `false` | 生成 `*.vue` shim；官方推荐交给 Volar |
| `tsConfig` | — | 共享扩展；`types`/`paths`/`noEmit` 在此无效 |
| `appTsConfig` | — | 扩展 app 那份，优先于 `tsConfig` |
| `serverTsConfig` | — | 扩展 server 那份，优先于 `tsConfig`；替代废弃的 `nitro.typescript.tsConfig` |
| `nodeTsConfig` | — | 扩展 node 那份 |
| `sharedTsConfig` | — | 扩展 shared 那份 |
| `builder` | `null`（可 `false`/`'shared'`） | 构建器环境类型；模块作者用 `'shared'` |
| `includeWorkspace` | `false` | 把父 workspace 纳进来；主题/模块作者用 |
| `hoist` | 17 项默认清单 | 深别名，pnpm monorepo `shamefully-hoist=false` 时用 |

## 自查

- 三个开关：`typeCheck`（false/true/build，要装 typescript+vue-tsc）、`strict`（默认 true）、`shim`（默认 false，官方更推荐 Volar）。
- `tsConfig` 是共享扩展，但 `types`/`paths`/`noEmit` 在此**无效**，`include`/`exclude`/`vueCompilerOptions` 与 `lib`/`jsx`/`jsxImportSource` **只作用 app**。
- 按上下文改就用四个 `*TsConfig`，其中 app/server 两个明确优先于 `tsConfig`。
- `nitro.typescript.tsConfig` 已废弃，改写 `typescript.serverTsConfig`。
- `builder`/`includeWorkspace`/`hoist` 主要服务模块作者与 monorepo；`hoist` 不支持子路径。

## 下一篇

下一篇 398 处理最恼人的一类报错：明明在用的自动导入，IDE 却提示找不到——根子在生成类型没就绪与上下文目录，讲清排查顺序。
