---
url: /posts/e067dbe87fc3466d776d8d5d75c714ad/
title: 类型是谁给的：自动生成的类型、默认不检查、以及四份 tsconfig
date: 2026-10-14T10:00:00+08:00
updated: 2026-10-14T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：官方在配置那页承诺"零配置 TypeScript 支持：靠自动生成的类型与 tsconfig.json，不用学习 TypeScript 就能写类型安全的代码"；而概念页给出的两个关键事实常常被读漏。第一，默认情况下 Nuxt 不会在 nuxt dev 或 nuxt build 时检查类型，原因是性能；要检查需要装 vue-tsc 与 typescript 并运行 nuxt typecheck，或者设置 typescript.typeCheck: true。第二，类型是自动生成的，存在 .nuxt 目录里，在你运行 dev 服务器或构建时生成，也可以手动用 nuxt prepare 生成；生成的 tsconfig 里包含推荐的基础配置、对自动导入的引用、API 路由类型与 #imports、~/file、#build/file 这类路径别名。Nuxt 还用 TypeScript 项目引用把类型拆成 app、node、server、shared 四个上下文，因此类型增强必须写在对应目录里。

categories:
  - 前端开发

tags:
  - nuxt4
  - typescript
  - 类型
  - tsconfig
  - 工具链

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


002 讲那条"零配置 TypeScript 支持"下面我留了一句警告：**类型是生成的，检查是另开的。** 这一讲就把这两件事彻底分清——它能解释两个很常见的困惑："为什么我项目里满屏红波浪线"和"为什么有类型错误却照样跑起来了"。

## 官方承诺的是什么

介绍页给的是这一句：

> **零配置 TypeScript 支持**：不用学习 TypeScript 就能写类型安全的代码，靠的是我们自动生成的类型与 `tsconfig.json`。

注意它的措辞——承诺的是**类型信息存在**，不是**类型会被检查**。这就是本节要拆的第一件事。

## 事实一：默认不检查类型

概念页开头写得很直接：

> 默认情况下，出于性能原因，Nuxt **不会**在运行 `nuxt dev` 或 `nuxt build` 时检查类型。

所以"跑起来了但类型是错的"不是 bug，而是默认行为。要打开检查，官方给了两条路。

**第一条：装依赖，跑命令。**

> 要在构建或开发时启用类型检查，把 `vue-tsc` 与 `typescript` 安装为开发依赖：

```bash
npm install --save-dev vue-tsc typescript
```

然后：

```bash
npx nuxt typecheck
```

**第二条：让构建过程自动做。**

> 要在构建或开发时启用类型检查，你也可以使用 `nuxt.config` 文件里的 `typescript.typeCheck` 选项：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  typescript: {
    typeCheck: true,
  },
})
```

两条路的差别值得想清楚：命令式适合放进 CI（"构建前先跑一次 typecheck"），配置式会让每次 dev/build 都变慢——官方之所以默认关掉它，理由就是性能。**我的建议是：开发时别开，CI 里必开。**（这是我的取舍，官方的写法是"你可以选择其一"。）

## 事实二：类型是生成的，而且有一个专门的目录

> Nuxt 项目依赖自动生成的类型才能正常工作。这些类型存储在 `.nuxt` 目录里，并在你**运行 dev 服务器或构建应用时生成**。你也可以通过运行 `nuxt prepare` 手动生成这些文件。

接着是这一页最有信息量的一段：

> 生成的 `.nuxt` 目录里的 `tsconfig.json` 文件为你的项目包含了**推荐的基础 TypeScript 配置**、对自动导入的引用、API 路由类型，以及像 `#imports`、`~/file` 或 `#build/file` 这样的路径别名，还有更多。

把这段和 006 讲、011 讲串起来看，你会发现它同时解释了三件事：

1. **自动导入为什么有类型**：因为生成的配置里有"对自动导入的引用"。
2. **`#imports` 为什么能被识别**：它是生成的 tsconfig 里的一个**路径别名**，不是一个真实存在的包。
3. **为什么刚 clone 的项目会满屏红**：`.nuxt` 还没生成，别名与自动导入的引用都不存在。跑一次 dev、build 或 `nuxt prepare` 就好了——这就是 035 讲"让类型先就绪"那句用词的确切含义。

还有一条必须记住的警告：

> Nuxt 依赖这份配置，而且 Nuxt 模块也可能扩展它。因此，**不推荐直接修改你的 `tsconfig.json` 文件**，因为这样做可能会覆盖掉重要的设置。请改为通过 `nuxt.config.ts` 扩展它。

也就是说：**根目录那份 `tsconfig.json` 是"引用者"，不是"配置源"**。要动 TypeScript 设置，走 `nuxt.config`；要加路径别名，也用官方给的方式而不是覆盖生成配置。

## 事实三：类型被拆成四个上下文

