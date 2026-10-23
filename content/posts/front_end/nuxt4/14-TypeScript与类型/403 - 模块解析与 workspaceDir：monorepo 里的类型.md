---
url: /posts/6c7a58c37cbb16e5f7cfed581a6529a7/
title: 403 - 模块解析与 workspaceDir：monorepo 里的类型
date: 2027-02-23T10:00:00+08:00
updated: 2027-02-23T10:00:00+08:00
author: cmdragon

summary:
  摘要：单包里"类型找不到"很少见，一进 monorepo 就多了：依赖被提到 workspace 根、pnpm 严格布局下 Nuxt 解析不到某个包、要跨包共享类型。Nuxt 给这三件事各配了一个开关。`workspaceDir` 官方说"Nuxt will attempt to detect your workspace directory automatically, but you can override it here"且"normally not needed"；`typescript.hoist` 给一批模块生成 `compilerOptions.paths` 深别名，官方点名的场景正是"pnpm monorepo with `shamefully-hoist=false`"，默认清单 17 项且**不支持子路径**；`typescript.includeWorkspace`（默认 `false`）把父 workspace 纳入项目，官方说主要用于主题与模块作者。加上 `typescript.builder` 与 `typesDir`/`buildDir` 的分工，就是 monorepo 里配类型的完整清单。

categories:
  - 前端开发

tags:
  - nuxt4
  - workspaceDir
  - monorepo
  - typescript.hoist
  - includeWorkspace
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

把 Nuxt 应用放进 pnpm workspace 之后，最先坏掉的往往不是运行，而是类型：编辑器开始报找不到 `h3`、找不到 `nitropack/types`，或者 `nuxt.config.ts` 里 import 同 workspace 的包飘红。这三个开关就是为这场景准备的。

## workspaceDir：多数时候不用管

nuxt-config 里 `## workspaceDir`：

> Define the workspace directory of your application. Often this is used when in a monorepo setup. Nuxt will attempt to detect your workspace directory automatically, but you can override it here. It is normally not needed to configure this option.

- 类型：`string`
- 默认：`"/<workspaceDir>"`

关键词是 **自动检测**＋**通常不需要配**。它存在的意义是自动检测猜错时给你一个覆盖口（080 讲过它和 `rootDir`、`typesDir`、`modulesDir` 怎么配合做多项目共置）。所以第一步不是配它，而是确认检测结果对不对。

## typescript.hoist：pnpm 严格布局的解药

真正处理"类型找不到某个包"的是 `typescript.hoist`。官方说明：

> Modules to generate deep aliases for within `compilerOptions.paths`. This does not yet support subpaths. It may be necessary when using Nuxt within a pnpm monorepo with `shamefully-hoist=false`.

三点：

1. 机制是给这些模块在 `compilerOptions.paths` 里生成**深别名**。
2. **尚不支持子路径**（`hoist` 里写 `some-pkg/sub` 这种不起作用）。
3. 点名的场景是 pnpm monorepo 且 `shamefully-hoist=false`——也就是依赖没被提升到根、解析不到那种。

默认清单（17 项）：

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

所以默认情况下这 17 个已经帮你别名好了。**你要动它，通常是发现报错的是这份清单之外的包**——把那个包名加进去，而不是重写整张表。

## typescript.includeWorkspace：把父 workspace 纳进来

> Include parent workspace in the Nuxt project. Mostly useful for themes and module authors.

- 类型：`boolean`
- 默认：`false`

用途官方限定得很窄——**主题作者与模块作者**。典型是你在 workspace 里做一个 layer/模块包，希望它的类型能看见父 workspace 的内容，就开这个。普通应用项目一般用不上。

## typescript.builder：多构建器的类型取舍

monorepo 里模块作者还会碰到第四个：

> Which builder types to include for your project. By default Nuxt infers this based on your `builder` option (defaulting to 'vite') but you can either turn off builder environment types (with `false`) to handle this fully yourself, or opt for a 'shared' option. The 'shared' option is advised for module authors, who will want to support multiple possible builders.

- 默认 `null`（按 `builder` 推断，缺省 vite）
- `false`＝不要构建器环境类型，自己全管
- `'shared'`＝要同时支持多种构建器的模块作者用

## 类型目录别跟着 buildDir 一起搬

workspace 里还容易踩一个坑：把 `buildDir` 挪走，结果类型位置也变了。357 那条要记住：`typesDir` 默认取 `buildDir`，但官方刻意把两者**分开**——"is separate from it so that the configurations referenced by your project `tsconfig.json` stay in place even when Nuxt builds elsewhere"。根 tsconfig 那四条 `references` 指向 `.nuxt` 下的四份文件（395），所以搬 `buildDir` 时如果没显式设 `typesDir`，生成的类型仍留在原位，这是设计。

## 排查顺序（monorepo 类型报错）

1. 确认 `workspaceDir` 检测结果对不对（一般不用配）。
2. 看报错的包在不在 `hoist` 默认 17 项里；不在就加进去，注意不支持子路径。
3. 是主题/模块包要跨 workspace 看类型 → `includeWorkspace: true`。
4. 模块要支持多种构建器 → `typescript.builder: 'shared'`。
5. 生成物本身可疑 → `nuxt cleanup` + `nuxt prepare` 重来一遍（408）。

## 自查

- `workspaceDir` 自动检测、官方说"normally not needed"，只在猜错时覆盖。
- `typescript.hoist` 是给指定模块生成 `compilerOptions.paths` 深别名，场景是 pnpm `shamefully-hoist=false`；默认 17 项、**不支持子路径**；要加的是清单外的包名。
- `typescript.includeWorkspace` 默认 `false`，官方限定用途为主题/模块作者。
- `typescript.builder` 默认 `null`（推断，缺省 vite），`false` 自己管，`'shared'` 给多构建器模块作者。
- `typesDir` 与 `buildDir` 被刻意分开，只搬 `buildDir` 时类型仍留原位是设计。

## 下一篇

下一篇 404 讲 `#imports` 这条显式通道：官方"Nuxt exposes every auto-import with the `#imports` alias"，以及关掉 `imports.autoImport` 之后为什么它仍然可用。
