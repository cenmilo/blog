---
url: /posts/00dcf4c1c49cb237176334202de17da8/
title: 414 - typescript 节逐条：typeCheck 与 shim 的取舍
date: 2027-02-27T09:00:00+08:00
updated: 2027-02-27T09:00:00+08:00
author: cmdragon

summary:
  摘要：397 把 `typescript` 那十一个键的默认值逐条列全了，这一篇只回答另一件事：每一个开关该不该动、动了换到什么。四个真决策：`typeCheck` 三档（`false` 默认/`'build'` 只在构建拦/`true` 开发期也跑，代价是 dev 启动与热更变慢，且都要先装 `typescript`+`vue-tsc`）；`strict` 从 `true` 关掉只该出现在 JS→TS 迁移途中；`shim` 官方**不推荐**开（更建议交给 Vue 官方扩展生成组件类型），唯一被点名的理由是 ESLint 这类看不懂 `.vue` 的工具；`tsConfig` 与四个 `*TsConfig` 的选择——先记 `types`/`paths`/`noEmit` 在 `tsConfig` 里根本无效，要按上下文改就必须用对应那一份。

categories:
  - 前端开发

tags:
  - nuxt4
  - typeCheck
  - shim
  - strict
  - 配置取舍
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`typescript` 这一节有十一个键，但九成项目一辈子只碰其中两三个。这一篇不讲清单（397 已经逐条列全），只讲每一处**该不该动、动了换来什么**。

## 决策一：`typeCheck` 的三档

官方给的三档语义：默认 `false`；`true` 会**在开发期也做类型检查**；`'build'` 把它限制到构建期；且无论哪一档都**要求装 `typescript` 与 `vue-tsc`** 作 devDependency。

三档各自的代价与收益：

| 取值 | 什么时候类型错会拦住你 | 代价 |
| --- | --- | --- |
| `false`（默认） | 只有手动跑 `nuxt typecheck` 时 | 类型错能一路带到合并 |
| `'build'` | 构建时失败 | dev 保持快 |
| `true` | 开发期就报 | dev 启动与热更变慢 |

官方默认 `false` 的理由写在 typescript 那页："By default, Nuxt doesn't check types when you run `nuxt dev` or `nuxt build`, for performance reasons."——这是性能换安全的选择，不是"类型不重要"。

CI 里常见的组合是：不占 dev 的时间，把检查放在流水线里显式跑 `nuxt typecheck`（399），或者构建时拦住用 `'build'`。

## 决策二：`strict` 什么时候该是 `false`

`strict` 默认 `true`。官方的表述是：strict checks 在启用了 `typescript.typeCheck` 时默认开启，给你更强的类型安全；正在把代码库转成 TypeScript 时可以**临时**关掉：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  typescript: {
    strict: false,
  },
})
```

用这一档的判断标准很清楚：它是给**迁移途中的逃生门**，不是长期状态。typescript 那页给的是"如果你正在把代码库转成 TypeScript，你可能想临时关掉严格检查"——临时两字是官方自己的措辞。新写的直接迁移项目把 `strict` 关着，等于自愿长期承担 373 那一类问题的风险。

## 决策三：`shim` 默认不开，而且官方不建议开

`shim` 生成 `*.vue` 的 shim。官方对这个键的态度在文档里少见地直白：

> Generate a `*.vue` shim. We recommend instead letting the official Vue extension generate accurate types for your components. Note that you may wish to set this to `true` if you are using other libraries, such as ESLint, that are unable to understand the type of `.vue` files.

- 默认 `false`。
- **首选方案不是它**：让 Vue 官方扩展（Volar）去生成准确的组件类型。
- 官方点名的唯一理由是"用了看不懂 `.vue` 文件的其它库，例如 ESLint"。

所以顺序应该是：先确认编辑器装了对的扩展；只有当某个工具链（典型就是 ESLint）确实读不懂 `.vue` 类型时，才用 `shim: true` 补一个 shim。把它当"类型报错的万能药"开上，等于用一份粗糙的兜底声明去覆盖本来准确的类型。

## 决策四：`tsConfig` 还是某个 `*TsConfig`

这一处最容易白改。规则（397 逐条列过）：

- `compilerOptions` 进所有生成的 tsconfig；但 `include`/`exclude`/`vueCompilerOptions` **只作用 app**（含遗留那份）。
- `lib`/`jsx`/`jsxImportSource` 这类 DOM/Vue 相关项**也只作用 app**。
- **`types`、`paths`、`noEmit` 由 Nuxt 按上下文管理，在 `tsConfig` 里设了无效。**

所以选择逻辑就两条：

1. 要改的是 `types`/`paths`/`noEmit`，或者只想影响某一个上下文 → 直接用对应的 `appTsConfig` / `serverTsConfig` / `nodeTsConfig` / `sharedTsConfig`。官方还写明 app 与 server 这两份"Options set here take precedence over `tsConfig`."。
2. 要改的是真·全局、且不在上面那三个例外里的 `compilerOptions` → 才用 `tsConfig`。

另一个常见误用：409 的 395 禁令同样适用——想调类型行为就动 `nuxt.config`，别去改 `.nuxt` 里生成的 tsconfig（`::warning`："the whole directory will be re-created when running `nuxt dev`"）。

服务端那份还有一条替代关系要记：`nitro.typescript.tsConfig` **已废弃**，它跟 `typescript.serverTsConfig` 保持同步，所以要改服务端的就写 `typescript.serverTsConfig`。

## 三个键通常不用碰

`builder`、`includeWorkspace`、`hoist` 这三个是给特定处境的（397/403）：

- `builder`：只有你是不想被构建器环境类型束缚、或要同时支持多构建器的模块作者才动（`false` / `'shared'`）。
- `includeWorkspace`：默认 `false`，官方限定主题与模块作者。
- `hoist`：只有 pnpm monorepo 且 `shamefully-hoist=false` 时，为**默认 17 项清单之外**的包补深别名（且不支持子路径）。

普通应用项目这三条一律保持默认即可。

## 自查

- `typeCheck` 三档是"什么时候拦"的取舍：`false` 默认（dev 快）、`'build'` 构建拦、`true` 开发期也跑但变慢；三档都要装 `typescript`+`vue-tsc`。
- `strict: false` 是 JS→TS 迁移途中的**临时**逃生门，不是长期状态。
- `shim` 官方不推荐开——首选让 Vue 官方扩展生成组件类型；只在 ESLint 这类读不懂 `.vue` 的工具面前设 `true`。
- `types`/`paths`/`noEmit` 在 `tsConfig` 里无效；按上下文改用四个 `*TsConfig`（app/server 两份优先于 `tsConfig`）；服务端别再写已废弃的 `nitro.typescript.tsConfig`。
- `builder`/`includeWorkspace`/`hoist` 通常保持默认。

## 下一篇

下一篇 415 起进入 415-434 样式与资源：样式接入的三条路——`css` 数组全局注入、组件内 import 或 `@import`、以及外部样式表通过 head 挂进来。