> 当你运行 `nuxt dev`、`nuxt build` 或 `nuxt prepare` 时，Nuxt 会为应用的不同部分生成**多个** `tsconfig.json` 文件。

- **`.nuxt/tsconfig.app.json`** —— `app/` 目录内应用代码的配置
- **`.nuxt/tsconfig.node.json`** —— 你的 `nuxt.config.ts` 以及不属于其它上下文的那些文件的配置
- **`.nuxt/tsconfig.server.json`** —— 服务端代码的配置（在适用时）
- **`.nuxt/tsconfig.shared.json`** —— 用于在应用与服务端上下文之间共享的代码（例如类型与非环境相关的工具函数）

官方给的动机是性能与 IDE 体验（"Nuxt 使用 TypeScript 项目引用来提升类型检查性能并提供更好的 IDE 支持。这个特性让 TypeScript 能把你的代码库拆成更小、更易管理的部分"）。

但这一节真正会咬人的是下面这条规则：

> 由于项目被划分成**多个类型上下文**，**在正确的上下文里增强类型**非常重要，这样才能确保它们被正确识别。除非被显式包含在合适上下文中，否则放在这些目录之外的增强文件将不被 TypeScript 识别。

官方直接给了三个落点：想增强 `app` 上下文的类型 → 增强文件放 `app/`；`server` 上下文 → 放 `server/`；应用与服务端之间共享的类型 → 放 `shared/`。

这条规则值得你现在就抄进笔记，因为后面会撞两次：一次是给 `definePageMeta` 的自定义字段加类型（那个 `declare module '#app'` 的 `.d.ts` 必须落在正确上下文里），一次是写测试时——只有 `test/nuxt`、`tests/nuxt` 默认进入应用类型上下文，别处的测试文件拿不到这些类型。

## 严格检查：一个开关决定另一件事是否默认发生

> 严格检查在启用 `typescript.typeCheck` 选项时，默认在 Nuxt 中开启，以给你更高的类型安全性。
>
> 如果你正在把代码库转换为 TypeScript，你可能想临时禁用严格检查，方法是在 `nuxt.config` 里把 `strict` 设为 `false`：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  typescript: {
    strict: false,
  },
})
```

把关系摆正：**`typeCheck` 决定"要不要检查"，`strict` 决定"检查时严不严"，而后者在前者开启时默认是开的。** 所以迁移旧项目时，合理组合是 `typeCheck: true` + `strict: false`：先把错误收集出来，再逐步收紧——而不是干脆关掉检查装作没事。

## 三个必须记住的结论

1. **默认不检查。** 有人检查才算有质量闸门：CI 里 `nuxt typecheck`，或者 `typescript.typeCheck: true`。
2. **类型来自 `.nuxt` 的生成物。** 编辑器一片红先想"生成过没有"，别急着改代码。
3. **增强类型要放在对应上下文目录里**（`app/`、`server/`、`shared/`），并且不要直接改 `tsconfig.json`。

## 自查

- [ ] 你的项目里现在有一条真在跑的类型检查吗（命令或配置）？
- [ ] 能区分"类型被生成"与"类型被检查"这两件事吗？
- [ ] 刚 clone 的项目满屏红时，你的第一反应是不是"跑一次 dev/build/nuxt prepare"？
- [ ] 你知道 `#imports` 是生成的 tsconfig 里的路径别名，而不是一个包吗？
- [ ] 你的类型增强文件放在正确的上下文目录里吗（`app/`、`server/`、`shared/`）？
- [ ] 有没有直接改根目录 `tsconfig.json`（官方不推荐）？需要严格度调整时用的是 `typescript.strict` 吗？

## 下一篇

008 - 服务端组件与岛屿：第一次见到这两个词该记住什么

## 本文依据

- `guide/concepts/typescript`（默认不在 dev/build 检查类型及其性能原因、装 vue-tsc 与 typescript 后跑 nuxt typecheck、typescript.typeCheck: true 的用法、自动生成的类型存于 .nuxt 且由 dev/build 生成、可用 nuxt prepare 手动生成、生成的 tsconfig 含推荐基础配置/自动导入引用/API 路由类型/`#imports` 与 `~/file`、`#build/file` 等路径别名、不推荐直接改 tsconfig.json 而应通过 nuxt.config.ts 扩展、项目引用与四份 tsconfig 及各自用途、增强类型必须在正确上下文（app/server/shared 三个落点）、严格检查在 typeCheck 启用时默认开启并可 typescript.strict: false 临时关闭）
- `getting-started/introduction`（"零配置 TypeScript 支持：自动生成的类型与 tsconfig.json"）
- `getting-started/installation`（不要求深入 Vue 但建议阅读 Vue 文档，用于对照"不用学 TS 也能写类型安全代码"这句承诺）
- `directory-structure/nuxt`（`.nuxt` 为生成物目录）
