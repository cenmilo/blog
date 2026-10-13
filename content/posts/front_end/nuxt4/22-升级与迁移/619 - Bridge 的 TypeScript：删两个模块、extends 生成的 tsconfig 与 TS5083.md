---
url: /posts/e20324ece94e195d028f02fa7cc5fa84/
title: 619 - Bridge 的 TypeScript：删两个模块、extends 生成的 tsconfig 与 TS5083
date: 2027-05-06T10:00:00+08:00
updated: 2027-05-06T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：`bridge/typescript` 只有两个 `##` 章节。第一章先删东西：**删 `@nuxt/typescript-build`（因为 Bridge 已经提供同样能力）**、**删 `@nuxt/typescript-runtime` 与 `nuxt-ts`（因为 Nuxt 2 自带运行时支持）**；紧接着在同一章下设 `bridge.typescript: true`，并在同一个代码块里带一句 `nitro: false, // If migration to Nitro is complete, set to true`。第二章改 `tsconfig.json`：**加一行 `"extends": "./.nuxt/tsconfig.json"`**，然后两条 `::note` 各管一个坑——**这个文件是生成的、不入库，所以测试前必须跑 `nuxi prepare`，否则报 `TS5083: Cannot read file '~/.nuxt/tsconfig.json'`**；**你自己 `tsconfig.json` 里的同名选项会覆盖继承来的，尤其覆盖 `compilerOptions.paths` 会让 TypeScript 不再采纳 `./.nuxt/tsconfig.json` 里的模块解析，结果 `#imports` 认不出来**，要追加别名请用 `nuxt.config` 里的 `alias`。

categories:
  - 前端开发

tags:
  - nuxt4
  - bridge
  - TypeScript
  - tsconfig
  - 升级迁移
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

Bridge 项目里改类型配置，最常出现的症状是：编辑器忽然认不出 `#imports`，或者 CI 里报一句 `TS5083`。这一节两处提示正好对应这两个症状。

## 第一步是删，不是加

`## Remove Modules` 下面是两条，各自带理由：

- **Remove `@nuxt/typescript-build`**：Bridge enables same functionality.
- **Remove `@nuxt/typescript-runtime` and `nuxt-ts`**：Nuxt 2 has built-in runtime support.

也就是说：**Bridge 会接管类型构建，Nuxt 2 自己也已经带运行时支持**，旧的那三个包都成了多余的。

紧接着是同一章里的小节（`### Set bridge.typescript`，层级上挂在 Remove Modules 下面）：

```ts [nuxt.config.ts]
import { defineNuxtConfig } from '@nuxt/bridge'

export default defineNuxtConfig({
  bridge: {
    typescript: true,
    nitro: false, // If migration to Nitro is complete, set to true
  },
})
```

注释那句 **"If migration to Nitro is complete, set to true"** 是这一节与 Bridge 其它小节共用的写法：**`nitro` 的状态是"迁移进度指示器"**，Bridge 的 TypeScript、Composition API、meta 各节示例都带着这一行。

## 第二步：让 tsconfig 去 extends 生成的那份

> If you are using TypeScript, you can edit your `tsconfig.json` to benefit from auto-generated Nuxt types:

```diff [tsconfig.json]
{
+ "extends": "./.nuxt/tsconfig.json",
  "compilerOptions": {
    ...
  }
}
```

改法只有一行。两条 `::note` 才是这一节真正要读的部分。

## 第一条 note：不入库的文件会让测试挂掉

> As `.nuxt/tsconfig.json` is generated and not checked into version control, you'll need to generate that file before running your tests. Add `nuxi prepare` as a step before your tests, otherwise you'll see `TS5083: Cannot read file '~/.nuxt/tsconfig.json'`

三条信息：**该文件生成且不进版本库**、**测试前要先生成它**、**不做的具体症状是 `TS5083`，报错里带的是 `~/.nuxt/tsconfig.json` 这个路径**。

官方给的修法是把 **`nuxi prepare`** 作为测试前的一个步骤——**不是让你把 `tsconfig.json` 提交进仓库**。

这条 note 还有后半句，指向 4.x 的做法：**"For modern Nuxt projects, we recommend using TypeScript project references instead of directly extending `.nuxt/tsconfig.json`."** ——**Bridge 这一节教的 `extends` 是旧写法**，新做法（也是迁移文档那三步用的）是指向四份生成的配置：

```json
{
  "files": [],
  "references": [
    { "path": "./.nuxt/tsconfig.app.json" },
    { "path": "./.nuxt/tsconfig.server.json" },
    { "path": "./.nuxt/tsconfig.shared.json" },
    { "path": "./.nuxt/tsconfig.node.json" }
  ]
}
```

两种写法不要混用：要么 `extends` 那一份、要么 references 那四份。

## 第二条 note：覆盖 paths 会让 #imports 消失

> Keep in mind that all options extended from `./.nuxt/tsconfig.json` will be overwritten by the options defined in your `tsconfig.json`. Overwriting options such as `"compilerOptions.paths"` with your own configuration will lead TypeScript to not factor in the module resolutions from `./.nuxt/tsconfig.json`. This can lead to module resolutions such as `#imports` not being recognized.
>
> In case you need to extend options provided by `./.nuxt/tsconfig.json` further, you can use the `alias` property within your `nuxt.config`. `nuxi` will pick them up and extend `./.nuxt/tsconfig.json` accordingly.

这段把"编辑器忽然不认 `#imports`"的成因说透了：**`extends` 是覆盖而不是合并**——你在自己 `tsconfig.json` 里写 `compilerOptions.paths`，生成的那份里的模块解析就整个不参与了。

正确姿势是**别在 tsconfig 里补别名，改到 `nuxt.config` 的 `alias` 里去加**：`nuxi` 会读取它并把结果写进 `./.nuxt/tsconfig.json`。**这条链是"配置 → 生成的 tsconfig"的单向通道**，反方向（手改生成文件）不会保留。

## 自查

- 删除项两条：`@nuxt/typescript-build`（Bridge 已提供）、`@nuxt/typescript-runtime` 与 `nuxt-ts`（Nuxt 2 自带运行时）。
- 开关是 `bridge.typescript: true`，示例里同时带 `nitro: false` 及其注释——**Nitro 迁移完成后改 true**。
- `tsconfig.json` 只加一行 `"extends": "./.nuxt/tsconfig.json"`。
- **测试前先 `nuxi prepare`**，否则 `TS5083`；不要把生成文件入库。
- **`compilerOptions.paths` 一覆盖就丢模块解析，`#imports` 会认不出来**；要加别名请写 `nuxt.config` 的 `alias`。
- 现代项目官方建议改用 project references（四份 tsconfig），与 `extends` 二选一。

## 下一篇

下一篇 620 读 Bridge 的 Composition API 那一节：它先要求**删掉 `@vue/composition-api` 与 `@nuxtjs/composition-api`**，再分两条路走——只用过 `@vue/composition-api` 的只要**删掉手动注册那一步**；从 `@nuxtjs/composition-api` 迁的要注意 **Bridge 会提供一个 shim 覆盖"大多数"导入，但**三个例外**没有替代：`withContext`、`useStatic`（无替代）、以及早已废弃的 `reqRef`/`reqSsrRef`。

## 本文依据

- https://nuxt.com/docs/4.x/bridge/typescript（`## Remove Modules` 两条及其括号理由；`### Set bridge.typescript` 代码块与 `nitro: false` 注释；`## Update tsconfig.json` 的引导句与 diff；第一条 `::note`（生成不入库、测试前 `nuxi prepare`、`TS5083: Cannot read file '~/.nuxt/tsconfig.json'`、现代项目建议 project references 并链接到 tsconfig 那一节）；第二条 `::note`（extends 的选项被本地覆盖、覆盖 `compilerOptions.paths` 导致 `#imports` 认不出、改用 `nuxt.config` 的 `alias` 由 nuxi 合并进生成文件））
- https://nuxt.com/docs/4.x/migration/configuration（四份 project references 的 `tsconfig.json` 完整示例与"改 `tsconfig.json` 不等于要把应用写成 TS"那句，用于对照两种写法）
